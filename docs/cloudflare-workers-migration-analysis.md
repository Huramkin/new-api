# Cloudflare Workers 迁移分析文档

> 本文档是对 new-api Go 程序的 HTTP 接口和程序结构的全面分析，旨在为未来重构为 Cloudflare Workers 程序提供参考。

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 架构概览](#2-架构概览)
- [3. HTTP 接口完整清单](#3-http-接口完整清单)
  - [3.1 API 路由 `/api/*`](#31-api-路由-api)
  - [3.2 Dashboard 路由](#32-dashboard-路由)
  - [3.3 Relay 路由（AI API 代理）](#33-relay-路由ai-api-代理)
  - [3.4 Video 路由](#34-video-路由)
  - [3.5 Web 路由（静态资源）](#35-web-路由静态资源)
- [4. 中间件分析](#4-中间件分析)
  - [4.1 中间件清单与 Workers 兼容性](#41-中间件清单与-workers-兼容性)
  - [4.2 全局中间件链](#42-全局中间件链)
- [5. 控制器层分析](#5-控制器层分析)
- [6. Relay（AI API 代理）层分析](#6-relayai-api-代理层分析)
  - [6.1 Relay 架构](#61-relay-架构)
  - [6.2 Provider 适配器列表](#62-provider-适配器列表)
  - [6.3 流式传输实现](#63-流式传输实现)
- [7. 数据模型与存储依赖](#7-数据模型与存储依赖)
  - [7.1 数据库模型](#71-数据库模型)
  - [7.2 缓存架构](#72-缓存架构)
- [8. 后台任务与定时作业](#8-后台任务与定时作业)
- [9. Cloudflare Workers 迁移可行性评估](#9-cloudflare-workers-迁移可行性评估)
  - [9.1 可直接迁移的功能](#91-可直接迁移的功能)
  - [9.2 需要适配的功能](#92-需要适配的功能)
  - [9.3 无法实现的功能及原因](#93-无法实现的功能及原因)
  - [9.4 Cloudflare 替代方案对照表](#94-cloudflare-替代方案对照表)
- [10. 推荐的迁移策略](#10-推荐的迁移策略)

---

## 1. 项目概述

new-api 是一个 AI API 网关/代理系统，用 Go 构建，聚合了 40+ 上游 AI 提供商（OpenAI、Claude、Gemini、Azure、AWS Bedrock 等）于一个统一的 API 之后。包含用户管理、计费、限流和管理员仪表板。

**技术栈：**
- **后端**: Go 1.22+, Gin Web 框架, GORM v2 ORM
- **前端**: React 18, Vite, Semi Design UI
- **数据库**: 同时支持 SQLite, MySQL, PostgreSQL
- **缓存**: Redis (go-redis) + 内存缓存
- **认证**: JWT, WebAuthn/Passkeys, OAuth (GitHub, Discord, OIDC 等)

---

## 2. 架构概览

项目采用分层架构：`Router → Middleware → Controller → Service → Model`

```
请求入口
  │
  ▼
router/          ── HTTP 路由定义（5 个路由文件）
  │
  ▼
middleware/      ── 认证、限流、CORS、日志、分发等（21 个中间件文件）
  │
  ▼
controller/      ── 请求处理器（58 个文件）
  │
  ▼
service/         ── 业务逻辑（53 个文件）
  │
  ▼
model/           ── GORM 数据模型 + DB 访问（34 个文件）
  │
  ▼
relay/           ── AI API 代理引擎 + Provider 适配器（187 个文件）
  └── channel/   ── 37 个上游提供商适配器
  └── common/    ── 共享 relay 工具
  └── helper/    ── 流处理、定价、验证
```

**其他关键目录：**

| 目录 | 说明 |
|------|------|
| `common/` | 共享工具（JSON、加密、Redis、环境变量、限流等）—— 45 个文件 |
| `dto/` | 数据传输对象（请求/响应结构体）—— 28 个文件 |
| `constant/` | 常量定义（API 类型、渠道类型、上下文 key）—— 13 个文件 |
| `types/` | 类型定义（Relay 格式、文件源、错误）—— 9 个文件 |
| `setting/` | 配置管理（模型比例、运营、系统、性能）—— 43 个文件 |
| `i18n/` | 后端国际化 |
| `oauth/` | OAuth 提供商实现 |
| `pkg/` | 内部包（cachex, ionet） |
| `web/` | React 前端 |

**服务器启动流程 (`main.go`)：**
1. `InitResources()` — 加载环境变量、初始化数据库、Redis、i18n、OAuth 提供商
2. 启动后台任务（渠道缓存同步、配额数据导出、渠道测试、Codex 刷新、订阅重置等）
3. 创建 Gin 引擎，挂载全局中间件
4. `router.SetRouter()` — 注册所有路由
5. 监听端口

---

## 3. HTTP 接口完整清单

### 3.1 API 路由 `/api/*`

**路由组中间件：** `RouteTag("api")`, `Gzip`, `BodyStorageCleanup`, `GlobalAPIRateLimit`

#### 3.1.1 公开接口（无认证）

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/setup` | `GetSetup` | — | 获取系统初始化状态 |
| POST | `/api/setup` | `PostSetup` | — | 执行系统初始化 |
| GET | `/api/status` | `GetStatus` | — | 获取系统状态 |
| GET | `/api/uptime/status` | `GetUptimeKumaStatus` | — | Uptime Kuma 状态 |
| GET | `/api/notice` | `GetNotice` | — | 获取公告 |
| GET | `/api/user-agreement` | `GetUserAgreement` | — | 获取用户协议 |
| GET | `/api/privacy-policy` | `GetPrivacyPolicy` | — | 获取隐私政策 |
| GET | `/api/about` | `GetAbout` | — | 关于页面 |
| GET | `/api/home_page_content` | `GetHomePageContent` | — | 首页内容 |
| GET | `/api/pricing` | `GetPricing` | `TryUserAuth` | 获取定价信息 |
| GET | `/api/ratio_config` | `GetRatioConfig` | `CriticalRateLimit` | 获取模型比例配置 |

#### 3.1.2 认证相关接口

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/verification` | `SendEmailVerification` | `EmailVerificationRateLimit`, `TurnstileCheck` | 发送邮箱验证 |
| GET | `/api/reset_password` | `SendPasswordResetEmail` | `CriticalRateLimit`, `TurnstileCheck` | 发送密码重置邮件 |
| POST | `/api/user/reset` | `ResetPassword` | `CriticalRateLimit` | 重置密码 |
| POST | `/api/verify` | `UniversalVerify` | `UserAuth`, `CriticalRateLimit` | 统一验证（2FA+Passkey） |

#### 3.1.3 OAuth 接口

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/oauth/state` | `GenerateOAuthCode` | `CriticalRateLimit` | 生成 OAuth state |
| GET | `/api/oauth/email/bind` | `EmailBind` | `CriticalRateLimit` | 邮箱绑定 |
| GET | `/api/oauth/wechat` | `WeChatAuth` | `CriticalRateLimit` | 微信登录 |
| GET | `/api/oauth/wechat/bind` | `WeChatBind` | `CriticalRateLimit` | 微信绑定 |
| GET | `/api/oauth/telegram/login` | `TelegramLogin` | `CriticalRateLimit` | Telegram 登录 |
| GET | `/api/oauth/telegram/bind` | `TelegramBind` | `CriticalRateLimit` | Telegram 绑定 |
| GET | `/api/oauth/:provider` | `HandleOAuth` | `CriticalRateLimit` | 通用 OAuth 处理 |

#### 3.1.4 支付 Webhook

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| POST | `/api/stripe/webhook` | `StripeWebhook` | Stripe 支付回调 |
| POST | `/api/creem/webhook` | `CreemWebhook` | Creem 支付回调 |
| POST | `/api/waffo/webhook` | `WaffoWebhook` | Waffo 支付回调 |

#### 3.1.5 用户接口 `/api/user`

**公开用户接口：**

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| POST | `/api/user/register` | `Register` | `CriticalRateLimit`, `TurnstileCheck` | 用户注册 |
| POST | `/api/user/login` | `Login` | `CriticalRateLimit`, `TurnstileCheck` | 用户登录 |
| POST | `/api/user/login/2fa` | `Verify2FALogin` | `CriticalRateLimit` | 2FA 登录验证 |
| POST | `/api/user/passkey/login/begin` | `PasskeyLoginBegin` | `CriticalRateLimit` | Passkey 登录开始 |
| POST | `/api/user/passkey/login/finish` | `PasskeyLoginFinish` | `CriticalRateLimit` | Passkey 登录完成 |
| GET | `/api/user/logout` | `Logout` | — | 登出 |
| POST | `/api/user/epay/notify` | `EpayNotify` | — | Epay 支付通知 |
| GET | `/api/user/epay/notify` | `EpayNotify` | — | Epay 支付通知 |
| GET | `/api/user/groups` | `GetUserGroups` | — | 获取用户组 |

**需要 `UserAuth` 的用户接口：**

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/user/self/groups` | `GetUserGroups` | — | 获取当前用户组 |
| GET | `/api/user/self` | `GetSelf` | — | 获取当前用户信息 |
| GET | `/api/user/models` | `GetUserModels` | — | 获取可用模型 |
| PUT | `/api/user/self` | `UpdateSelf` | — | 更新个人信息 |
| DELETE | `/api/user/self` | `DeleteSelf` | — | 删除账户 |
| GET | `/api/user/token` | `GenerateAccessToken` | — | 生成访问令牌 |
| GET | `/api/user/passkey` | `PasskeyStatus` | — | Passkey 状态 |
| POST | `/api/user/passkey/register/begin` | `PasskeyRegisterBegin` | — | 注册 Passkey 开始 |
| POST | `/api/user/passkey/register/finish` | `PasskeyRegisterFinish` | — | 注册 Passkey 完成 |
| POST | `/api/user/passkey/verify/begin` | `PasskeyVerifyBegin` | — | 验证 Passkey 开始 |
| POST | `/api/user/passkey/verify/finish` | `PasskeyVerifyFinish` | — | 验证 Passkey 完成 |
| DELETE | `/api/user/passkey` | `PasskeyDelete` | — | 删除 Passkey |
| GET | `/api/user/aff` | `GetAffCode` | — | 获取邀请码 |
| GET | `/api/user/topup/info` | `GetTopUpInfo` | — | 充值信息 |
| GET | `/api/user/topup/self` | `GetUserTopUps` | — | 获取充值记录 |
| POST | `/api/user/topup` | `TopUp` | `CriticalRateLimit` | 兑换码充值 |
| POST | `/api/user/pay` | `RequestEpay` | `CriticalRateLimit` | Epay 充值 |
| POST | `/api/user/amount` | `RequestAmount` | — | 查询金额 |
| POST | `/api/user/stripe/pay` | `RequestStripePay` | `CriticalRateLimit` | Stripe 充值 |
| POST | `/api/user/stripe/amount` | `RequestStripeAmount` | — | Stripe 金额 |
| POST | `/api/user/creem/pay` | `RequestCreemPay` | `CriticalRateLimit` | Creem 充值 |
| POST | `/api/user/waffo/pay` | `RequestWaffoPay` | `CriticalRateLimit` | Waffo 充值 |
| POST | `/api/user/aff_transfer` | `TransferAffQuota` | — | 邀请配额转移 |
| PUT | `/api/user/setting` | `UpdateUserSetting` | — | 更新用户设置 |
| GET | `/api/user/2fa/status` | `Get2FAStatus` | — | 2FA 状态 |
| POST | `/api/user/2fa/setup` | `Setup2FA` | — | 设置 2FA |
| POST | `/api/user/2fa/enable` | `Enable2FA` | — | 启用 2FA |
| POST | `/api/user/2fa/disable` | `Disable2FA` | — | 禁用 2FA |
| POST | `/api/user/2fa/backup_codes` | `RegenerateBackupCodes` | — | 重新生成备份码 |
| GET | `/api/user/checkin` | `GetCheckinStatus` | — | 签到状态 |
| POST | `/api/user/checkin` | `DoCheckin` | `TurnstileCheck` | 签到 |
| GET | `/api/user/oauth/bindings` | `GetUserOAuthBindings` | — | OAuth 绑定列表 |
| DELETE | `/api/user/oauth/bindings/:provider_id` | `UnbindCustomOAuth` | — | 解绑 OAuth |

**需要 `AdminAuth` 的用户管理接口：**

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/user/` | `GetAllUsers` | 获取所有用户 |
| GET | `/api/user/topup` | `GetAllTopUps` | 获取所有充值 |
| POST | `/api/user/topup/complete` | `AdminCompleteTopUp` | 管理员完成充值 |
| GET | `/api/user/search` | `SearchUsers` | 搜索用户 |
| GET | `/api/user/:id/oauth/bindings` | `GetUserOAuthBindingsByAdmin` | 查看用户 OAuth 绑定 |
| DELETE | `/api/user/:id/oauth/bindings/:provider_id` | `UnbindCustomOAuthByAdmin` | 管理员解绑 OAuth |
| DELETE | `/api/user/:id/bindings/:binding_type` | `AdminClearUserBinding` | 清除用户绑定 |
| GET | `/api/user/:id` | `GetUser` | 获取用户详情 |
| POST | `/api/user/` | `CreateUser` | 创建用户 |
| POST | `/api/user/manage` | `ManageUser` | 管理用户 |
| PUT | `/api/user/` | `UpdateUser` | 更新用户 |
| DELETE | `/api/user/:id` | `DeleteUser` | 删除用户 |
| DELETE | `/api/user/:id/reset_passkey` | `AdminResetPasskey` | 重置 Passkey |
| GET | `/api/user/2fa/stats` | `Admin2FAStats` | 2FA 统计 |
| DELETE | `/api/user/:id/2fa` | `AdminDisable2FA` | 禁用用户 2FA |

#### 3.1.6 订阅接口 `/api/subscription`

**用户订阅（`UserAuth`）：**

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/subscription/plans` | `GetSubscriptionPlans` | — | 获取订阅计划 |
| GET | `/api/subscription/self` | `GetSubscriptionSelf` | — | 获取我的订阅 |
| PUT | `/api/subscription/self/preference` | `UpdateSubscriptionPreference` | — | 更新订阅偏好 |
| POST | `/api/subscription/epay/pay` | `SubscriptionRequestEpay` | `CriticalRateLimit` | Epay 订阅支付 |
| POST | `/api/subscription/stripe/pay` | `SubscriptionRequestStripePay` | `CriticalRateLimit` | Stripe 订阅支付 |
| POST | `/api/subscription/creem/pay` | `SubscriptionRequestCreemPay` | `CriticalRateLimit` | Creem 订阅支付 |

**管理员订阅（`AdminAuth`）：**

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/subscription/admin/plans` | `AdminListSubscriptionPlans` | 列出订阅计划 |
| POST | `/api/subscription/admin/plans` | `AdminCreateSubscriptionPlan` | 创建订阅计划 |
| PUT | `/api/subscription/admin/plans/:id` | `AdminUpdateSubscriptionPlan` | 更新订阅计划 |
| PATCH | `/api/subscription/admin/plans/:id` | `AdminUpdateSubscriptionPlanStatus` | 更新计划状态 |
| POST | `/api/subscription/admin/bind` | `AdminBindSubscription` | 绑定订阅 |
| GET | `/api/subscription/admin/users/:id/subscriptions` | `AdminListUserSubscriptions` | 用户订阅列表 |
| POST | `/api/subscription/admin/users/:id/subscriptions` | `AdminCreateUserSubscription` | 创建用户订阅 |
| POST | `/api/subscription/admin/user_subscriptions/:id/invalidate` | `AdminInvalidateUserSubscription` | 使订阅失效 |
| DELETE | `/api/subscription/admin/user_subscriptions/:id` | `AdminDeleteUserSubscription` | 删除用户订阅 |

**订阅支付回调（无认证）：**

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| POST | `/api/subscription/epay/notify` | `SubscriptionEpayNotify` | Epay 通知 |
| GET | `/api/subscription/epay/notify` | `SubscriptionEpayNotify` | Epay 通知 |
| GET | `/api/subscription/epay/return` | `SubscriptionEpayReturn` | Epay 返回 |
| POST | `/api/subscription/epay/return` | `SubscriptionEpayReturn` | Epay 返回 |

#### 3.1.7 系统配置接口 `/api/option` (需要 `RootAuth`)

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/option/` | `GetOptions` | 获取系统选项 |
| PUT | `/api/option/` | `UpdateOption` | 更新系统选项 |
| GET | `/api/option/channel_affinity_cache` | `GetChannelAffinityCacheStats` | 渠道亲和缓存统计 |
| DELETE | `/api/option/channel_affinity_cache` | `ClearChannelAffinityCache` | 清除亲和缓存 |
| POST | `/api/option/rest_model_ratio` | `ResetModelRatio` | 重置模型比例 |
| POST | `/api/option/migrate_console_setting` | `MigrateConsoleSetting` | 迁移控制台设置 |

#### 3.1.8 自定义 OAuth 提供商 `/api/custom-oauth-provider` (需要 `RootAuth`)

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| POST | `/api/custom-oauth-provider/discovery` | `FetchCustomOAuthDiscovery` | 获取 OIDC 发现文档 |
| GET | `/api/custom-oauth-provider/` | `GetCustomOAuthProviders` | 列出 OAuth 提供商 |
| GET | `/api/custom-oauth-provider/:id` | `GetCustomOAuthProvider` | 获取提供商详情 |
| POST | `/api/custom-oauth-provider/` | `CreateCustomOAuthProvider` | 创建提供商 |
| PUT | `/api/custom-oauth-provider/:id` | `UpdateCustomOAuthProvider` | 更新提供商 |
| DELETE | `/api/custom-oauth-provider/:id` | `DeleteCustomOAuthProvider` | 删除提供商 |

#### 3.1.9 性能监控 `/api/performance` (需要 `RootAuth`)

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/performance/stats` | `GetPerformanceStats` | 性能统计 |
| DELETE | `/api/performance/disk_cache` | `ClearDiskCache` | 清除磁盘缓存 |
| POST | `/api/performance/reset_stats` | `ResetPerformanceStats` | 重置统计 |
| POST | `/api/performance/gc` | `ForceGC` | 强制 GC |
| GET | `/api/performance/logs` | `GetLogFiles` | 获取日志文件 |
| DELETE | `/api/performance/logs` | `CleanupLogFiles` | 清理日志 |

#### 3.1.10 比例同步 `/api/ratio_sync` (需要 `RootAuth`)

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/ratio_sync/channels` | `GetSyncableChannels` | 获取可同步渠道 |
| POST | `/api/ratio_sync/fetch` | `FetchUpstreamRatios` | 获取上游比例 |

#### 3.1.11 渠道管理 `/api/channel` (需要 `AdminAuth`)

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/channel/` | `GetAllChannels` | — | 获取所有渠道 |
| GET | `/api/channel/search` | `SearchChannels` | — | 搜索渠道 |
| GET | `/api/channel/models` | `ChannelListModels` | — | 渠道模型列表 |
| GET | `/api/channel/models_enabled` | `EnabledListModels` | — | 已启用模型 |
| GET | `/api/channel/:id` | `GetChannel` | — | 获取渠道详情 |
| POST | `/api/channel/:id/key` | `GetChannelKey` | `RootAuth`, `CriticalRateLimit`, `DisableCache`, `SecureVerificationRequired` | 获取渠道密钥 |
| GET | `/api/channel/test` | `TestAllChannels` | — | 测试所有渠道 |
| GET | `/api/channel/test/:id` | `TestChannel` | — | 测试单个渠道 |
| GET | `/api/channel/update_balance` | `UpdateAllChannelsBalance` | — | 更新所有余额 |
| GET | `/api/channel/update_balance/:id` | `UpdateChannelBalance` | — | 更新单个余额 |
| POST | `/api/channel/` | `AddChannel` | — | 添加渠道 |
| PUT | `/api/channel/` | `UpdateChannel` | — | 更新渠道 |
| DELETE | `/api/channel/disabled` | `DeleteDisabledChannel` | — | 删除已禁用渠道 |
| POST | `/api/channel/tag/disabled` | `DisableTagChannels` | — | 按标签禁用 |
| POST | `/api/channel/tag/enabled` | `EnableTagChannels` | — | 按标签启用 |
| PUT | `/api/channel/tag` | `EditTagChannels` | — | 编辑标签渠道 |
| DELETE | `/api/channel/:id` | `DeleteChannel` | — | 删除渠道 |
| POST | `/api/channel/batch` | `DeleteChannelBatch` | — | 批量删除 |
| POST | `/api/channel/fix` | `FixChannelsAbilities` | — | 修复渠道能力 |
| GET | `/api/channel/fetch_models/:id` | `FetchUpstreamModels` | — | 获取上游模型 |
| POST | `/api/channel/fetch_models` | `FetchModels` | — | 批量获取模型 |
| POST | `/api/channel/codex/oauth/start` | `StartCodexOAuth` | — | Codex OAuth 开始 |
| POST | `/api/channel/codex/oauth/complete` | `CompleteCodexOAuth` | — | Codex OAuth 完成 |
| POST | `/api/channel/:id/codex/oauth/start` | `StartCodexOAuthForChannel` | — | 渠道 Codex OAuth 开始 |
| POST | `/api/channel/:id/codex/oauth/complete` | `CompleteCodexOAuthForChannel` | — | 渠道 Codex OAuth 完成 |
| POST | `/api/channel/:id/codex/refresh` | `RefreshCodexChannelCredential` | — | 刷新 Codex 凭据 |
| GET | `/api/channel/:id/codex/usage` | `GetCodexChannelUsage` | — | Codex 用量 |
| POST | `/api/channel/ollama/pull` | `OllamaPullModel` | — | Ollama 拉取模型 |
| POST | `/api/channel/ollama/pull/stream` | `OllamaPullModelStream` | — | Ollama 拉取流 |
| DELETE | `/api/channel/ollama/delete` | `OllamaDeleteModel` | — | 删除 Ollama 模型 |
| GET | `/api/channel/ollama/version/:id` | `OllamaVersion` | — | Ollama 版本 |
| POST | `/api/channel/batch/tag` | `BatchSetChannelTag` | — | 批量设置标签 |
| GET | `/api/channel/tag/models` | `GetTagModels` | — | 标签模型 |
| POST | `/api/channel/copy/:id` | `CopyChannel` | — | 复制渠道 |
| POST | `/api/channel/multi_key/manage` | `ManageMultiKeys` | — | 多密钥管理 |
| POST | `/api/channel/upstream_updates/apply` | `ApplyChannelUpstreamModelUpdates` | — | 应用上游更新 |
| POST | `/api/channel/upstream_updates/apply_all` | `ApplyAllChannelUpstreamModelUpdates` | — | 应用所有更新 |
| POST | `/api/channel/upstream_updates/detect` | `DetectChannelUpstreamModelUpdates` | — | 检测上游更新 |
| POST | `/api/channel/upstream_updates/detect_all` | `DetectAllChannelUpstreamModelUpdates` | — | 检测所有更新 |

#### 3.1.12 令牌管理 `/api/token` (需要 `UserAuth`)

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/token/` | `GetAllTokens` | — | 获取所有令牌 |
| GET | `/api/token/search` | `SearchTokens` | `SearchRateLimit` | 搜索令牌 |
| GET | `/api/token/:id` | `GetToken` | — | 获取令牌 |
| POST | `/api/token/:id/key` | `GetTokenKey` | `CriticalRateLimit`, `DisableCache` | 获取令牌密钥 |
| POST | `/api/token/` | `AddToken` | — | 添加令牌 |
| PUT | `/api/token/` | `UpdateToken` | — | 更新令牌 |
| DELETE | `/api/token/:id` | `DeleteToken` | — | 删除令牌 |
| POST | `/api/token/batch` | `DeleteTokenBatch` | — | 批量删除 |

#### 3.1.13 用量接口 `/api/usage/token` (需要 `TokenAuthReadOnly`)

| 方法 | 路径 | 处理函数 | 附加中间件 | 说明 |
|------|------|----------|-----------|------|
| GET | `/api/usage/token/` | `GetTokenUsage` | `CORS`, `CriticalRateLimit` | 令牌用量 |

#### 3.1.14 兑换码 `/api/redemption` (需要 `AdminAuth`)

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/redemption/` | `GetAllRedemptions` | 获取所有兑换码 |
| GET | `/api/redemption/search` | `SearchRedemptions` | 搜索 |
| GET | `/api/redemption/:id` | `GetRedemption` | 获取详情 |
| POST | `/api/redemption/` | `AddRedemption` | 添加 |
| PUT | `/api/redemption/` | `UpdateRedemption` | 更新 |
| DELETE | `/api/redemption/invalid` | `DeleteInvalidRedemption` | 删除无效 |
| DELETE | `/api/redemption/:id` | `DeleteRedemption` | 删除 |

#### 3.1.15 日志接口 `/api/log`

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| GET | `/api/log/` | `GetAllLogs` | `AdminAuth` | 获取所有日志 |
| DELETE | `/api/log/` | `DeleteHistoryLogs` | `AdminAuth` | 删除历史日志 |
| GET | `/api/log/stat` | `GetLogsStat` | `AdminAuth` | 日志统计 |
| GET | `/api/log/self/stat` | `GetLogsSelfStat` | `UserAuth` | 个人日志统计 |
| GET | `/api/log/channel_affinity_usage_cache` | `GetChannelAffinityUsageCacheStats` | `AdminAuth` | 亲和用量缓存 |
| GET | `/api/log/search` | `SearchAllLogs` | `AdminAuth` | 搜索所有日志 |
| GET | `/api/log/self` | `GetUserLogs` | `UserAuth` | 获取个人日志 |
| GET | `/api/log/self/search` | `SearchUserLogs` | `UserAuth`, `SearchRateLimit` | 搜索个人日志 |
| GET | `/api/log/token` | `GetLogByKey` | `CORS`, `CriticalRateLimit`, `TokenAuthReadOnly` | 按令牌查日志 |

#### 3.1.16 数据统计 `/api/data`

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| GET | `/api/data/` | `GetAllQuotaDates` | `AdminAuth` | 全部配额统计 |
| GET | `/api/data/self` | `GetUserQuotaDates` | `UserAuth` | 个人配额统计 |

#### 3.1.17 其他管理接口

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| GET | `/api/group/` | `GetGroups` | `AdminAuth` | 获取组 |
| GET | `/api/prefill_group/` | `GetPrefillGroups` | `AdminAuth` | 预填充组 |
| POST | `/api/prefill_group/` | `CreatePrefillGroup` | `AdminAuth` | 创建预填充组 |
| PUT | `/api/prefill_group/` | `UpdatePrefillGroup` | `AdminAuth` | 更新预填充组 |
| DELETE | `/api/prefill_group/:id` | `DeletePrefillGroup` | `AdminAuth` | 删除预填充组 |
| GET | `/api/mj/self` | `GetUserMidjourney` | `UserAuth` | MJ 任务 |
| GET | `/api/mj/` | `GetAllMidjourney` | `AdminAuth` | 所有 MJ 任务 |
| GET | `/api/task/self` | `GetUserTask` | `UserAuth` | 个人任务 |
| GET | `/api/task/` | `GetAllTask` | `AdminAuth` | 所有任务 |

#### 3.1.18 供应商/模型元数据 (需要 `AdminAuth`)

**供应商 `/api/vendors`：**

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/vendors/` | `GetAllVendors` | 获取所有供应商 |
| GET | `/api/vendors/search` | `SearchVendors` | 搜索供应商 |
| GET | `/api/vendors/:id` | `GetVendorMeta` | 获取详情 |
| POST | `/api/vendors/` | `CreateVendorMeta` | 创建 |
| PUT | `/api/vendors/` | `UpdateVendorMeta` | 更新 |
| DELETE | `/api/vendors/:id` | `DeleteVendorMeta` | 删除 |

**模型 `/api/models`：**

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/models/sync_upstream/preview` | `SyncUpstreamPreview` | 同步预览 |
| POST | `/api/models/sync_upstream` | `SyncUpstreamModels` | 同步上游模型 |
| GET | `/api/models/missing` | `GetMissingModels` | 缺失模型 |
| GET | `/api/models/` | `GetAllModelsMeta` | 所有模型元数据 |
| GET | `/api/models/search` | `SearchModelsMeta` | 搜索模型 |
| GET | `/api/models/:id` | `GetModelMeta` | 获取详情 |
| POST | `/api/models/` | `CreateModelMeta` | 创建 |
| PUT | `/api/models/` | `UpdateModelMeta` | 更新 |
| DELETE | `/api/models/:id` | `DeleteModelMeta` | 删除 |

#### 3.1.19 部署管理 `/api/deployments` (需要 `AdminAuth`)

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/deployments/settings` | `GetModelDeploymentSettings` | 部署设置 |
| POST | `/api/deployments/settings/test-connection` | `TestIoNetConnection` | 测试连接 |
| GET | `/api/deployments/` | `GetAllDeployments` | 所有部署 |
| GET | `/api/deployments/search` | `SearchDeployments` | 搜索 |
| POST | `/api/deployments/test-connection` | `TestIoNetConnection` | 测试连接 |
| GET | `/api/deployments/hardware-types` | `GetHardwareTypes` | 硬件类型 |
| GET | `/api/deployments/locations` | `GetLocations` | 位置 |
| GET | `/api/deployments/available-replicas` | `GetAvailableReplicas` | 可用副本 |
| POST | `/api/deployments/price-estimation` | `GetPriceEstimation` | 价格估算 |
| GET | `/api/deployments/check-name` | `CheckClusterNameAvailability` | 检查名称 |
| POST | `/api/deployments/` | `CreateDeployment` | 创建部署 |
| GET | `/api/deployments/:id` | `GetDeployment` | 获取部署 |
| GET | `/api/deployments/:id/logs` | `GetDeploymentLogs` | 部署日志 |
| GET | `/api/deployments/:id/containers` | `ListDeploymentContainers` | 容器列表 |
| GET | `/api/deployments/:id/containers/:container_id` | `GetContainerDetails` | 容器详情 |
| PUT | `/api/deployments/:id` | `UpdateDeployment` | 更新部署 |
| PUT | `/api/deployments/:id/name` | `UpdateDeploymentName` | 更新名称 |
| POST | `/api/deployments/:id/extend` | `ExtendDeployment` | 延长部署 |
| DELETE | `/api/deployments/:id` | `DeleteDeployment` | 删除部署 |

### 3.2 Dashboard 路由

**路由组中间件：** `RouteTag("old_api")`, `Gzip`, `GlobalAPIRateLimit`, `CORS`, `TokenAuth`

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/dashboard/billing/subscription` | `GetSubscription` | OpenAI 兼容计费 |
| GET | `/v1/dashboard/billing/subscription` | `GetSubscription` | OpenAI 兼容计费 |
| GET | `/dashboard/billing/usage` | `GetUsage` | OpenAI 兼容用量 |
| GET | `/v1/dashboard/billing/usage` | `GetUsage` | OpenAI 兼容用量 |

### 3.3 Relay 路由（AI API 代理）

**引擎级中间件：** `CORS`, `DecompressRequestMiddleware`, `BodyStorageCleanup`, `StatsMiddleware`

#### 3.3.1 模型列表

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| GET | `/v1/models` | `ListModels` (按 header 分发) | `TokenAuth` | OpenAI/Claude/Gemini 模型列表 |
| GET | `/v1/models/:model` | `RetrieveModel` | `TokenAuth` | 获取模型详情 |
| GET | `/v1beta/models` | `ListModels(Gemini)` | `TokenAuth` | Gemini 原生模型列表 |
| GET | `/v1beta/openai/models` | `ListModels(OpenAI)` | `TokenAuth` | Gemini 兼容模型列表 |

#### 3.3.2 Playground

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| POST | `/pg/chat/completions` | `Playground` | `SystemPerformanceCheck`, `UserAuth`, `Distribute` | 聊天 Playground |

#### 3.3.3 AI API 代理 `/v1`

**中间件链：** `SystemPerformanceCheck` → `TokenAuth` → `ModelRequestRateLimit` → `Distribute`

| 方法 | 路径 | Relay 格式 | 说明 |
|------|------|-----------|------|
| GET | `/v1/realtime` | `OpenAIRealtime` | WebSocket 实时 API |
| POST | `/v1/messages` | `Claude` | Claude Messages API |
| POST | `/v1/completions` | `OpenAI` | OpenAI Completions |
| POST | `/v1/chat/completions` | `OpenAI` | **核心：Chat Completions** |
| POST | `/v1/responses` | `OpenAIResponses` | OpenAI Responses API |
| POST | `/v1/responses/compact` | `OpenAIResponsesCompaction` | Responses 压缩版 |
| POST | `/v1/edits` | `OpenAIImage` | 图片编辑 |
| POST | `/v1/images/generations` | `OpenAIImage` | 图片生成 |
| POST | `/v1/images/edits` | `OpenAIImage` | 图片编辑 |
| POST | `/v1/embeddings` | `Embedding` | Embeddings |
| POST | `/v1/audio/transcriptions` | `OpenAIAudio` | 语音转文字 |
| POST | `/v1/audio/translations` | `OpenAIAudio` | 语音翻译 |
| POST | `/v1/audio/speech` | `OpenAIAudio` | 文字转语音 |
| POST | `/v1/rerank` | `Rerank` | 重排序 |
| POST | `/v1/engines/:model/embeddings` | `Gemini` | Gemini Embeddings |
| POST | `/v1/models/*path` | `Gemini` | Gemini 原生 API |
| POST | `/v1/moderations` | `OpenAI` | 内容审核 |
| POST | `/v1beta/models/*path` | `Gemini` | Gemini Beta API |

**未实现的端点（返回 501）：**
`/v1/images/variations`, `/v1/files`, `/v1/fine-tunes`, `/v1/models/:model` (DELETE)

#### 3.3.4 Midjourney `/mj` 和 `/:mode/mj`

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| GET | `/mj/image/:id` | `RelayMidjourneyImage` | 无认证 | 获取 MJ 图片 |
| POST | `/mj/submit/action` | `RelayMidjourney` | `TokenAuth`, `Distribute` | 动作 |
| POST | `/mj/submit/imagine` | `RelayMidjourney` | 同上 | 生成图片 |
| POST | `/mj/submit/change` | `RelayMidjourney` | 同上 | 变体 |
| POST | `/mj/submit/describe` | `RelayMidjourney` | 同上 | 描述 |
| POST | `/mj/submit/blend` | `RelayMidjourney` | 同上 | 混合 |
| POST | `/mj/submit/edits` | `RelayMidjourney` | 同上 | 编辑 |
| POST | `/mj/submit/video` | `RelayMidjourney` | 同上 | 视频 |
| POST | `/mj/submit/shorten` | `RelayMidjourney` | 同上 | 缩短 |
| POST | `/mj/submit/modal` | `RelayMidjourney` | 同上 | 弹窗 |
| GET | `/mj/task/:id/fetch` | `RelayMidjourney` | 同上 | 获取任务 |
| GET | `/mj/task/:id/image-seed` | `RelayMidjourney` | 同上 | 获取种子 |
| POST | `/mj/task/list-by-condition` | `RelayMidjourney` | 同上 | 条件查询 |
| POST | `/mj/insight-face/swap` | `RelayMidjourney` | 同上 | 换脸 |
| POST | `/mj/submit/upload-discord-images` | `RelayMidjourney` | 同上 | 上传图片 |

#### 3.3.5 Suno `/suno`

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| POST | `/suno/submit/:action` | `RelayTask` | `TokenAuth`, `Distribute` | 提交任务 |
| POST | `/suno/fetch` | `RelayTaskFetch` | 同上 | 批量获取 |
| GET | `/suno/fetch/:id` | `RelayTaskFetch` | 同上 | 获取单个 |

### 3.4 Video 路由

| 方法 | 路径 | 处理函数 | 中间件 | 说明 |
|------|------|----------|--------|------|
| GET | `/v1/videos/:task_id/content` | `VideoProxy` | `TokenOrUserAuth` | 视频内容代理 |
| POST | `/v1/video/generations` | `RelayTask` | `TokenAuth`, `Distribute` | 视频生成 |
| GET | `/v1/video/generations/:task_id` | `RelayTaskFetch` | 同上 | 获取视频任务 |
| POST | `/v1/videos/:video_id/remix` | `RelayTask` | 同上 | 视频混剪 |
| POST | `/v1/videos` | `RelayTask` | 同上 | 视频生成 |
| GET | `/v1/videos/:task_id` | `RelayTaskFetch` | 同上 | 获取视频任务 |
| POST | `/kling/v1/videos/text2video` | `RelayTask` | `KlingRequestConvert`, `TokenAuth`, `Distribute` | Kling 文生视频 |
| POST | `/kling/v1/videos/image2video` | `RelayTask` | 同上 | Kling 图生视频 |
| GET | `/kling/v1/videos/text2video/:task_id` | `RelayTaskFetch` | 同上 | Kling 任务 |
| GET | `/kling/v1/videos/image2video/:task_id` | `RelayTaskFetch` | 同上 | Kling 任务 |
| POST | `/jimeng/` | `RelayTask` | `JimengRequestConvert`, `TokenAuth`, `Distribute` | 即梦生成 |

### 3.5 Web 路由（静态资源）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/*` | 嵌入的 `web/dist` 静态文件 |
| * | NoRoute | SPA 回退到 `index.html`；`/v1`、`/api`、`/assets` 前缀返回 404 |

---

## 4. 中间件分析

### 4.1 中间件清单与 Workers 兼容性

| 中间件 | 文件 | 功能说明 | Workers 兼容性 |
|--------|------|----------|---------------|
| `RequestId()` | `request-id.go` | 生成唯一请求 ID | ✅ 完全兼容 |
| `PoweredBy()` | `cors.go` | 添加版本 header | ✅ 完全兼容 |
| `I18n()` | `i18n.go` | 语言检测 | ✅ 完全兼容 |
| `CORS()` | `cors.go` | CORS 头部处理 | ✅ 完全兼容 |
| `Cache()` | `cache.go` | 静态资源缓存头 | ✅ 完全兼容 |
| `DisableCache()` | `disable-cache.go` | 禁用缓存头 | ✅ 完全兼容 |
| `RouteTag()` | `logger.go` | 路由分类标记 | ✅ 完全兼容 |
| `RelayPanicRecover()` | `recover.go` | Panic 恢复 | ✅ 完全兼容（JS try/catch） |
| `JimengRequestConvert()` | `jimeng_adapter.go` | 即梦请求格式转换 | ✅ 完全兼容 |
| `KlingRequestConvert()` | `kling_adapter.go` | Kling 请求格式转换 | ✅ 完全兼容 |
| `TurnstileCheck()` | `turnstile-check.go` | Cloudflare Turnstile 验证 | ⚠️ 需要适配（session → KV/cookie） |
| `SecureVerificationRequired()` | `secure_verification.go` | 安全验证（2FA 门控） | ⚠️ 需要适配（session → KV/cookie） |
| `DecompressRequestMiddleware()` | `gzip.go` | Gzip/Brotli 解压 | ⚠️ 需要适配（gzip OK，brotli 可能需要 WASM） |
| `SetUpLogger()` | `logger.go` | 请求日志 | ⚠️ 需要适配（改用 console.log/Workers Analytics） |
| `UserAuth()` / `AdminAuth()` / `RootAuth()` | `auth.go` | Session 认证 | ⚠️ 需要适配（session → JWT/KV） |
| `TryUserAuth()` | `auth.go` | 软认证 | ⚠️ 同上 |
| `TokenAuth()` | `auth.go` | API Token 认证 | ❌ 不兼容（重度 DB 依赖） |
| `TokenAuthReadOnly()` | `auth.go` | 只读 Token 认证 | ❌ 不兼容（直接 DB 调用） |
| `Distribute()` | `distributor.go` | 渠道选择/分发 | ❌ 不兼容（重度 DB/缓存/服务依赖） |
| `GlobalWebRateLimit()` | `rate-limit.go` | Web 限流 | ❌ 不兼容（Redis/内存状态） |
| `GlobalAPIRateLimit()` | `rate-limit.go` | API 限流 | ❌ 不兼容（同上） |
| `CriticalRateLimit()` | `rate-limit.go` | 关键操作限流 | ❌ 不兼容（同上） |
| `SearchRateLimit()` | `rate-limit.go` | 搜索限流 | ❌ 不兼容（同上） |
| `ModelRequestRateLimit()` | `model-rate-limit.go` | 按模型限流 | ❌ 不兼容（同上） |
| `EmailVerificationRateLimit()` | `email-verification-rate-limit.go` | 邮箱验证限流 | ❌ 不兼容（同上） |
| `StatsMiddleware()` | `stats.go` | 活跃连接计数 | ❌ 不兼容（内存原子计数器） |
| `SystemPerformanceCheck()` | `performance.go` | 系统性能门控 | ❌ 不兼容（OS 级指标） |
| `BodyStorageCleanup()` | `body_cleanup.go` | 请求体存储清理 | ❌ 不兼容（文件系统） |

### 4.2 全局中间件链

服务器启动时应用的全局中间件：

```
gin.CustomRecovery()     → Panic 恢复
middleware.RequestId()   → 请求 ID
middleware.PoweredBy()   → 版本头
middleware.I18n()        → 国际化
middleware.SetUpLogger() → 请求日志
sessions.Sessions()      → Cookie Session
```

---

## 5. 控制器层分析

控制器按业务领域分为以下几大类：

### 5.1 认证与用户管理

| 文件 | 关键功能 | 依赖 |
|------|----------|------|
| `user.go` | 登录、注册、个人信息 CRUD、充值 | DB, Sessions, Redis |
| `oauth.go` | OAuth 登录/绑定 | DB, 外部 OAuth, Sessions |
| `custom_oauth.go` | 自定义 OAuth CRUD | DB, HTTP Client (OIDC) |
| `passkey.go` | WebAuthn 注册/登录/验证 | DB, Sessions, WebAuthn 库 |
| `twofa.go` | TOTP 2FA | DB, TOTP 库 |
| `wechat.go` | 微信 OAuth | 外部微信 API |
| `telegram.go` | Telegram OAuth | Telegram API |
| `codex_oauth.go` | Codex OAuth | Sessions, 外部 Codex OAuth |

### 5.2 AI API 代理（核心）

| 文件 | 关键功能 | 依赖 |
|------|----------|------|
| `relay.go` | **核心中转入口** — 分发到格式处理器，重试逻辑，配额管理 | relay 包, service/quota, model/channel |
| `playground.go` | 聊天 Playground | 同 relay.go |

### 5.3 渠道管理

| 文件 | 关键功能 | 依赖 |
|------|----------|------|
| `channel.go` | 渠道 CRUD、多密钥管理、批量操作 | DB, HTTP Client |
| `channel-billing.go` | 上游余额查询 | HTTP Client, 多种提供商 API |
| `channel-test.go` | 渠道测试（含后台定时测试） | HTTP Client, DB |
| `channel_upstream_update.go` | 检测/应用上游模型更新 | DB, HTTP Client, 通知 |

### 5.4 计费与支付

| 文件 | 关键功能 | 依赖 |
|------|----------|------|
| `billing.go` | OpenAI 兼容计费接口 | DB |
| `topup.go` | Epay 充值 | Epay SDK, DB |
| `topup_stripe.go` | Stripe 充值 | Stripe SDK, DB |
| `topup_creem.go` | Creem 充值 | Creem API, DB |
| `topup_waffo.go` | Waffo 充值 | Waffo SDK, DB |
| `subscription*.go` | 订阅管理与支付 | 多种支付 SDK, DB |

### 5.5 异步任务

| 文件 | 关键功能 | 依赖 |
|------|----------|------|
| `midjourney.go` | Midjourney 任务管理 + 后台轮询 | DB, HTTP Client |
| `task.go` | 通用异步任务管理 + 后台轮询 | DB, HTTP Client |
| `video_proxy.go` | 视频内容代理 | HTTP Client |

### 5.6 系统管理

| 文件 | 关键功能 | 依赖 |
|------|----------|------|
| `option.go` | 系统选项管理 | DB |
| `misc.go` | 状态、邮箱验证、密码重置 | SMTP, DB |
| `setup.go` | 初始化向导 | DB |
| `performance.go` | 性能监控、GC、磁盘缓存 | OS 文件系统 |
| `log.go` | 日志管理 | DB (LOG_DB) |
| `deployment.go` | io.net GPU 部署 | io.net API |
| `model.go` | 模型列表（OpenAI 兼容） | Model Cache |
| `model_meta.go` / `vendor_meta.go` | 模型/供应商元数据 | DB |
| `ratio_sync.go` | 比例同步 | HTTP Client, DB |

---

## 6. Relay（AI API 代理）层分析

### 6.1 Relay 架构

```
客户端请求
    │
    ▼
controller/relay.go → Relay()
    │
    ├── helper.GetAndValidateRequest()   → 解析验证请求体
    ├── relaycommon.GenRelayInfo()        → 构建 RelayInfo 上下文
    ├── middleware/distributor.go          → 选择渠道（组→模型→优先级）
    │
    ▼
relay/compatible_handler.go → TextHelper() / ImageHelper() / AudioHelper()
    │
    ├── GetAdaptor(apiType)              → 获取提供商特定适配器
    ├── adaptor.ConvertOpenAIRequest()   → 转换为上游格式
    ├── adaptor.DoRequest()              → 发送 HTTP 请求到上游
    ├── adaptor.DoResponse()             → 处理响应（流式或非流式）
    │
    ▼
service/quota.go → PostTextConsumeQuota() → 计费结算
```

**适配器接口：**
- `Adaptor` — 同步请求/响应（Chat、Embedding、Image、Audio、Rerank、Responses）
- `TaskAdaptor` — 异步任务生命周期（Video、Music），包含预估计费、提交计费、完成计费、轮询方法

**请求转换链 (`RequestConversionChain`)：**
请求可能经过多次格式转换，例如：
- Claude 格式 → OpenAI 格式 → Gemini 格式
- Gemini 格式 → OpenAI 格式

### 6.2 Provider 适配器列表

| 适配器 | 提供商 | 说明 |
|--------|--------|------|
| `openai/` | OpenAI, Azure, 兼容提供商 | 最完整的适配器，含 Responses API、Realtime、Audio |
| `claude/` | Anthropic Claude | Claude Messages API |
| `gemini/` | Google Gemini | Gemini 原生 + OpenAI 兼容 |
| `aws/` | AWS Bedrock | Claude via Bedrock |
| `vertex/` | Google Vertex AI | Gemini via Vertex |
| `ali/` | 阿里通义 | 含 Rerank、Image (Wan) |
| `baidu/` / `baidu_v2/` | 百度文心 | v1/v2 两个版本 |
| `tencent/` | 腾讯混元 | |
| `zhipu/` / `zhipu_4v/` | 智谱 GLM | v3/v4 两个版本 |
| `xunfei/` | 讯飞星火 | **WebSocket 协议** |
| `volcengine/` | 字节火山引擎/豆包 | TTS 使用 **WebSocket** |
| `deepseek/` | DeepSeek | |
| `minimax/` | MiniMax | |
| `moonshot/` | 月之暗面 | |
| `cohere/` | Cohere | |
| `ollama/` | Ollama（本地） | |
| `cloudflare/` | Cloudflare Workers AI | |
| `siliconflow/` | 硅基流动 | |
| `mistral/` | Mistral AI | |
| `perplexity/` | Perplexity | |
| `xai/` | xAI (Grok) | |
| `coze/` | Coze（字节） | |
| `dify/` | Dify | |
| `jina/` | Jina AI | Rerank/Embedding |
| `jimeng/` | 即梦 | 图片生成 |
| `palm/` | Google PaLM（遗留） | |
| `replicate/` | Replicate | |
| `mokaai/` | MokaAI | |
| `codex/` | OpenAI Codex | |
| `submodel/` | Submodel（元适配器） | |
| `openrouter/` | OpenRouter | |
| `lingyiwanwu/` | 零一万物 | |
| `ai360/` | 360 AI | |
| `xinference/` | Xinference | |

**异步任务适配器（视频/音乐生成）：**

| 适配器 | 提供商 | 生成类型 |
|--------|--------|----------|
| `task/ali/` | 阿里 | 视频 |
| `task/kling/` | 快手可灵 | 视频 |
| `task/jimeng/` | 即梦 | 视频 |
| `task/vertex/` | Google Vertex (Veo) | 视频 |
| `task/gemini/` | Google Gemini | 视频 |
| `task/vidu/` | Vidu | 视频 |
| `task/doubao/` | 字节豆包 | 视频 |
| `task/sora/` | OpenAI Sora | 视频 |
| `task/hailuo/` | 海螺/MiniMax | 视频 |
| `task/suno/` | Suno | 音乐 |

### 6.3 流式传输实现

**SSE（Server-Sent Events）：**
- `relay/helper/stream_scanner.go` — 核心流处理器
- 使用 `bufio.Scanner` 扫描 HTTP 响应体（64KB 初始缓冲，最大 64MB）
- 解析 `data:` 前缀的 SSE 行
- 缓冲通道 + 处理协程
- **Ping 机制**：可配置间隔（默认 10s）发送 keep-alive
- **流超时**：超时未收到数据则关闭连接
- **最大 Ping 时长**：30 分钟
- 客户端断连检测
- `sync.WaitGroup` + 多协程 + Panic 恢复

**WebSocket：**
- `relay/websocket.go` — OpenAI Realtime API 代理
- 升级 HTTP 到 WebSocket，双向代理消息
- 讯飞星火适配器原生使用 WebSocket
- 火山引擎 TTS 使用 WebSocket

---

## 7. 数据模型与存储依赖

### 7.1 数据库模型

| 模型 | 表名 | 说明 |
|------|------|------|
| `User` | `users` | 用户账户（配额、组、角色、状态、OAuth 绑定） |
| `Token` | `tokens` | API 密钥（配额、限流、模型限制、过期） |
| `Channel` | `channels` | 上游提供商配置（端点、密钥、模型、优先级、标签） |
| `Ability` | `abilities` | 组-模型-渠道映射 |
| `Log` | `logs` | API 使用日志 |
| `Option` | `options` | 系统键值配置 |
| `Redemption` | `redemptions` | 兑换码 |
| `Task` | `tasks` | 异步任务（视频/音乐） |
| `Midjourney` | `midjourneys` | Midjourney 特定任务 |
| `QuotaData` | `quota_data` | 聚合配额使用数据 |
| `Subscription` | `subscriptions` | 订阅计划 |
| `UserSubscription` | `user_subscriptions` | 用户订阅实例 |
| `TopUp` | `top_ups` | 支付交易记录 |
| `Pricing` | `pricings` | 每模型定价 |
| `ModelMeta` | `model_metas` | 模型元数据 |
| `VendorMeta` | `vendor_metas` | 供应商元数据 |
| `Passkey` | `passkeys` | WebAuthn 凭据 |
| `TwoFA` | `two_fas` | TOTP 密钥和备份码 |
| `CustomOAuthProvider` | `custom_oauth_providers` | 自定义 OAuth 提供商 |
| `UserOAuthBinding` | `user_oauth_bindings` | 用户 OAuth 绑定 |
| `PrefillGroup` | `prefill_groups` | 预配置组模板 |
| `Setup` | `setups` | 系统初始化记录 |
| `Checkin` | `checkins` | 每日签到记录 |
| `MissingModel` | `missing_models` | 缺失模型追踪 |

### 7.2 缓存架构

**两级缓存：**

1. **内存缓存** (`model/channel_cache.go`)：
   - `group2model2channels` map：组 → 模型 → 排序的渠道 ID 列表（按优先级）
   - `channelsIDM` map：渠道 ID → 完整 Channel 对象
   - 每 `SYNC_FREQUENCY` 秒同步一次（默认 60s）
   - `channelSyncLock` (RWMutex) 保护

2. **Redis 缓存**（启用 `REDIS_CONN_STRING` 时）：
   - **Token 缓存** (`token_cache.go`)：Redis Hash (`token:{hmac_key}`)
   - **User 缓存** (`user_cache.go`)：Redis Hash (`user:{id}`)
   - 支持原子字段更新（`HINCRBY` 配额，`HSET` 单字段）
   - TTL 匹配 `SyncFrequency`

3. **渠道亲和缓存** (`service/channel_affinity.go`)：
   - 混合内存 + Redis 缓存
   - 基于模板的键提取（从请求体提取 conversation_id 等）
   - 粘性路由决策

---

## 8. 后台任务与定时作业

| 任务 | 触发方式 | 说明 |
|------|----------|------|
| `SyncChannelCache()` | 每 `SYNC_FREQUENCY`s（默认 60s） | 从 DB 重载所有渠道和能力到内存 |
| `SyncOptions()` | 每 `SYNC_FREQUENCY`s | 从 DB 重载系统选项 |
| `UpdateQuotaData()` | 每 `DATA_EXPORT_INTERVAL` 分钟 | 将聚合配额数据从内存刷入 DB |
| `AutomaticallyTestChannels()` | 每 `CHANNEL_TEST_FREQUENCY` | 测试所有渠道，禁用失败的 |
| `AutomaticallyUpdateChannels()` | 每 `CHANNEL_UPDATE_FREQUENCY` | 更新渠道余额 |
| `UpdateMidjourneyTaskBulk()` | 持续循环 | 轮询上游 Midjourney 任务状态 |
| `UpdateTaskBulk()` | 持续循环 | 轮询上游通用任务状态 |
| `StartChannelUpstreamModelUpdateTask()` | 定期 | 检测上游模型列表变更 |
| `StartCodexCredentialAutoRefreshTask()` | 每 10 分钟 | 刷新 Codex OAuth 凭据 |
| `StartSubscriptionQuotaResetTask()` | 每 1 分钟 | 重置到期订阅配额 |
| `InitBatchUpdater()` | 每 `BATCH_UPDATE_INTERVAL`s | 批量配额更新以减少 DB 写入 |
| `StartSystemMonitor()` | 定期 | 监控系统资源（CPU、内存、磁盘） |

---

## 9. Cloudflare Workers 迁移可行性评估

### 9.1 可直接迁移的功能

以下功能可以直接在 Cloudflare Workers 中实现，无需重大修改：

| 功能 | 对应接口 | 说明 |
|------|----------|------|
| **CORS 处理** | 所有接口 | 纯头部操作 |
| **请求 ID 生成** | 所有接口 | `crypto.randomUUID()` |
| **国际化** | 所有接口 | 纯字符串解析 |
| **缓存头设置** | 静态资源 | 纯头部操作 |
| **请求格式转换** | Kling/Jimeng 适配器 | 纯 JSON 转换 |
| **AI API 请求转发（基本）** | `/v1/chat/completions` 等 | Workers `fetch()` |
| **SSE 流式代理** | 所有流式接口 | Workers 支持 `ReadableStream` 和 SSE |
| **静态资源托管** | `web/*` | Workers Sites / Pages |
| **支付 Webhook 处理** | Stripe/Creem/Waffo webhook | 签名验证 + HTTP 响应 |
| **公开信息接口** | `/api/status`, `/api/notice` 等 | 简单 KV 读取 |
| **Turnstile 验证** | 注册/登录等 | Workers 原生 Turnstile 支持 |
| **模型列表接口** | `/v1/models` | 缓存数据返回 |

### 9.2 需要适配的功能

| 功能 | Workers 替代方案 | 改造复杂度 |
|------|------------------|-----------|
| **数据库访问 (GORM)** | Cloudflare D1 (SQLite) 或 Hyperdrive (PostgreSQL/MySQL 代理) | 🔴 高 — 需要重写所有 ORM 调用 |
| **Redis 缓存** | Cloudflare KV + Durable Objects，或 Upstash Redis (HTTP API) | 🟡 中 — 需要替换所有 Redis 操作 |
| **Session 管理** | 签名 Cookie + KV 存储，或 JWT 无状态 token | 🟡 中 |
| **限流** | Cloudflare Rate Limiting API，或 Durable Objects 计数器 | 🟡 中 |
| **Token 认证** | D1 查询或 KV 缓存 token 信息 | 🟡 中 |
| **渠道选择/分发** | 需要完全重写，使用 KV 缓存渠道配置 | 🔴 高 |
| **配额管理** | Durable Objects 原子计数，或 D1 事务 | 🔴 高 — 并发安全要求 |
| **邮件发送** | 外部 SMTP 服务 API (SendGrid, Mailgun 等) | 🟢 低 |
| **OAuth 流程** | Workers 可以发起 HTTP 请求，需要状态管理 | 🟡 中 |
| **日志系统** | Workers Analytics Engine + Logpush | 🟡 中 |
| **Gzip 解压** | Workers 原生支持 `DecompressionStream` | 🟢 低 |
| **请求体大小限制** | Workers 有 100MB 请求体限制（企业版） | 🟢 低 — 注意默认限制 |

### 9.3 无法实现的功能及原因

#### ❌ 1. WebSocket 双向代理 (`/v1/realtime`)

**原因：** Cloudflare Workers 支持 WebSocket，但存在以下限制：
- Workers 的 CPU 时间限制（Free: 10ms, Paid: 30s per request）不适合长时间运行的 WebSocket 连接
- Realtime API 需要维持客户端-服务器之间的持久双向连接，单个 Worker 请求不适合作为长时间运行的代理
- **Durable Objects 可以维持 WebSocket 连接**（每个对象最多可处理 ~30,000 个并发 WebSocket），但增加了显著的架构复杂度

**可能的替代方案：** 使用 Durable Objects 作为 WebSocket 代理，但需要完全重新设计架构

#### ❌ 2. 讯飞星火 WebSocket 适配器

**原因：** 讯飞星火 API 使用 WebSocket 协议进行通信（不是 HTTP），Workers 作为客户端发起 WebSocket 连接有限制：
- Workers 可以接收入站 WebSocket，也可以通过 `fetch` 发起出站 WebSocket
- 但需要在单个请求的生命周期内完成，受 CPU 时间限制
- 对于流式响应，可能超出执行时间限制

**可能的替代方案：** 降级为不支持讯飞星火，或使用 Durable Objects

#### ❌ 3. 火山引擎 TTS WebSocket 适配器

**原因：** 同讯飞星火，使用 WebSocket 协议进行 TTS 通信

#### ❌ 4. 后台定时任务/Cron Jobs

**原因：** Workers 没有原生的持久运行后台进程

| 任务 | 不可行原因 | 替代方案 |
|------|-----------|----------|
| `SyncChannelCache` | 需要持续内存状态 | Workers Cron Triggers + KV |
| `UpdateQuotaData` | 需要内存聚合 | Durable Objects 或外部服务 |
| `AutomaticallyTestChannels` | 长时间运行 | Workers Cron Triggers |
| `UpdateMidjourneyTaskBulk` | 持续循环轮询 | Workers Cron Triggers + Queue |
| `UpdateTaskBulk` | 持续循环轮询 | Workers Cron Triggers + Queue |
| `CodexCredentialRefresh` | 定期刷新 | Workers Cron Triggers |
| `SubscriptionQuotaReset` | 每分钟检查 | Workers Cron Triggers |
| `SystemMonitor` | OS 级指标 | **完全不可行** |

**可能的替代方案：** Cloudflare Workers 的 Cron Triggers 可以实现大部分定时任务，但频率最高为每分钟一次，且每次调用有执行时间限制

#### ❌ 5. 系统性能监控 (`SystemPerformanceCheck`)

**原因：** Workers 运行在 V8 隔离环境中，无法访问：
- 主机 CPU 使用率
- 内存使用率
- 磁盘使用率
- 操作系统级指标

**不可替代：** Workers 有自己的资源限制，由 Cloudflare 平台管理

#### ❌ 6. 磁盘缓存 (`BodyStorageCleanup`, `DiskCache`)

**原因：** Workers 没有文件系统访问权限，无法：
- 将大请求体溢出到磁盘
- 缓存文件到本地磁盘
- 管理临时文件

**可能的替代方案：** R2 存储（对象存储），但延迟高于本地磁盘

#### ❌ 7. 内存状态共享（活跃连接计数、批量更新缓冲）

**原因：** Workers V8 隔离环境是临时的，不同请求可能在不同隔离环境中执行：
- `sync/atomic` 原子计数器不在隔离环境间共享
- `InMemoryRateLimiter` 的状态不持久
- 批量更新的内存缓冲无法跨请求持久化

**可能的替代方案：** Durable Objects（提供全局唯一的持久状态实例）

#### ❌ 8. 强制 GC / pprof / 日志文件管理

**原因：** 运行时管理功能不适用于 Workers 环境：
- 无法控制 V8 垃圾回收
- 无法运行 Go pprof
- 无日志文件（只有流式日志输出）

#### ❌ 9. 长时间 SSE 流式传输（超过 30s）

**原因：** Workers 付费版 CPU 时间限制为 30 秒（Wall time 可以更长，但 CPU 密集操作受限）。对于大多数 Chat Completions：
- 简单对话通常在 30s 内完成 ✅
- 复杂推理（如 Claude 长思考）可能超过限制 ⚠️
- 30 分钟的 Ping 超时不可能实现 ❌

**注意：** Workers Unbound 模式下 Wall time 最多 15 分钟（企业版 30 分钟），CPU 时间最多 30 秒。大部分流式请求在 Wall time 内可以完成，因为流式传输主要是 I/O 等待而非 CPU 计算。

#### ❌ 10. Ollama 本地模型管理

**原因：** Ollama 是本地部署的模型运行时：
- Workers 无法连接到本地网络中的 Ollama 实例
- 拉取模型、版本管理等操作不适用

**不可替代：** 需要移除此功能

#### ❌ 11. io.net GPU 部署管理

**原因：** 涉及长时间运行的部署操作和状态管理
- 可以通过 API 调用部分实现
- 但日志流、容器管理等需要长连接

### 9.4 Cloudflare 替代方案对照表

| Go 原始组件 | Cloudflare Workers 替代 | 适配难度 |
|------------|------------------------|---------|
| **Gin Web 框架** | Workers Router (itty-router / Hono) | 🟡 中 |
| **GORM / SQLite/MySQL/PG** | D1 (SQLite) + Drizzle ORM | 🔴 高 |
| **Redis** | KV + Durable Objects + Upstash Redis | 🟡 中 |
| **Cookie Sessions** | 签名 Cookie / JWT / KV | 🟡 中 |
| **go-redis 限流** | Cloudflare Rate Limiting / Durable Objects | 🟡 中 |
| **goroutine 后台任务** | Cron Triggers + Queues | 🟡 中 |
| **sync.Mutex / sync/atomic** | Durable Objects (单实例并发安全) | 🔴 高 |
| **net/http Client** | Workers `fetch()` | 🟢 低 |
| **bufio.Scanner SSE** | Workers `ReadableStream` + TransformStream | 🟢 低 |
| **WebSocket 代理** | Durable Objects WebSocket | 🔴 高 |
| **embed.FS 静态文件** | Workers Sites / Pages | 🟢 低 |
| **os 文件系统** | R2 对象存储 | 🟡 中 |
| **SMTP 邮件** | 外部邮件 API (SendGrid 等) | 🟢 低 |
| **Go i18n** | JS i18n 库 | 🟢 低 |
| **GORM Auto-Migration** | D1 Migrations (Wrangler) | 🟡 中 |

---

## 10. 推荐的迁移策略

### 10.1 建议采用渐进式迁移

考虑到项目的复杂度（~241 个端点、37 个提供商适配器、20+ 后台任务），建议采用**分阶段渐进式迁移**：

### 第一阶段：核心 AI 代理功能

**目标：** 将核心 AI API 代理功能迁移到 Workers

**迁移范围：**
- `/v1/chat/completions` — Chat Completions 代理
- `/v1/completions` — Completions 代理
- `/v1/embeddings` — Embeddings 代理
- `/v1/images/generations` — 图片生成代理
- `/v1/audio/*` — 音频接口代理
- `/v1/rerank` — Rerank 代理
- `/v1/messages` — Claude Messages 代理
- `/v1/models` — 模型列表

**技术方案：**
- 使用 Hono 框架作为路由层
- Token 认证通过 D1 或 KV 缓存实现
- 渠道配置存储在 KV 中
- 使用 Workers `fetch()` 转发请求到上游
- SSE 流使用 `ReadableStream` 管道

**Workers 代码结构示例：**
```
src/
├── index.ts          — 主入口 + 路由
├── middleware/
│   ├── auth.ts       — Token 认证
│   ├── cors.ts       — CORS
│   └── rateLimit.ts  — 限流（使用 Rate Limiting API）
├── relay/
│   ├── handler.ts    — 核心代理处理
│   ├── stream.ts     — SSE 流处理
│   └── adapters/
│       ├── openai.ts
│       ├── claude.ts
│       ├── gemini.ts
│       └── ...
├── services/
│   ├── quota.ts      — 配额管理
│   └── channel.ts    — 渠道选择
└── bindings.d.ts     — D1/KV/DO 绑定类型
```

### 第二阶段：管理 API

**迁移范围：**
- `/api/user/*` — 用户管理
- `/api/token/*` — 令牌管理
- `/api/log/*` — 日志查询
- 认证系统（JWT 替代 Session）

**技术方案：**
- D1 作为主数据库
- JWT 无状态认证
- Workers Analytics Engine 作为日志存储

### 第三阶段：异步任务 + 支付

**迁移范围：**
- `/mj/*`, `/suno/*` — Midjourney/Suno 代理
- `/v1/video/*`, `/kling/*`, `/jimeng/*` — 视频生成
- 支付 webhook
- 订阅管理

**技术方案：**
- Queues 处理异步任务轮询
- Cron Triggers 替代后台循环
- Durable Objects 管理任务状态

### 第四阶段：管理后台

**迁移范围：**
- `/api/channel/*` — 渠道管理
- `/api/option/*` — 系统配置
- `/api/subscription/*` — 订阅管理
- 前端静态资源

**技术方案：**
- Workers Sites 或 Pages 托管前端
- D1 + KV 存储配置

### 10.2 需要放弃或降级的功能

| 功能 | 建议 |
|------|------|
| WebSocket Realtime API | 使用 Durable Objects 实现，或暂时不迁移 |
| 讯飞星火适配器 | 不迁移（使用 Durable Objects 或外部代理） |
| 火山引擎 TTS WebSocket | 不迁移 |
| Ollama 本地模型 | 不迁移（本质上不兼容） |
| 系统性能监控 | 使用 Workers Analytics 替代 |
| 磁盘缓存 | 使用 R2 替代，或依赖 Workers 内存限制 |
| GC / pprof | 不适用 |
| io.net 部署管理 | 降级为纯 API 调用 |

### 10.3 架构差异总结

| 方面 | 当前 Go 架构 | Cloudflare Workers 架构 |
|------|-------------|------------------------|
| **运行模型** | 长运行单进程 | 按请求的无状态隔离环境 |
| **并发** | goroutine + channel | 单线程事件循环 + `async/await` |
| **状态管理** | 进程内内存（map、atomic） | KV / Durable Objects / D1 |
| **数据库** | TCP 长连接 (GORM) | HTTP 连接 (D1 / Hyperdrive) |
| **后台任务** | goroutine 持续运行 | Cron Triggers + Queues |
| **文件系统** | 本地磁盘 | R2 对象存储 |
| **WebSocket** | 全双工持久连接 | Durable Objects WebSocket |
| **执行时间** | 无限制 | 30s CPU / 15min Wall (Paid) |
| **内存** | 服务器 RAM | 128MB (Workers) |
| **部署** | Docker / 二进制 | Wrangler / `wrangler deploy` |

---

## 附录：接口统计

| 路由文件 | 接口数量（约） |
|----------|--------------|
| `api-router.go` | ~170 |
| `relay-router.go` | ~55 |
| `video-router.go` | ~11 |
| `dashboard.go` | 4 |
| `web-router.go` | 1 + NoRoute |
| **总计** | **~241** |

| 组件 | 文件数 |
|------|--------|
| Router | 6 |
| Middleware | 21 |
| Controller | 58 |
| Service | 53 |
| Model | 34 |
| Relay | 187 |
| Setting | 43 |
| Common | 45 |
| DTO | 28 |
| Constant | 13 |
| Types | 9 |
