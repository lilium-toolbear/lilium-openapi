# Lilium API 错误模型规范 v1

适用对象：接入 Lilium 的第三方开发者  
文档状态：草案

## 目录

- [1. 文档说明](#overview)
- [2. 统一错误响应结构](#envelope)
- [3. 通用规则](#rules)
- [4. 通用错误码](#common-codes)
- [5. 钱包能力错误码](#wallet-codes)
- [6. 清算能力错误码](#clearing-codes)

<a id="overview"></a>
## 1. 文档说明

本规范定义 Lilium 对外 API 统一使用的错误响应结构与错误码分类。

适用范围：

- 钱包 API
- 开放清算 API
- Chat Bot HTTP API（`https://chat.kuma.homes`）
- 其他通过 `toolbear_ui` 暴露的对外 HTTP API

本规范只描述错误模型本身。具体接口、字段、状态机与业务语义，仍以各 API
规范为准：

- [Lilium 钱包 API 接入规范 v1](./lilium-wallet-api-design.md)
- [Lilium 开放清算 API 接入规范 v1.1](./lilium-open-clearing-api-design.md)
- [Lilium Chat Bot API 接入规范 v1](./lilium-chat-bot-api-design.md)
- [Lilium 平台认证与路由规范 v1.1](./lilium-platform-authentication.md)

<a id="envelope"></a>
## 2. 统一错误响应结构

所有非 `2xx` 响应统一返回：

```json
{
  "error": {
    "code": "INVALID_FIELD",
    "message": "request validation failed",
    "retryable": false,
    "details": [
      {
        "loc": ["body", "amount"],
        "msg": "Field required",
        "type": "missing"
      }
    ]
  },
  "request_id": "req_001"
}
```

字段语义：

- `error.code`
  - 稳定、可编程处理的错误码
  - 调用方应优先依赖它做分支判断
- `error.message`
  - 人类可读错误描述
  - 可用于日志、运营排障、前端展示
- `error.retryable`
  - 是否建议调用方直接重试同一请求
  - 默认 `false`
- `error.details`
  - 可选的结构化附加信息
  - 当服务端掌握更细粒度的校验或错误上下文时返回
  - 调用方可将其用于调试、表单提示或日志，不应假设它在所有错误中都存在
- `request_id`
  - 当前错误响应的请求标识
  - 用于日志检索、工单排障与跨系统追踪

<a id="rules"></a>
## 3. 通用规则

### 3.1 所有外部 API 共用同一错误 envelope

钱包、清算以及其他对外 API 的区别只体现在：

- HTTP status
- `error.code`
- `error.message`

不体现在错误响应结构本身。

### 3.2 文案不是主契约

调用方不应依赖完整 `error.message` 进行程序逻辑判断。  
稳定契约应以：

- HTTP status
- `error.code`

为主。

<a id="common-codes"></a>
## 4. 通用错误码

以下错误码可跨多个 API 能力复用：

- `UNAUTHORIZED`
- `UNAUTHORIZED_PARTNER`
- `INVALID_SCOPE`
- `INVALID_REQUEST`
- `INVALID_FIELD`
- `NOT_FOUND`
- `CONFLICT`
- `INTERNAL_ERROR`
- `MACHINE_TOKEN_NOT_ALLOWED`
- `PARTNER_SUBACCOUNTS_NOT_ALLOWED`
- `ADMIN_ACCESS_REQUIRED`
- `INVALID_CLIENT`
- `RESUME_NOT_FOUND`

说明：

- 某些错误码来自平台认证与权限模型
- 某些错误码来自统一 HTTP / validation / access 控制
- 具体接口是否会返回这些错误码，以各 API 规范和实现为准

<a id="wallet-codes"></a>
## 5. 钱包能力错误码

钱包转账等相关接口会使用以下能力专属错误码：

- `MISSING_IDEMPOTENCY_KEY`
- `INVALID_IDEMPOTENCY_KEY`
- `RECIPIENT_NOT_FOUND`
- `INVALID_RECIPIENT`
- `SELF_TRANSFER`
- `INSUFFICIENT_BALANCE`
- `INVALID_AMOUNT`
- `IDEMPOTENCY_CONFLICT`

典型语义：

- 缺少幂等 key -> `400`
- 收款方不存在 -> `404`
- 非法收款方 / 自转账 / 金额非法 / 余额不足 -> `422`
- 幂等冲突 -> `409`

钱包接口的完整业务上下文见：

- [Lilium 钱包 API 接入规范 v1](./lilium-wallet-api-design.md)

<a id="clearing-codes"></a>
## 6. 清算能力错误码

清算、支付意图、批量清算与 hosted checkout 相关接口会使用以下能力专属错误码：

- `INVALID_ACCOUNT_CODE`
- `INVALID_OPERATION`
- `INTENT_NOT_CONFIRMABLE`
- `INTENT_EXPIRED`
- `USER_BALANCE_INSUFFICIENT`
- `INSUFFICIENT_ESCROW`
- `PARTNER_BALANCE_INSUFFICIENT`
- `INVALID_BATCH_SIZE`
- `WEBHOOK_SIGNATURE_INVALID`
- `RATE_LIMIT_EXCEEDED`

说明：

- 清算能力仍沿用统一 envelope
- 清算与钱包的差异通过 `error.code` 集合体现，而不是通过不同 schema

清算接口的完整业务上下文见：

- [Lilium 开放清算 API 接入规范 v1.1](./lilium-open-clearing-api-design.md)
