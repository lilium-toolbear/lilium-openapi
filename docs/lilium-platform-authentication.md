# Lilium 平台认证与路由规范 v1.1

适用对象：接入 Lilium 平台能力的第三方开发者  
文档状态：草案

## 目录

- [1. 文档说明](#overview)
- [2. 统一域名与平台入口](#routing)
- [3. 用户认证（OIDC）](#oidc)
- [4. 服务端认证（OAuth2 client_credentials）](#machine-auth)
- [5. Machine Token 账户绑定](#effective-account)
- [6. API 请求认证](#api-auth)
- [7. 公开 scope](#scopes)
- [8. Token 安全要求](#token-security)
- [9. 平台安全边界](#security)

<a id="overview"></a>
## 1. 文档说明

本文档定义 Lilium 平台级、可复用的认证与入口约定，适用于：

- 第三方站点通过 Lilium 识别用户
- 第三方服务端调用 Lilium API
- 后续新增的非清算类平台 API

本文档不定义具体业务对象、状态机或清算语义。  
清算能力本身见：

- [Lilium 开放清算 API 接入规范 v1](./lilium-open-clearing-api-design.md)

示例第三方接入实现见：

- [lilium-bank-demo](https://github.com/kuma-dzmm/lilium-bank-demo)

<a id="routing"></a>
## 2. 统一域名与平台入口

Lilium 使用统一域名：

- `https://lilium.kuma.homes`

当前平台入口至少包括：

- `https://lilium.kuma.homes/oauth/authorize`
- `https://lilium.kuma.homes/oauth/token`
- `https://lilium.kuma.homes/userinfo`
- `https://lilium.kuma.homes/.well-known/openid-configuration`
- `https://lilium.kuma.homes/.well-known/jwks.json`
- `https://lilium.kuma.homes/openapi.json`

业务文档可以在此基础上定义自己的业务端点，例如清算相关的 `/api/v1/*`。

<a id="oidc"></a>
## 3. 用户认证（OIDC）

第三方站点侧的用户识别使用 `OIDC Authorization Code`。

第三方应接入 Lilium 提供的以下 OIDC 能力：

- authorization endpoint
- token endpoint
- userinfo endpoint
- discovery document
- JWKS endpoint

支持的客户端类型：

- v1 仅支持第三方服务端 OIDC 客户端（confidential client）
- 第三方应在自己的服务端完成授权码交换与 refresh token 存储
- 不支持仅浏览器侧持有长期 refresh token 的 public client 模式

第三方应将 ID Token 中的 `sub` 视为 Lilium `user_id`。`user_id` 类型为 `string`，最大长度 `255`；调用方不得假设固定格式。

OIDC scope：

- `openid`
- `profile`

`id_token` 至少包含：

- `iss`
- `sub`
- `aud`
- `exp`
- `iat`
- `display_name`
- `avatar_url`

其中：

- `sub` 是稳定主标识
- `display_name`、`avatar_url` 是 Lilium 自定义 claims
- `display_name` 与 `avatar_url` 是签发时快照，不保证实时最新

`userinfo` 响应至少可包含：

- `sub`
- `display_name`
- `avatar_url`

使用规则：

- `sub` 必须用于标识用户
- 第三方必须将 `sub` 作为自己的稳定外键
- `display_name`、`avatar_url` 是可变资料，不应作为主键使用
- 如果第三方需要最新资料，应调用 `userinfo endpoint`

OIDC token 生命周期：

- OIDC 登录成功后返回短期 `access_token`
- OIDC 授权码交换成功后同时返回 `id_token` 与 `refresh_token`
- OIDC `refresh_token` 默认有效期为 `180` 天
- OIDC `refresh_token` 用于为同一用户会话续期，不适用于第三方服务端 `client_credentials` token
- OIDC `refresh_token` 是否返回、何时失效、是否被吊销，均以授权服务器响应为准

授权码交换成功时，`/oauth/token` 响应可为：

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "rt_user_...",
  "id_token": "eyJhbG...",
  "token_type": "bearer",
  "expires_in": 900
}
```

使用 OIDC `refresh_token` 续期：

```http
POST /oauth/token HTTP/1.1
Host: lilium.kuma.homes
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&client_id=<client_id>&client_secret=<client_secret>&refresh_token=<refresh_token>
```

规则：

- OIDC `refresh_token` 的默认绝对有效期为 `180` 天
- OIDC `refresh_token` 可采用轮换策略；如果返回了新的 `refresh_token`，第三方应立即替换旧值
- 如果 OIDC `refresh_token` 过期、被吊销或刷新失败，第三方应重新发起用户登录流程
- OIDC `refresh_token` 只能用于 OIDC 用户会话续期，不能用于第三方服务端 `client_credentials` token 刷新

<a id="machine-auth"></a>
## 4. 服务端认证（OAuth2 client_credentials）

第三方服务端调用 Lilium API 时，使用 OAuth2 `client_credentials` 获取短期 access token 和 refresh token。

第三方调用方身份由以下配置确定：

- `client_id`
- `client_secret`

初始获取 token：

```http
POST /oauth/token HTTP/1.1
Host: lilium.kuma.homes
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<client_id>&client_secret=<client_secret>
```

绑定子账户获取 token：

```http
POST /oauth/token HTTP/1.1
Host: lilium.kuma.homes
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<client_id>&client_secret=<client_secret>&effective_account_id=sub_0123456789abcdef0123
```

`effective_account_id` 说明：

- 可选参数
- 类型同 `user_id`
- 值等于目标账户的 `user_id`
- 不传时，machine token 绑定主账户
- 传入时，machine token 绑定该主账户名下的某个子账户
- 签发时完成归属校验，目标账户必须属于该主账户且状态允许绑定
- token 生命周期内不可在业务请求中切换账户

响应示例：

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "rt_...",
  "token_type": "bearer",
  "expires_in": 900
}
```

所有 `*_user_id` claims 与 `sub` 均沿用同一 `user_id` 规则：类型为 `string`，最大长度 `255`，调用方不得假设固定格式。

access token 中至少包含：

- `sub`
  调用方对应的 Lilium `user_id`（主账户）
- `client_id`
  签发此 token 的凭证标识
- `scope`
  授权范围
- `exp`
  过期时间
- `principal_id`
  issued principal 唯一标识，用于审计回溯
- `owner_user_id`
  主账户 `user_id`，等于 `sub`
- `effective_account_user_id`
  token 实际绑定的操作账户；未传 `effective_account_id` 时等于 `sub`，传入子账户时等于该子账户 `user_id`

默认有效期：

- access token：`15` 分钟
- refresh token：`30` 天

refresh token 续期：

```http
POST /oauth/token HTTP/1.1
Host: lilium.kuma.homes
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&client_id=<client_id>&client_secret=<client_secret>&refresh_token=<refresh_token>
```

响应示例：

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "rt_new_...",
  "token_type": "bearer",
  "expires_in": 900
}
```

规则：

- 此处的 refresh token 仅适用于第三方服务端凭证，不适用于 OIDC 用户登录返回的 refresh token
- 每次成功刷新都会返回新的 refresh token，并使旧 refresh token 失效
- refresh token 继承原 principal 的 `effective_account_user_id`，不接受新的 `effective_account_id`
- 如果 refresh token 过期、被吊销或已被轮换，第三方必须重新执行 `client_credentials` 获取新的 token 对
- 想访问另一个账户，重新发起一次 `client_credentials` 并传入对应的 `effective_account_id`
- machine refresh 请求必须携带 `client_id + client_secret`
- 服务端会根据 refresh token 所属类型决定刷新逻辑；OIDC refresh token 与 machine refresh token 不能交叉使用

<a id="effective-account"></a>
## 5. Machine Token 账户绑定

Machine token 是"单账户能力 token"：

- `owner_user_id` 表示该 token 所属主账户
- `effective_account_user_id` 表示该 token 可以实际操作的账户
- 一张 token 只能代表一个 `effective_account_user_id`

这意味着：

- 绑定主账户的 token，只能操作主账户自己的数据
- 绑定子账户的 token，只能操作该子账户的数据
- 想操作另一个账户，必须重新申请绑定该账户的 token

重要语义：

- machine token 的 `sub` 继续保持主账户 `owner_user_id`，用于兼容已有依赖 `sub` 的代码
- **不允许把 `sub` 当作最终操作账户**，必须统一使用 `effective_account_user_id`
- 所有 API 授权判断一律使用 `effective_account_user_id` 作为账户边界

同时持有多组 token：

- 主账户 token
- 子账户 A token
- 子账户 B token

彼此独立刷新，互不影响。

<a id="api-auth"></a>
## 6. API 请求认证

所有 API 请求都必须携带以下请求头：

```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

写接口必须额外携带：

```http
Idempotency-Key: <uuid>
```

POST 请求示例：

```http
POST /api/v1/payment-intents HTTP/1.1
Host: lilium.kuma.homes
Authorization: Bearer eyJhbG...
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

GET 请求示例：

```http
GET /api/v1/payment-intents/pi_001 HTTP/1.1
Host: lilium.kuma.homes
Authorization: Bearer eyJhbG...
```

## 7. 公开 scope

当前公开文档定义的 scope 如下：

- `clearing:basic`
  访问清算相关公开 API：`/api/v1/payment-intents`、`/api/v1/clearing-instructions`、`/api/v1/clearing-batches`
- `wallet:read`
  访问钱包只读 API：`/api/wallet/balance`、`/api/wallet/stats`、`/api/wallet/transactions`、`/api/wallet/wealth_leaderboard`
- `wallet:transfer`
  访问钱包转账 API：`/api/wallet/transfer`
- `chat:bots:read`
  访问 Chat Bot 所有者只读 API：`GET /api/chat/bots*`、`GET /api/chat/commands/directory`（`chat.kuma.homes`，见 [Chat Bot API](./lilium-chat-bot-api-design.md) §3.2）
- `chat:bots:manage`
  访问 Chat Bot 所有者写 API：`POST` / `PATCH` / `DELETE /api/chat/bots*`（签发 Chat Bot Token 等）

规则：

- access token 的 `scope` claim 由凭证配置决定
- v1 不支持在 token 请求中动态下调或上调 scope；服务端直接按该凭证配置签发
- 如果 token 缺少调用某个 API 所需的 scope，服务端返回 `INVALID_SCOPE`
- `wallet:transfer` 与 `wallet:read` 独立授权，不互相包含

<a id="token-security"></a>
## 8. Token 安全要求

- access token 应保持短有效期；v1 默认 `15` 分钟
- refresh token 应仅存放在服务端安全存储中，不应暴露到浏览器
- 第三方不应在浏览器中暴露 access token
- 第三方应在 access token 过期前使用 refresh token 续期
- refresh token 应视为长期凭证，泄露后应立即吊销并重新签发
- Lilium 保留在检测到异常时吊销 token 的权利

<a id="security"></a>
## 9. 平台安全边界

第三方必须满足以下要求：

- 所有 API 调用都使用 HTTPS
- 妥善保护 `client_secret`
- 不在浏览器中暴露 `client_secret` 或 access token
- 不把 `user_id` 当作直接授权凭证
- 对所有 Webhook 做签名校验

具体业务能力仍可能附加自己的安全边界。  
例如清算能力额外要求不得绕过 Hosted Checkout 直接对用户做冻结或扣款。

## 修订历史

- `2026-04-09`: 初始平台认证与路由规范成稿。
- `2026-04-09`: 将平台级认证能力从清算文档中拆出，作为共享前置能力文档。
- `2026-04-12`: v1.1 — 新增 `effective_account_id` 参数与子账户绑定语义；新增 `principal_id`、`owner_user_id`、`effective_account_user_id` JWT claims；新增 `wallet:read`、`wallet:transfer` scope；refresh token 继承 effective account。
- `2026-04-12`: 统一 `user_id` 契约：`sub`、`owner_user_id`、`effective_account_user_id` 与 `effective_account_id` 均沿用同一 `user_id` 规则（`string`，最大长度 `255`，不假设固定格式）。
