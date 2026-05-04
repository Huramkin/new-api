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
- [11. 前端页面迁移实现](#11-前端页面迁移实现)
  - [11.1 前端项目结构](#111-前端项目结构)
  - [11.2 前端技术栈与依赖](#112-前端技术栈与依赖)
  - [11.3 前端路由与页面清单](#113-前端路由与页面清单)
  - [11.4 布局组件体系](#114-布局组件体系)
  - [11.5 前端与后端的交互方式](#115-前端与后端的交互方式)
  - [11.6 状态管理](#116-状态管理)
  - [11.7 国际化实现](#117-国际化实现)
  - [11.8 样式体系](#118-样式体系)
  - [11.9 当前构建与部署方式](#119-当前构建与部署方式)
  - [11.10 Cloudflare Workers/Pages 前端部署方案](#1110-cloudflare-workerspages-前端部署方案)
  - [11.11 前端迁移需要调整的部分](#1111-前端迁移需要调整的部分)
  - [11.12 前端迁移中无法实现的功能](#1112-前端迁移中无法实现的功能)
  - [11.13 前端迁移步骤](#1113-前端迁移步骤)

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

---

## 11. 前端页面迁移实现

### 11.1 前端项目结构

前端项目位于 `web/` 目录，是一个独立的 React SPA 应用。

```
web/
├── package.json                     # 依赖管理与脚本
├── vite.config.js                   # Vite 构建配置
├── tailwind.config.js               # Tailwind CSS 配置
├── postcss.config.js                # PostCSS 配置
├── index.html                       # HTML 入口
├── public/                          # 静态资源（logo、favicon、支付图标等）
└── src/
    ├── index.jsx                    # React 入口（ReactDOM.createRoot）
    ├── index.css                    # 全局样式（~970 行）
    ├── App.jsx                      # 路由定义
    │
    ├── pages/                       # 页面级组件（与路由 1:1 对应）
    │   ├── About/                   # 关于页面
    │   ├── Channel/                 # 渠道管理（管理员）
    │   ├── Chat/                    # 聊天界面（iframe 嵌入）
    │   ├── Chat2Link/               # 聊天直链跳转
    │   ├── Dashboard/               # 仪表板
    │   ├── Forbidden/               # 403 页面
    │   ├── Home/                    # 首页
    │   ├── Log/                     # 使用日志
    │   ├── Midjourney/              # Midjourney 日志
    │   ├── Model/                   # 模型管理（管理员）
    │   ├── ModelDeployment/         # 模型部署（管理员）
    │   ├── NotFound/                # 404 页面
    │   ├── Playground/              # AI Playground
    │   ├── Pricing/                 # 定价页面
    │   ├── PrivacyPolicy/           # 隐私政策
    │   ├── Redemption/              # 兑换码管理（管理员）
    │   ├── Setting/                 # 系统设置（管理员，含12个子标签页）
    │   │   ├── Chat/                # 聊天设置
    │   │   ├── Dashboard/           # 仪表板设置
    │   │   ├── Drawing/             # 绘图设置
    │   │   ├── Model/               # 模型设置
    │   │   ├── Operation/           # 运营设置
    │   │   ├── Payment/             # 支付设置
    │   │   ├── Performance/         # 性能设置
    │   │   ├── RateLimit/           # 限流设置
    │   │   └── Ratio/               # 比例设置
    │   ├── Setup/                   # 系统初始化向导
    │   ├── Subscription/            # 订阅管理（管理员）
    │   ├── Task/                    # 任务日志
    │   ├── Token/                   # API 令牌管理
    │   ├── TopUp/                   # 钱包/充值
    │   ├── User/                    # 用户管理（管理员）
    │   └── UserAgreement/           # 用户协议
    │
    ├── components/
    │   ├── auth/                    # 登录、注册、OAuth、密码重置表单
    │   ├── common/                  # 共享 UI 组件
    │   │   ├── ui/                  # CardPro, JSONEditor, Loading 等
    │   │   ├── markdown/            # Markdown 渲染器
    │   │   ├── DocumentRenderer/    # 文档渲染（Markdown/HTML/iframe）
    │   │   ├── modals/              # 风险确认、安全验证弹窗
    │   │   └── logo/                # 自定义 OAuth 图标
    │   ├── dashboard/               # 仪表板子组件（统计卡片、图表、公告等）
    │   ├── layout/                  # 布局框架（PageLayout、HeaderBar、SiderBar、Footer）
    │   │   └── headerbar/           # 头部导航（Logo、语言选择、主题切换、用户区）
    │   ├── playground/              # Playground 子组件（17 个文件）
    │   ├── settings/                # 管理员设置面板
    │   │   └── personal/            # 个人设置（2FA、Passkey、OAuth 绑定等）
    │   ├── setup/                   # 初始化向导步骤组件
    │   ├── table/                   # 数据表格（channels、tokens、users 等 11 个实体）
    │   │   └── (每个实体含 modals/)
    │   ├── topup/                   # 充值组件
    │   └── model-deployments/       # 模型部署组件
    │
    ├── constants/                   # 应用常量
    ├── context/                     # React Context（Status、Theme、User）
    ├── contexts/                    # Playground Context
    ├── helpers/                     # 工具模块（API、认证、格式化等）
    ├── hooks/                       # 自定义 Hooks（数据获取与逻辑）
    │   ├── channels/                # 渠道数据 Hook
    │   ├── chat/                    # 聊天 Hook
    │   ├── common/                  # 通用 Hook（响应式、导航、通知等）
    │   ├── dashboard/               # 仪表板数据 Hook
    │   ├── playground/              # Playground Hook
    │   └── (tokens, users, logs 等)
    ├── i18n/                        # 国际化配置与语言包
    │   └── locales/                 # zh-CN, zh-TW, en, fr, ru, ja, vi
    └── services/                    # 服务层（安全验证）
```

### 11.2 前端技术栈与依赖

#### 核心依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `react` / `react-dom` | ^18.2.0 | React 框架 |
| `react-router-dom` | ^6.3.0 | 客户端路由 |
| `@douyinfe/semi-ui` | ^2.69.1 | **Semi Design UI 组件库**（主要 UI 框架） |
| `@douyinfe/semi-icons` | ^2.63.1 | Semi Design 图标 |
| `axios` | 1.13.5 | HTTP 客户端 |
| `i18next` + `react-i18next` | ^23.16.8 / ^13.0.0 | 国际化 |
| `i18next-browser-languagedetector` | ^7.2.0 | 语言自动检测 |
| `sse.js` | ^2.6.0 | Server-Sent Events 客户端（Playground 流式传输） |
| `react-markdown` | ^10.1.0 | Markdown 渲染 |
| `marked` | ^4.1.1 | Markdown 解析 |
| `katex` / `rehype-katex` / `remark-math` | — | LaTeX 数学公式渲染 |
| `mermaid` | ^11.6.0 | 图表渲染 |
| `@visactor/react-vchart` | ~1.8.8 | 数据图表（仪表板） |
| `@visactor/vchart-semi-theme` | ~1.8.8 | VChart Semi 主题 |
| `@lobehub/icons` | ^2.0.0 | AI 提供商品牌图标（首页展示） |
| `dayjs` | ^1.11.11 | 日期处理 |
| `lucide-react` | ^0.511.0 | 图标库 |
| `react-dropzone` | ^14.2.3 | 文件上传 |
| `react-turnstile` | ^1.0.5 | Cloudflare Turnstile CAPTCHA |
| `react-telegram-login` | ^1.1.2 | Telegram 登录 |
| `qrcode.react` | ^4.2.0 | 二维码生成 |
| `clsx` | ^2.1.1 | 类名工具 |
| `use-debounce` | ^10.0.4 | 防抖 Hook |

#### 构建工具

| 包名 | 用途 |
|------|------|
| `vite` ^5.2.0 | 构建工具 |
| `@vitejs/plugin-react` | React Vite 插件 |
| `@douyinfe/vite-plugin-semi` | Semi Design Vite 集成 |
| `tailwindcss` ^3 | 原子化 CSS |
| `postcss` / `autoprefixer` | CSS 处理 |
| `prettier` / `eslint` | 代码格式化/检查 |

#### Cloudflare Workers/Pages 兼容性

**所有前端依赖均为纯浏览器端库，不依赖 Node.js 服务端运行时**，因此前端代码本身 100% 兼容 Cloudflare 部署环境。前端构建产物（HTML/JS/CSS/静态资源）是纯静态文件，可以直接部署到任何静态托管服务。

### 11.3 前端路由与页面清单

所有路由定义在 `web/src/App.jsx` 中：

#### 公开页面

| 路由 | 页面组件 | 说明 |
|------|----------|------|
| `/` | `Home` (lazy) | 首页：英雄横幅、API 地址展示、AI 提供商图标、公告弹窗 |
| `/setup` | `Setup` | 系统初始化向导（数据库、管理员账户、使用模式） |
| `/login` | `LoginForm` | 登录：用户名/密码、OAuth、Passkey、Turnstile、2FA |
| `/register` | `RegisterForm` | 注册：支持 OAuth、邮箱验证、邀请码 |
| `/reset` | `PasswordResetForm` | 请求密码重置 |
| `/user/reset` | `PasswordResetConfirm` | 密码重置确认（含 token） |
| `/oauth/github` | `OAuth2Callback` | GitHub OAuth 回调 |
| `/oauth/discord` | `OAuth2Callback` | Discord OAuth 回调 |
| `/oauth/oidc` | `OAuth2Callback` | OIDC OAuth 回调 |
| `/oauth/linuxdo` | `OAuth2Callback` | LinuxDO OAuth 回调 |
| `/oauth/:provider` | `DynamicOAuth2Callback` | 动态自定义 OAuth 回调 |
| `/forbidden` | `Forbidden` | 403 禁止访问 |
| `/about` | `About` (lazy) | 关于页面 |
| `/pricing` | `Pricing` | 定价页面（可配置是否需要登录） |
| `/user-agreement` | `UserAgreement` (lazy) | 用户协议 |
| `/privacy-policy` | `PrivacyPolicy` (lazy) | 隐私政策 |
| `/console/chat/:id?` | `Chat` | 聊天界面（iframe 嵌入外部聊天 UI） |
| `*` | `NotFound` | 404 页面 |

#### 需要登录的页面（`PrivateRoute`）

| 路由 | 页面组件 | 说明 |
|------|----------|------|
| `/chat2link` | `Chat2Link` | 聊天直链跳转 |
| `/console` | `Dashboard` (lazy) | **主仪表板**：统计卡片、图表、API 信息、公告、FAQ、在线状态监控 |
| `/console/playground` | `Playground` | **AI Playground**：模型选择、参数控制、流式 SSE 响应、代码查看、调试面板 |
| `/console/token` | `Token` | API 令牌管理（CRUD、复制密钥） |
| `/console/topup` | `TopUp` | 钱包管理：充值卡、订阅计划、邀请/推荐 |
| `/console/log` | `Log` | 使用日志表格（筛选、列选择） |
| `/console/midjourney` | `Midjourney` | Midjourney 绘图日志 |
| `/console/task` | `Task` | 任务日志（异步任务如音频） |
| `/console/personal` | `PersonalSetting` | 个人设置：资料、2FA、Passkey、OAuth 绑定、密码、通知、签到日历 |

#### 需要管理员权限的页面（`AdminRoute`，`role >= 10`）

| 路由 | 页面组件 | 说明 |
|------|----------|------|
| `/console/models` | `ModelPage` | 模型管理：模型/供应商/预填充组 CRUD、同步向导 |
| `/console/deployment` | `ModelDeploymentPage` | 模型部署管理（io.net 集成） |
| `/console/subscription` | `Subscription` | 订阅计划管理 |
| `/console/channel` | `Channel` | **渠道/提供商管理**：CRUD、测试、标签、批量操作、上游更新 |
| `/console/redemption` | `Redemption` | 兑换码管理 |
| `/console/user` | `User` | 用户管理：CRUD、角色、启用/禁用、绑定管理 |
| `/console/setting` | `Setting` | **系统设置**（仅 root 用户，含 12 个标签页） |

#### 系统设置标签页详情 (`/console/setting`)

| 标签 | 组件 | 子设置项 |
|------|------|---------|
| 运营 | `OperationSetting` | 通用设置、日志、监控、信用额度、签到、敏感词、渠道亲和、导航模块、侧栏模块 |
| 仪表板 | `DashboardSetting` | 数据面板、API 信息、公告、FAQ、Uptime Kuma |
| 聊天 | `ChatsSetting` | 聊天配置 |
| 绘图 | `DrawingSetting` | 绘图配置 |
| 支付 | `PaymentSetting` | 通用支付、Epay、Stripe、Creem、Waffo 网关 |
| 比例 | `RatioSetting` | 组比例、模型比例、上游比例同步、模型设置可视化编辑 |
| 限流 | `RateLimitSetting` | 请求限流配置 |
| 模型 | `ModelSetting` | 全局模型、Claude、Gemini、Grok 模型设置 |
| 模型部署 | `ModelDeploymentSetting` | io.net 部署设置 |
| 性能 | `PerformanceSetting` | 性能配置、磁盘缓存、GC、日志文件 |
| 系统 | `SystemSetting` | SMTP、OAuth（GitHub/Discord/OIDC/LinuxDO/微信/Telegram/自定义）、Turnstile、系统设置 |
| 其他 | `OtherSetting` | 首页内容、关于页面、公告、HTTP 状态码规则 |

### 11.4 布局组件体系

前端使用统一布局框架，所有页面共享同一套布局组件：

```
┌─────────────────────────────────────────────────┐
│                   HeaderBar                      │
│  ┌──────┐  ┌──────────────────────┐  ┌────────┐ │
│  │ Logo │  │    Navigation        │  │UserArea│ │
│  └──────┘  └──────────────────────┘  └────────┘ │
├──────────┬──────────────────────────────────────┤
│          │                                       │
│ SiderBar │           Content                     │
│ (仅       │         (App 路由)                   │
│ /console │                                       │
│ 下显示)   │                                       │
│          │                                       │
├──────────┴──────────────────────────────────────┤
│                   FooterBar                      │
└─────────────────────────────────────────────────┘
```

| 组件 | 文件 | 职责 |
|------|------|------|
| `PageLayout` | `components/layout/PageLayout.jsx` | 顶层布局，包裹整个应用。使用 Semi `Layout`。加载用户/状态数据。 |
| `HeaderBar` | `components/layout/headerbar/index.jsx` | 固定顶部导航栏：Logo、导航链接、主题切换、语言选择、通知铃铛、用户下拉菜单 |
| `SiderBar` | `components/layout/SiderBar.jsx` | 左侧边栏（仅 `/console/*` 路由显示，可折叠）：聊天、控制台、个人、管理员四个分区 |
| `FooterBar` | `components/layout/Footer.jsx` | 页脚：版权、链接、支持自定义 HTML 页脚 |
| `SetupCheck` | `components/layout/SetupCheck.js` | HOC：系统未初始化时重定向到 `/setup` |
| `NoticeModal` | `components/layout/NoticeModal.jsx` | 系统公告弹窗 |
| `SemiLocaleWrapper` | `index.jsx` | Semi UI 语言包同步（i18n → Semi locale） |

### 11.5 前端与后端的交互方式

#### HTTP 客户端 (`helpers/api.js`)

```javascript
export let API = axios.create({
  baseURL: import.meta.env.VITE_REACT_APP_SERVER_URL || '',
  headers: {
    'New-API-User': getUserIdFromLocalStorage(),
    'Cache-Control': 'no-store',
  },
});
```

**关键特性：**
- **Base URL**：默认为空字符串（同源），即前端与后端在同一域名下部署
- **可配置**：通过 `VITE_REACT_APP_SERVER_URL` 环境变量指定不同的 API 服务器
- **GET 请求去重**：并发相同 GET 请求自动合并为单个请求
- **全局错误拦截**：401 自动跳转登录、429 显示限流提示、500 显示服务器错误
- **可跳过错误处理**：请求配置 `{ skipErrorHandler: true }` 可绕过全局拦截

#### 认证机制

| 机制 | 说明 |
|------|------|
| **Session Cookie** | 主要认证方式。Go 后端使用 Gin Sessions，设置 HttpOnly Cookie（30 天有效，SameSite=Strict） |
| **`New-API-User` Header** | 每个请求携带用户 ID，辅助后端识别用户 |
| **Bearer Token** | 仅在特定场景使用（如 Ollama SSE 流、Chat iframe），非全局 |
| **localStorage `user`** | 存储完整用户对象（id、username、role、token、group、setting 等） |

#### 认证流程

```
登录请求 POST /api/user/login
    │
    ├── 返回 { require_2fa: true } → 显示 2FA 验证 → POST /api/user/login/2fa
    │
    └── 成功返回用户数据
         │
         ├── localStorage.setItem('user', JSON.stringify(data))
         ├── UserContext.dispatch({ type: 'login', payload: data })
         ├── updateAPI()  // 重建 axios 实例
         └── 跳转到 /console
```

#### 路由守卫

| 守卫 | 逻辑 |
|------|------|
| `PrivateRoute` | 检查 `localStorage.user` 是否存在，否则跳转 `/login` |
| `AdminRoute` | 同上 + 检查 `user.role >= 10`，否则跳转 `/forbidden` |
| `AuthRedirect` | 已登录用户访问 `/login`、`/register` 时自动跳转 `/console` |

#### 流式传输（SSE）

前端有两处使用 SSE：

1. **Playground 聊天流** (`hooks/playground/useApiRequest.jsx`)：
   - 使用 `sse.js` 库连接 `/pg/chat/completions`
   - 发送 `Content-Type: application/json` 和 `New-Api-User` 头
   - 处理 `data: [DONE]` 结束信号

2. **Ollama 模型拉取** (`components/table/channels/modals/OllamaModelModal.jsx`)：
   - 使用原生 `fetch()` + `ReadableStream` 连接 `/api/channel/ollama/pull/stream`
   - 发送 `Authorization: Bearer` 和 `New-API-User` 头
   - 手动解析 SSE `data:` 行

#### 前端调用的 API 端点汇总（~150+ 个）

| 分类 | 主要端点 | 调用位置 |
|------|---------|---------|
| **系统状态** | `GET /api/status`, `GET /api/notice` | `PageLayout`, `Home` |
| **认证** | `POST /api/user/login`, `GET /api/user/logout`, `POST /api/user/register` | 认证组件 |
| **OAuth** | `GET /api/oauth/state`, `GET /api/oauth/:provider` | OAuth 组件 |
| **用户自服务** | `GET/PUT /api/user/self`, `GET /api/user/models` | 个人设置、TopUp |
| **2FA/Passkey** | `GET/POST /api/user/2fa/*`, `POST /api/user/passkey/*` | 安全设置 |
| **管理员-用户** | `GET /api/user/`, `POST /api/user/manage` | 用户管理 |
| **渠道** | `GET/POST/PUT/DELETE /api/channel/*` | 渠道管理 |
| **令牌** | `GET/POST/PUT/DELETE /api/token/*` | 令牌管理 |
| **日志** | `GET /api/log/*`, `GET /api/mj/*`, `GET /api/task/*` | 日志页面 |
| **充值/支付** | `POST /api/user/topup`, `POST /api/user/stripe/pay` | 充值页面 |
| **订阅** | `GET/POST /api/subscription/*` | 订阅管理 |
| **设置** | `GET/PUT /api/option/` | 系统设置 |
| **模型** | `GET/POST /api/models/*`, `GET/POST /api/vendors/*` | 模型管理 |
| **部署** | `GET/POST /api/deployments/*` | 部署管理 |
| **Playground** | `POST /pg/chat/completions` | Playground |
| **定价** | `GET /api/pricing`, `GET /api/ratio_config` | 定价页面 |
| **公开页面** | `GET /api/about`, `GET /api/home_page_content` | About、Home |
| **性能** | `GET /api/performance/*` | 性能设置 |
| **自定义OAuth** | `GET/POST /api/custom-oauth-provider/*` | OAuth 设置 |

### 11.6 状态管理

使用 **React Context + `useReducer`**，无外部状态管理库。

| Context | 位置 | 状态结构 | 说明 |
|---------|------|---------|------|
| `UserContext` | `context/User/` | `{ user: Object \| undefined }` | 用户登录/登出状态 |
| `StatusContext` | `context/Status/` | `{ status: Object \| undefined }` | 系统配置（从 `/api/status` 加载） |
| `ThemeContext` | `context/Theme/` | `{ theme: 'light'\|'dark'\|'auto' }` | 主题模式 |
| `PlaygroundContext` | `contexts/PlaygroundContext.jsx` | 图片粘贴处理 | Playground 局部 Context |

**数据持久化**：大量使用 `localStorage` 缓存：
- `user` — 用户信息
- `status` — 系统配置
- `system_name`, `logo`, `footer_html` — 提取的配置值
- `i18nextLng` — 语言偏好
- `theme-mode` — 主题模式
- `channel_models` — 缓存的模型列表

**数据获取**：通过 `hooks/` 目录下的自定义 Hooks 实现（如 `useChannelsData`, `useDashboardData`），调用 axios API 实例管理组件级状态。

### 11.7 国际化实现

| 配置项 | 值 |
|--------|---|
| 库 | `i18next` + `react-i18next` + `i18next-browser-languagedetector` |
| 支持语言 | `zh-CN`（简体中文，回退语言）、`zh-TW`（繁体中文）、`en`、`fr`、`ru`、`ja`、`vi` |
| 翻译文件 | `web/src/i18n/locales/{lang}.json` — 扁平 JSON，key 为中文源字符串 |
| 用法 | `useTranslation()` Hook，调用 `t('中文key')` |
| Semi UI 同步 | `SemiLocaleWrapper` 将 `i18n.language` 映射到 Semi 的 locale 对象 |
| 语言检测 | 自动检测浏览器语言，支持用户手动切换 |
| 规范化 | `normalizeLanguage()` 将变体映射到标准语言代码 |

### 11.8 样式体系

前端采用**三层混合样式**方案：

| 层 | 技术 | 说明 |
|---|------|------|
| 1 | **Tailwind CSS** | 原子化工具类，颜色映射到 Semi Design CSS 变量 |
| 2 | **Semi Design UI CSS** | 组件级主题，通过 `@douyinfe/vite-plugin-semi` 集成 |
| 3 | **自定义 CSS** (`index.css`) | ~970 行自定义样式：布局、动画、深色模式、响应式 |

**CSS Layer 顺序**：`tailwind-base → semi → tailwind-components → tailwind-utils`，确保 Tailwind 工具类可覆盖 Semi 样式。

**深色模式**：通过 `body[theme-mode="dark"]` 属性 + `document.documentElement.classList.add('dark')` 激活，Semi Design 和 Tailwind 同步切换。

**无 CSS Modules / styled-components / CSS-in-JS。**

### 11.9 当前构建与部署方式

#### 构建流程

```bash
cd web/
bun install          # 安装依赖
bun run build        # Vite 构建 → web/dist/
cd ..
go build             # embed.FS 将 web/dist/ 嵌入 Go 二进制
```

#### Go 嵌入机制

```go
// main.go
//go:embed web/dist
var buildFS embed.FS

//go:embed web/dist/index.html
var indexPage []byte
```

- Go 使用 `embed.FS` 将整个 `web/dist/` 目录编译进二进制文件
- 启动时，`indexPage` 会被注入 Umami/Google Analytics 脚本
- `router/web-router.go` 通过 `gin-contrib/static` 提供静态文件服务
- 未匹配路由返回 `index.html`（SPA 回退）

#### 前端环境变量

仅一个 Vite 环境变量：

| 变量 | 用途 | 默认值 |
|------|------|--------|
| `VITE_REACT_APP_SERVER_URL` | API 服务器 URL | `''`（同源） |

#### 分离部署支持

Go 后端支持 `FRONTEND_BASE_URL` 环境变量。设置后，非 API 路由会重定向到该外部前端 URL，支持前后端分离部署。

### 11.10 Cloudflare Workers/Pages 前端部署方案

#### 推荐方案：Cloudflare Pages

**Cloudflare Pages 是部署前端 SPA 的最佳选择**，原因如下：

| 特性 | 说明 |
|------|------|
| **原生 SPA 支持** | 自动处理 `_redirects` 或 `_headers` 文件，支持 SPA 回退到 `index.html` |
| **自动构建** | 连接 Git 仓库后自动触发构建部署 |
| **全球 CDN** | 静态资源通过 Cloudflare CDN 分发 |
| **自定义域名** | 支持绑定自定义域名 |
| **Preview 环境** | 每个 PR 自动生成预览 URL |
| **无限带宽** | 免费套餐即可获得无限带宽 |
| **与 Workers 同域** | 可以通过 Workers Routes 将 API 和前端部署在同一域名下 |

#### 部署配置

**`wrangler.toml`（Pages 配置示例）：**
```toml
name = "new-api-frontend"
compatibility_date = "2024-01-01"

[site]
bucket = "./web/dist"
```

**Pages 构建设置：**
```
Build command: cd web && bun install && bun run build
Build output directory: web/dist
Root directory: /
Environment variables:
  VITE_REACT_APP_SERVER_URL = https://api.your-domain.com
```

**SPA 路由回退 (`web/dist/_redirects`)：**
```
/*  /index.html  200
```

#### 同域部署架构

```
                    ┌──────────────────┐
                    │  Cloudflare CDN  │
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              │                              │
    ┌─────────┴─────────┐      ┌────────────┴────────────┐
    │  Cloudflare Pages │      │   Cloudflare Workers     │
    │  (前端静态资源)    │      │   (API 后端)             │
    │  your-domain.com  │      │   your-domain.com/api/*  │
    │  your-domain.com/  │     │   your-domain.com/v1/*   │
    │  your-domain.com/  │     │   your-domain.com/pg/*   │
    │  console/*        │      │   your-domain.com/mj/*   │
    └───────────────────┘      └─────────────────────────┘
```

**Workers Routes 配置**（将 API 路由指向 Worker）：
```toml
# Workers wrangler.toml
routes = [
  { pattern = "your-domain.com/api/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/v1/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/v1beta/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/pg/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/mj/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/suno/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/kling/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/jimeng/*", zone_name = "your-domain.com" },
  { pattern = "your-domain.com/dashboard/*", zone_name = "your-domain.com" },
]
```

这样前端和 API 在同一域名下，**无需 CORS 配置**，前端可以使用相对路径调用 API（不需要设置 `VITE_REACT_APP_SERVER_URL`），Session Cookie 也能正常工作。

### 11.11 前端迁移需要调整的部分

#### 11.11.1 环境变量配置

**当前**：`VITE_REACT_APP_SERVER_URL` 默认同源

**迁移方案**：

| 部署方式 | 配置 | 说明 |
|---------|------|------|
| 同域部署 | 不设置（默认同源） | Pages 处理前端，Workers Routes 处理 API |
| 跨域部署 | 设置 `VITE_REACT_APP_SERVER_URL` | 需要在 Workers 端配置 CORS |

#### 11.11.2 认证机制调整

**当前**：Gin Sessions（Cookie-based，服务端存储）

**迁移方案**：

| 方案 | 说明 | 复杂度 |
|------|------|--------|
| **A. JWT 无状态 Token** | 后端签发 JWT，前端存 localStorage，每次请求 Bearer Token | 🟢 低 — 前端改动最小 |
| **B. 签名 Cookie + KV** | Workers 签发签名 Cookie，Session 数据存 KV | 🟡 中 — 需要自实现 Session |
| **C. 同域 Cookie + Workers Session** | Workers 使用加密 Cookie 存储 Session | 🟡 中 |

**推荐方案 A（JWT 无状态 Token）**：

前端改动：
1. 登录成功后将 JWT 存入 `localStorage`（当前已经在做）
2. 在 axios 请求拦截器中全局添加 `Authorization: Bearer <token>` 头
3. 移除对 Cookie Session 的依赖

```javascript
// helpers/api.js 修改
API.interceptors.request.use((config) => {
  const user = JSON.parse(localStorage.getItem('user'));
  if (user?.token) {
    config.headers.Authorization = `Bearer ${user.token}`;
  }
  return config;
});
```

#### 11.11.3 SSE 流式传输

**当前**：使用 `sse.js` 库连接 `/pg/chat/completions`

**迁移方案**：**无需修改前端代码**。Cloudflare Workers 完全支持 SSE 响应（`ReadableStream` + `TransformStream`），前端的 `sse.js` 客户端不需要任何更改。

#### 11.11.4 Turnstile CAPTCHA

**当前**：使用 `react-turnstile` 组件

**迁移方案**：**无需修改前端代码**。Cloudflare Turnstile 在 Pages/Workers 环境下工作更好（同一 Cloudflare 生态）。后端验证仍通过 `challenges.cloudflare.com/turnstile/v0/siteverify` API。

#### 11.11.5 Vite 开发代理

**当前**：Vite `server.proxy` 将 `/api`、`/mj`、`/pg` 代理到 `http://localhost:3000`

**迁移方案**：开发时需要将代理目标改为 Workers 的本地开发地址：

```javascript
// vite.config.js 修改
server: {
  proxy: {
    '/api': { target: 'http://localhost:8787', changeOrigin: true },
    '/mj':  { target: 'http://localhost:8787', changeOrigin: true },
    '/pg':  { target: 'http://localhost:8787', changeOrigin: true },
    '/v1':  { target: 'http://localhost:8787', changeOrigin: true },
  },
},
```

（`8787` 是 `wrangler dev` 的默认端口）

#### 11.11.6 构建输出处理

**当前**：`go:embed` 将 `web/dist/` 嵌入 Go 二进制

**迁移方案**：
- 前端独立构建，产物部署到 Cloudflare Pages
- 不再需要 `go:embed`，前后端完全分离
- CI/CD 中分别构建前端和 Workers 后端

#### 11.11.7 Analytics 脚本注入

**当前**：Go 启动时将 Umami/Google Analytics 脚本注入 `index.html`

```go
// main.go 中
indexPage = []byte(strings.Replace(string(indexPage),
    "</head>", umamiScript+"</head>", 1))
```

**迁移方案**：

| 方案 | 说明 |
|------|------|
| **A. 构建时注入** | 在 Vite 构建配置中通过 `vite-plugin-html` 注入脚本 |
| **B. Pages Functions** | 使用 Cloudflare Pages Functions 在 `_middleware.ts` 中动态注入 |
| **C. Zaraz** | 使用 Cloudflare Zaraz 管理第三方脚本（推荐） |

推荐方案 C（Zaraz）— Cloudflare 原生的第三方脚本管理工具，零代码配置。

### 11.12 前端迁移中无法实现的功能

#### ❌ 1. Chat iframe 外部聊天链接（部分受限）

**当前**：`/console/chat/:id?` 页面通过 iframe 嵌入外部聊天 UI（如 ChatGPT Next Web），使用用户的 API 密钥

**问题**：
- iframe 嵌入的外部聊天 UI 本身不受控，取决于第三方服务是否允许被 iframe 嵌入
- 跨域 iframe 可能受 CSP（Content Security Policy）限制

**影响**：功能本身可迁移，但依赖第三方服务的可嵌入性，非 Cloudflare 限制

#### ❌ 2. Ollama 模型拉取流（后端限制）

**当前**：`OllamaModelModal` 通过 SSE 流式显示 Ollama 模型拉取进度

**问题**：Ollama 是本地部署的模型运行时，Workers 无法连接本地 Ollama 实例。此功能在后端层面已不可行。

**前端影响**：相关 UI 组件（`OllamaModelModal`）需要隐藏或移除

#### ❌ 3. 性能监控面板（后端限制）

**当前**：`/console/setting` → 性能标签页显示 CPU、内存、磁盘使用率，支持强制 GC、清除磁盘缓存

**问题**：Workers 无法提供 OS 级系统指标

**前端影响**：性能设置页面需要简化或改为显示 Workers Analytics 数据

#### ❌ 4. io.net 模型部署管理（部分受限）

**当前**：`/console/deployment` 完整的 GPU 部署管理界面

**问题**：部署创建、日志查看等功能依赖长时间运行的后端操作

**前端影响**：基本 CRUD 可保留，实时日志和容器详情可能需要降级

### 11.13 前端迁移步骤

#### 第一步：前端独立部署到 Cloudflare Pages

1. **创建 Cloudflare Pages 项目**
   ```bash
   cd web
   npx wrangler pages project create new-api-frontend
   ```

2. **添加 SPA 回退配置**
   ```bash
   # web/public/_redirects
   /*  /index.html  200
   ```

3. **配置环境变量**
   - 开发环境：不设置 `VITE_REACT_APP_SERVER_URL`（使用 Vite 代理）
   - 生产环境（跨域）：设置为 Workers API 的 URL
   - 生产环境（同域）：不设置

4. **构建并部署**
   ```bash
   cd web
   bun install
   bun run build
   npx wrangler pages deploy dist --project-name new-api-frontend
   ```

5. **配置自定义域名**（可选）
   在 Cloudflare Dashboard → Pages → 自定义域 中添加

#### 第二步：调整前端认证方式

1. 添加 axios 请求拦截器，全局携带 Bearer Token
2. 移除对服务端 Session Cookie 的依赖
3. 保留 `localStorage` 用户缓存机制

#### 第三步：配置同域路由

1. 在 Cloudflare Dashboard 中为同一域名配置 Workers Routes
2. API 路径（`/api/*`、`/v1/*` 等）路由到 Workers
3. 其他路径（`/`、`/console/*` 等）由 Pages 处理

#### 第四步：移除不兼容的 UI 组件

1. 隐藏 Ollama 相关 UI（`OllamaModelModal`、Ollama 版本检查）
2. 简化性能监控面板
3. 评估 io.net 部署管理的可行性

#### 第五步：CI/CD 配置

```yaml
# GitHub Actions 示例
name: Deploy Frontend
on:
  push:
    branches: [main]
    paths: ['web/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: cd web && bun install && bun run build
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          command: pages deploy web/dist --project-name new-api-frontend
```

#### 前端迁移工作量评估

| 任务 | 复杂度 | 说明 |
|------|--------|------|
| Pages 部署配置 | 🟢 低 | 标准 Vite SPA 部署 |
| SPA 回退配置 | 🟢 低 | 添加 `_redirects` 文件 |
| 环境变量调整 | 🟢 低 | 一个变量 |
| 认证方式调整 | 🟡 中 | 添加请求拦截器，需配合后端 JWT 实现 |
| 同域路由配置 | 🟡 中 | Workers Routes + Pages 协调 |
| Analytics 迁移 | 🟢 低 | 使用 Zaraz 或构建时注入 |
| 移除不兼容 UI | 🟢 低 | 条件渲染或移除少量组件 |
| Vite 开发代理调整 | 🟢 低 | 改代理目标端口 |
| CI/CD 配置 | 🟢 低 | 标准 GitHub Actions |
| **总体** | 🟢 **低** | **前端迁移是整个项目中最简单的部分** |

> **结论**：前端是一个标准的 React SPA，所有代码运行在浏览器端，不依赖服务端运行时。迁移到 Cloudflare Pages 几乎不需要修改前端代码本身，主要工作在于部署配置和认证方式的调整。前端迁移应作为整体迁移的**第一步**，因为它风险最低、效果最直接。
