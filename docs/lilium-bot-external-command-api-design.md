# Lilium Bot External Command 接入规范 v1

适用对象：接入 Lilium Bot 外部命令能力的第三方开发者  
文档状态：草案

## 目录

- [1. 文档说明](#overview)
- [2. 能力边界](#boundary)
- [3. 接入登记信息](#registration)
- [4. 无状态命令 HTTP 入口](#stateless-http)
- [5. 请求签名](#request-signature)
- [6. 调用信封](#invoke-envelope)
- [7. 响应 Content-Type](#response-content-type)
- [8. JSON 响应](#json-response)
- [9. SSE 响应](#sse-response)
- [10. Effect 数据结构](#effects)
- [11. 错误模型](#errors)
- [12. 幂等与重试](#idempotency)
- [13. 安全要求](#security)
- [14. Stateful WebSocket 模式预告](#stateful-ws)
- [15. HTTP Callback 预留](#http-callback)
- [16. 接入清单](#checklist)

<a id="overview"></a>
## 1. 文档说明

Lilium Bot External Command 允许第三方服务为聊天室提供 slash command 能力。
用户在聊天室发送命令，例如：

```text
/demo args
```

Lilium Bot 会将命令调用转发到第三方服务。第三方服务返回结构化 `effects`，
由 Lilium Bot 在聊天室中执行输出。

本规范当前定义 **无状态 HTTP 命令**。  
Stateful WebSocket 命令属于后续扩展，见 [14. Stateful WebSocket 模式预告](#stateful-ws)。

本规范不定义 Lilium 内部实现、部署结构、数据库表或 Bot 运行时细节。第三方只需要实现本文档描述的 HTTP endpoint、验签、响应格式与错误处理规则。

协议版本：

- `lilium.external-command.v1`

<a id="boundary"></a>
## 2. 能力边界

Lilium 负责：

- 命令注册与展示
- 房间 ACL、命令开关与 rate limit
- 用户消息解析
- 向第三方服务发起签名请求
- 校验第三方响应
- 执行合法 `effects`
- 在聊天室发送消息或图片
- 记录调用结果与排障信息

第三方负责：

- 命令业务逻辑
- 参数解析与业务校验
- 返回结构化结果
- 保持自身业务幂等性
- 校验 Lilium 请求签名
- 保护共享密钥

第三方不能：

- 直接访问 Lilium Bot 内部状态
- 指定任意聊天室发送消息
- 直接执行管理操作
- 直接发起钱包、清算或其他平台动作
- 返回未定义的 effect 类型并期待 Lilium 执行

<a id="registration"></a>
## 3. 接入登记信息

接入外部命令前，第三方需要向 Lilium 提供命令登记信息。

### 3.1 命令元数据

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `config_id` | 是 | 外部命令配置标识，例如 `demo`。用于签名 `keyid` 与排障 |
| `name` | 是 | slash command 名称，必须以 `/` 开头，例如 `/demo` |
| `description` | 是 | 命令短描述，用于命令列表 |
| `help_text` | 否 | 命令详细帮助文本 |
| `aliases` | 否 | 命令别名，例如 `/演示` |
| `group` | 否 | 命令分组，例如 `外部命令` |

说明：

- `name` 与 `aliases` 不能和现有命令冲突。
- 命令是否在某个房间可用，由 Lilium 的房间 ACL、命令开关与 rate limit 决定。
- 第三方不需要在请求中自行判断房间 ACL；Lilium 会在调用第三方 endpoint 前完成准入检查。
- `llm_note` 属于 Lilium 内部审核和维护字段，不作为第三方提交字段。第三方应把用户可见说明写入 `description` / `help_text`。

### 3.2 外部 endpoint 配置

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `endpoint` | 是 | 第三方接收命令调用的 HTTPS endpoint |
| `timeout_ms` | 是 | Lilium 等待第三方响应的超时时间 |
| `shared_secret` | 是 | Lilium 与第三方共享的 HMAC 密钥字符串 |

v1 当前只开放无状态 HTTP 命令，等价于 `stateless` 模式。第三方不需要在登记材料里选择或提交 `mode`。

示例登记材料：

| 项 | 示例 |
| --- | --- |
| `config_id` | `demo` |
| `name` | `/demo` |
| `description` | `调用第三方 Demo 服务` |
| `help_text` | `## /demo\n\n**用法**\n- /demo args` |
| `aliases` | `/演示` |
| `group` | `外部命令` |
| `endpoint` | `https://partner.example.com/lilium/commands/demo` |
| `timeout_ms` | `30000` |
| `shared_secret` | `replace-with-a-random-shared-secret` |

<a id="stateless-http"></a>
## 4. 无状态命令 HTTP 入口

第三方必须实现：

```http
POST <endpoint>
```

Lilium 会发送：

```http
POST /lilium/commands/demo HTTP/1.1
Host: partner.example.com
Content-Type: application/json
Accept: application/json, text/event-stream
Idempotency-Key: cmd_01H...
Content-Digest: sha-256=:BASE64_BODY_SHA256:
Signature-Input: lilium=("@method" "@authority" "@path" "content-type" "accept" "idempotency-key" "content-digest");created=1778395200;expires=1778395230;keyid="demo";alg="hmac-sha256"
Signature: lilium=:BASE64_HMAC_SIGNATURE:
```

请求体是 [6. 调用信封](#invoke-envelope) 定义的 JSON。

第三方可以返回：

- `application/json`
- `text/event-stream`

Lilium 根据响应 `Content-Type` 自动选择解析方式。命令登记信息中不配置响应模式。

<a id="request-signature"></a>
## 5. 请求签名

Lilium 使用 RFC 9421 HTTP Message Signatures 对请求签名。

签名规则：

- 签名 label 固定为 `lilium`
- `keyid` 等于外部命令 `config_id`
- `alg` 为 `hmac-sha256`
- HMAC key 为登记时配置的 `shared_secret`
- `created` 与 `expires` 必填
- 默认签名有效期为 `30` 秒
- 请求必须包含 `Content-Digest`

签名覆盖组件至少包括：

- `@method`
- `@authority`
- `@path`
- `content-type`
- `accept`
- `idempotency-key`
- `content-digest`

第三方验签要求：

- 校验 `Signature-Input` 与 `Signature`
- 校验 `keyid` 是否对应已登记命令
- 校验 `created` / `expires` 时间窗口
- 校验 `Content-Digest` 是否匹配请求 body
- 拒绝过期、未来时间过大、digest 不匹配或签名不合法的请求

`shared_secret` 只用于 HMAC，不得写入响应、日志或错误消息。

### 5.1 响应不要求签名

v1 只要求第三方校验 Lilium 发出的请求签名。第三方响应不要求使用 RFC 9421
签名，也不要求提供 `Content-Digest`。

响应侧的协议校验依赖：

- HTTPS 传输
- HTTP status
- 响应 `Content-Type`
- JSON / SSE schema
- `api_version`
- effect 顺序与 payload 校验

<a id="invoke-envelope"></a>
## 6. 调用信封

请求 body：

```json
{
  "api_version": "lilium.external-command.v1",
  "type": "command.invoke",
  "invocation_id": "cmd_01H...",
  "sent_at": "2026-05-10T12:00:00Z",
  "command": {
    "config_id": "demo",
    "name": "/demo",
    "matched_name": "/demo",
    "args": "args",
    "argv": ["args"],
    "raw_text": "/demo args",
    "mode": "stateless"
  },
  "room": {
    "id": "room_123",
    "type": "group"
  },
  "sender": {
    "id": "user_123"
  },
  "message": {
    "id": "msg_123",
    "text": "/demo args",
    "created_at": "2026-05-10T12:00:00Z"
  }
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `api_version` | 固定为 `lilium.external-command.v1` |
| `type` | 固定为 `command.invoke` |
| `invocation_id` | Lilium 生成的本次调用 ID |
| `sent_at` | Lilium 发起请求时间 |
| `command.config_id` | 命令配置标识 |
| `command.name` | 命令主名称 |
| `command.matched_name` | 用户实际触发的名称，可能是 alias |
| `command.args` | 命令名后的原始参数字符串 |
| `command.argv` | Lilium 对 `args` 做的简单空白切分，复杂解析仍应以 `args` 为准 |
| `command.raw_text` | 用户原始消息文本 |
| `room.id` | 聊天室标识 |
| `room.type` | 聊天室类型，可能为空 |
| `sender.id` | 发送者 Lilium 用户标识 |
| `message.id` | 触发命令的消息 ID |

所有 ID 字段都按字符串处理。第三方不得假设 `sender.id`、`room.id` 或
`message.id` 是 UUID，也不得截断或大小写折叠。

### 6.1 协议能力

v1 协议能力由 `api_version = "lilium.external-command.v1"` 固定定义，不在每次调用信封中重复携带。

`lilium.external-command.v1` 支持：

- JSON 响应：`application/json`
- SSE 响应：`text/event-stream`
- effect 类型：`reply`、`send_text`、`send_image_url`、`noop`
- `markdown` 字段，默认值为 `true`

如果未来新增 effect 类型或协商能力，应通过新协议版本或单独的版本能力文档定义，不应把静态能力列表放入每次调用请求。

<a id="response-content-type"></a>
## 7. 响应 Content-Type

第三方必须通过 `Content-Type` 表达响应格式。

支持：

- `application/json`
- `text/event-stream`

允许带参数：

- `application/json; charset=utf-8`
- `text/event-stream; charset=utf-8`

不支持：

- `text/plain`
- `text/html`
- 未设置 `Content-Type`
- 与响应体不匹配的 `Content-Type`

HTTP 非 `2xx` 响应会被 Lilium 视为集成错误。Lilium 不会把非 `2xx` 响应 body 直接转发到聊天室。

<a id="json-response"></a>
## 8. JSON 响应

成功响应：

```json
{
  "api_version": "lilium.external-command.v1",
  "type": "command.result",
  "invocation_id": "cmd_01H...",
  "status": "ok",
  "effects": [
    {
      "type": "reply",
      "text": "处理完成",
      "markdown": true
    }
  ]
}
```

业务拒绝响应可以保持很短，用户可见文案放在 effect 中：

```json
{
  "api_version": "lilium.external-command.v1",
  "type": "command.result",
  "invocation_id": "cmd_01H...",
  "status": "rejected",
  "effects": [
    {
      "type": "reply",
      "text": "参数不完整",
      "markdown": true
    }
  ]
}
```

`status` 取值：

- `ok`
- `rejected`

规则：

- 第三方业务拒绝必须返回 HTTP `2xx` + `status: "rejected"`。
- 业务拒绝的用户可见文案应通过 `effects` 返回。
- `error` 是可选诊断字段，不应承载额外的用户展示文案。
- 如果不返回 `effects`，Lilium 可以只记录拒绝结果，不保证向聊天室发送第三方提供的错误文本。

<a id="sse-response"></a>
## 9. SSE 响应

当第三方需要分多次返回结果时，可以使用 SSE。

响应头：

```http
Content-Type: text/event-stream
Cache-Control: no-cache
```

事件示例：

```text
event: effect
data: {"type":"reply","text":"第一条","markdown":true}

event: effect
data: {"type":"reply","text":"第二条","markdown":true}

event: done
data: {"status":"ok"}
```

SSE 规则：

- `effect` event 的 `data` 必须是合法 effect JSON。
- Lilium 按 SSE event 到达顺序执行 effects。
- `done` event 表示命令结束。
- `done.data.status` 取值为 `ok` 或 `rejected`。
- 流在 `done` 前断开时，Lilium 将本次调用标记为集成错误。
- malformed event 会导致 Lilium 停止消费后续事件。

<a id="effects"></a>
## 10. Effect 数据结构

JSON 响应的 `effects` 数组直接包含 effect payload。SSE 响应的 `effect`
event `data` 也直接包含 effect payload。

示例：

```json
{
  "type": "reply",
  "text": "处理完成",
  "markdown": true
}
```

执行顺序：

- JSON 响应按 `effects` 数组顺序执行。
- SSE 响应按 `effect` event 到达顺序执行。
- Stateless HTTP v1 不要求第三方提供 `effect_id` 或 `sequence`。
- Stateful WebSocket 后续会在 effect frame 外层增加 `sequence` / `effect_id`，
  用于 ack、resume 和去重；这不属于当前无状态 HTTP 响应格式。

### 10.1 `reply`

回复触发命令的消息。

```json
{
  "type": "reply",
  "text": "处理完成",
  "markdown": true
}
```

### 10.2 `send_text`

向当前聊天室发送普通文本消息。

```json
{
  "type": "send_text",
  "text": "房间消息",
  "markdown": true
}
```

### 10.3 `send_image_url`

向当前聊天室发送图片 URL。

```json
{
  "type": "send_image_url",
  "url": "https://partner.example.com/images/result.png",
  "caption": "可选说明"
}
```

### 10.4 `noop`

不产生聊天室输出。

```json
{
  "type": "noop"
}
```

通用规则：

- effect 只能作用于触发命令的聊天室。
- 第三方不能指定目标 room。
- 未知 `effect.type` 会被拒绝。
- `markdown` 默认值为 `true`。
- Lilium 会限制单次调用的 effect 数量、文本长度和媒体 URL 安全性。

<a id="errors"></a>
## 11. 错误模型

### 11.1 本地准入错误

以下错误发生在 Lilium 本地，不会调用第三方 endpoint：

- 命令不存在
- 命令在当前房间不可用
- 命令被房间关闭
- rate limit 命中
- 管理员命令权限不足

### 11.2 第三方业务拒绝

第三方已经理解请求，但因业务规则拒绝执行时，应返回：

- HTTP `2xx`
- `Content-Type: application/json` 或 `text/event-stream`
- `status: "rejected"`
- 可选 `error`
- 可选 `effects`

不要用 HTTP `400` 表示用户参数错误。HTTP 非 `2xx` 在 Lilium 侧会被归类为集成错误。

### 11.3 集成错误

以下情况会被 Lilium 视为集成错误：

- 第三方 endpoint 超时
- 网络、DNS、TLS 失败
- HTTP 非 `2xx`
- 响应缺少或使用不支持的 `Content-Type`
- JSON / SSE schema 不合法
- `api_version` 不支持
- effect schema 不合法

集成错误不会把第三方原始响应直接展示给用户。Lilium 会返回通用失败提示，并记录排障信息。

### 11.4 Effect 执行错误

Lilium 收到合法响应后，执行 effect 时仍可能失败，例如：

- 文本过长
- 图片 URL 不允许
- 图片发送失败
- 聊天室发送失败

v1 使用保守策略：某个 effect 执行失败后，Lilium 可以停止执行后续 effects，并将本次调用标记为失败或部分失败。

<a id="idempotency"></a>
## 12. 幂等与重试

v1 默认不要求 Lilium 自动重试命令调用。

Lilium 会在请求头中发送 `Idempotency-Key`。v1 中该值默认等于
`invocation_id`，并被纳入 RFC 9421 签名覆盖范围。

第三方仍应按 `Idempotency-Key` 做幂等保护：

- 同一个 `Idempotency-Key` 的重复请求应返回同一业务结果，或安全地拒绝重复执行。
- 如果第三方命令会产生外部副作用，必须在第三方系统内保证不会重复提交。
- `invocation_id` 可用于日志检索和跨系统排障。

SSE 场景下，第三方应保证同一次 invocation 内：

- 返回的 effect 序列稳定
- 断线后的重复处理不会产生不同业务结果

<a id="security"></a>
## 13. 安全要求

- 生产环境 endpoint 必须使用 HTTPS。
- `shared_secret` 至少应使用 32 字节随机值。
- 第三方必须校验 RFC 9421 请求签名。
- 第三方必须校验 `Content-Digest`。
- 第三方必须拒绝过期签名。
- 第三方不得在日志中记录完整 `shared_secret` 或完整 `Signature`。
- 第三方不得将 Lilium 请求 body 原样转发到不可信系统。
- 第三方返回的 `text` 不应包含 HTML 或脚本片段。
- 第三方提供的图片 URL 应稳定可访问，且不应指向内网地址。
- 第三方应对自身 endpoint 做限流和监控。

<a id="stateful-ws"></a>
## 14. Stateful WebSocket 模式预告

Stateful WebSocket 模式用于需要持续监听聊天室消息或 timer 的命令，例如游戏或多轮交互。

该模式当前为规划能力，v1 无状态 HTTP 接入不要求实现。

设计方向：

- Lilium Bot 主动连接第三方 WebSocket endpoint。
- 第三方维护业务权威状态。
- Lilium 负责聊天室监听、timer、`/stop` 本地释放和 effect 执行。
- 第三方通过 WebSocket 推送 effects。
- Lilium 对已执行 effect 返回 ack。
- Bot 重启后通过 `session.resume` 携带最后 ack 的 `sequence` / `effect_id` 恢复。

Stateful WebSocket 将继续使用：

- `api_version: "lilium.external-command.v1"`
- RFC 9421 风格的握手认证或等价签名认证
- 与无状态命令相同的 effect schema

正式开放前，会补充单独的 WebSocket frame 规范。

<a id="http-callback"></a>
## 15. HTTP Callback 预留

HTTP Callback 不是当前开放能力。

如果未来支持 callback，第三方也不能直接操作 Lilium Bot 运行时状态。callback 会作为单独 ingress 模式设计，并要求 Lilium 在服务端完成验签、持久化与异步消费。

当前接入方不应依赖 callback。

<a id="checklist"></a>
## 16. 接入清单

第三方接入前确认：

- 已向 Lilium 提供命令 `config_id`、`name`、说明和 endpoint。
- endpoint 支持 `POST` JSON 请求。
- endpoint 按 `Idempotency-Key` 做幂等保护。
- endpoint 校验 RFC 9421 `Signature-Input` 与 `Signature`。
- endpoint 校验 `Content-Digest`。
- endpoint 根据业务结果返回 `application/json` 或 `text/event-stream`。
- 业务拒绝使用 `status: "rejected"`，不使用 HTTP `400` 表示用户参数错误。
- 返回的 effect 类型属于本文档定义范围。
- endpoint 超时、重复请求和业务副作用都有幂等保护。
- 生产环境使用 HTTPS，并妥善保护 `shared_secret`。
