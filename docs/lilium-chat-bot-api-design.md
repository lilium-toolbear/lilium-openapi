# Lilium Chat Bot API 接入规范 v1

适用对象：在 ToolBear Chat 上开发外置 Bot 的第三方开发者
文档状态：草案

## 目录

- [1. 文档说明](#overview)
- [2. 系统边界](#scope)
- [3. 公开 API](#public-boundary)
- [4. 认证](#entrypoints)
- [5. 接入流程](#onboarding)
- [6. Command Catalog 同步](#bot-http)
- [7. Bot Gateway WebSocket](#bot-ws)
- [8. Stateful Session 帧协议](#stateful-session)
- [9. Slash Command 与频道绑定](#slash-commands)
- [10. Effects 与消息 Components](#effects)
- [11. 错误模型](#errors)
- [12. 幂等与重试](#idempotency)
- [13. 安全要求](#security)
- [14. 接入清单](#checklist)

<a id="overview"></a>
## 1. 文档说明

Lilium Chat Bot API 面向在 **ToolBear Chat** 上运行的外置 Bot。集成方使用 **Chat Bot Token** 连接运行时：

```http
Authorization: Bearer <bot_token>
```

接入前须持有 Chat Bot Token（`lcbot_...`）。Token 的创建、scope 配置与轮换由 Bot 所有者完成。

Bot 进程连接 Chat 后端；**不需要** Bot 暴露公网 HTTP 回调地址。

| 项目 | 值 |
| --- | --- |
| Chat API Base | `https://chat.kuma.homes` |
| Bot Gateway WebSocket | `wss://chat.kuma.homes/api/chat/bot/ws` |
| Bot 协议版本 | `lilium.chat.bot.v1` |

本规范只描述第三方需要知道的公开入口、认证方式、字段与安全要求。不描述内部实现，服务器代码请参考 `https://github.com/lilium-toolbear/lilium-chat`。

相关文档：

- [Lilium API 错误模型规范 v1](./lilium-error-model.md)（HTTP 错误信封；Bot Gateway 帧内错误见本文 §11）

<a id="scope"></a>
## 2. 系统边界

本文档**仅**适用于 **ToolBear Chat**（`https://chat.kuma.homes`）。

同仓库中的 [Lilium Bot External Command 接入规范](./lilium-bot-external-command-api-design.md) 描述的是 **DZMM Bot**（`dzmm_archive` / `bots/`）的外部命令回调能力，面向 DZMM.ai 聊天室生态，与 ToolBear Chat **完全无关**。两套系统、协议、运行时互不兼容，文档也不应交叉引用为「可选接入路径」。

<a id="public-boundary"></a>
## 3. 公开 API

| Method | Path | Scope | 说明 |
| --- | --- | --- | --- |
| `PUT` | `/api/chat/bot/commands` | `chat:commands:manage` | 同步全局 slash command catalog |
| `GET` | `/api/chat/bot/ws` | `chat:runtime:connect` | Bot Gateway WebSocket 升级 |

Bot 发消息、改消息、流式输出、components **只**通过 Gateway WS 的 `delivery_result.effects` 或 `session.effects`（见 §7、§10）。Bot Token 的 HTTP 面**仅** `PUT /api/chat/bot/commands` 用于 catalog 同步，**无**消息 mutation HTTP 端点。

<a id="entrypoints"></a>
## 4. 认证

### 4.1 Chat Bot Token

Bot 进程所有 Runtime 请求使用 Chat Bot Token：

```http
Authorization: Bearer <bot_token>
```

WebSocket 额外要求 subprotocol：

```http
GET /api/chat/bot/ws HTTP/1.1
Authorization: Bearer <bot_token>
Sec-WebSocket-Protocol: lilium.chat.bot.v1
```

### 4.2 Scopes

| Scope | 用途 |
| --- | --- |
| `chat:runtime:connect` | 连接 Bot Gateway WebSocket |
| `chat:commands:manage` | `PUT /api/chat/bot/commands` |
| `chat:messages:write` | `delivery_result` / `session.effects` 中的发消息类 effect |

Scope 不足返回 `403`，`error.code` 为 `FORBIDDEN` 或 `BOT_SCOPE_DENIED`。

### 4.3 通用 HTTP 约定

- 所有 mutating HTTP 请求必须带 `Idempotency-Key`（client-generated UUIDv7）
- 响应头 `X-Request-Id`；请求可传 `X-Request-Id` 覆盖
- 时间字段为 ISO 8601 UTC
- ID 为不透明字符串，调用方不得解析其内部结构

<a id="onboarding"></a>
## 5. 接入流程

```text
1. 从 Bot 所有者处取得 Chat Bot Token（含所需 scope）
2. Bot 进程启动：
   a. PUT /api/chat/bot/commands
   b. 连接 wss://.../api/chat/bot/ws，hello → ready
3. 请频道管理员在 ToolBear 中为该频道 allow 你的 command
4. 用户 invoke → Bot 收 delivery / session.start → 回 delivery_result / session.*
```

**频道限制**：Bot 仅在 `kind=channel` 群聊频道工作；DM 不支持 slash command（`409 UNSUPPORTED_CHANNEL_KIND`）。

**在线要求**：stateless invoke 与 stateful session 启动前会检查 Bot Gateway 连接；离线时用户侧收到 `503 BOT_OFFLINE`（`retryable=true`）。Bot 进程应常驻 WS 并带指数退避重连。

<a id="bot-http"></a>
## 6. Command Catalog 同步

Bot Runtime 的 HTTP 面只有 catalog 同步。消息 mutation 见 §7 Bot Gateway WebSocket。

### 6.1 同步 Slash Command Catalog

```http
PUT /api/chat/bot/commands
Authorization: Bearer <bot_token>
Idempotency-Key: <uuidv7>
Content-Type: application/json
```

请求体：

```json
{
  "commands": [
    {
      "name": "ask",
      "aliases": ["ai"],
      "description": "Ask the assistant",
      "help_text": "Usage: /ask prompt:<text>",
      "options": [
        {
          "name": "prompt",
          "type": "string",
          "required": true,
          "description": "Question"
        }
      ],
      "default_member_permission": "member",
      "execution": { "mode": "stateless" }
    },
    {
      "name": "werewolf",
      "aliases": [],
      "description": "Stateful game session",
      "help_text": "Start a werewolf game in this channel",
      "options": [],
      "default_member_permission": "member",
      "execution": {
        "mode": "stateful",
        "stateful": {
          "mutex_scope": "channel",
          "default_ttl_seconds": 3600,
          "max_ttl_seconds": 7200,
          "listen_capability": {
            "message_types": ["text"],
            "include_bot_messages": false,
            "include_own_messages": true
          }
        }
      }
    }
  ]
}
```

字段要点：

| 字段 | 说明 |
| --- | --- |
| `name` | Canonical slash 名称，不含 `/`；全站唯一 |
| `aliases` | 别名；用户 invoke 时通过 `invoked_name` 区分 |
| `help_text` | 平台 `/help <command>` 展示用 |
| `options[].type` | `string` \| `integer` \| `number` \| `boolean` \| `user` \| `channel` \| `role` |
| `default_member_permission` | `member` \| `admin` \| `owner` |
| `execution.mode` | `stateless` \| `stateful` |
| `execution.stateful` | stateful 时必填：mutex、TTL、`listen_capability` |

全站 slash 名称冲突返回 `409 COMMAND_NAME_CONFLICT`。

响应 `200`：

```json
{
  "commands": [
    {
      "bot_command_id": "00000000-0000-7000-8000-000000000701",
      "name": "ask",
      "aliases": ["ai"],
      "status": "active",
      "execution_mode": "stateless",
      "stateful_config": null,
      "definition_hash": "sha256:...",
      "schema_version": 3,
      "updated_at": "2026-06-21T05:30:00Z"
    }
  ]
}
```

Bot 应在每次启动或 catalog 变更后调用此接口。频道侧 allow 时绑定的是 `bot_command_id` 与 `definition_hash` 快照。

<a id="bot-ws"></a>
## 7. Bot Gateway WebSocket

Bot 侧所有消息 mutation（发消息、改消息、流式、components）**只**在本节 WebSocket 上通过 `delivery_result.effects` 或 `session.effects` 提交。

### 7.1 建连与握手

Bot 连接后发 `hello`：

```json
{
  "type": "hello",
  "api_version": "lilium.chat.bot.v1",
  "last_received_delivery_id": null
}
```

重连时填入上次成功处理的 `delivery_id`，用于去重与恢复。

Server 回 `ready`：

```json
{
  "type": "ready",
  "api_version": "lilium.chat.bot.v1",
  "bot_id": "00000000-0000-7000-8000-000000000601",
  "session_id": "00000000-0000-7000-8000-000000000901",
  "server_time": "2026-06-26T00:00:00Z"
}
```

### 7.2 Delivery 种类

| `kind` | 触发来源 | Bot 离线时 |
| --- | --- | --- |
| `command_invocation` | 用户 slash invoke（stateless） | 用户侧 `503 BOT_OFFLINE` |
| `message_interaction` | 用户点击 Bot 消息的 button/select | 同上 |
| `message_event` | stateful `listen_capability` 匹配的频道消息 | 丢弃，无用户可见错误 |

所有 delivery 为 **at-least-once**。Bot 必须按 `delivery_id` 去重。

### 7.3 Stateless：`command_invocation`

```json
{
  "type": "delivery",
  "api_version": "lilium.chat.bot.v1",
  "delivery_id": "01J...",
  "kind": "command_invocation",
  "invocation_id": "00000000-0000-7000-8000-000000000811",
  "channel_id": "00000000-0000-7000-8000-000000000201",
  "command": {
    "bot_command_id": "00000000-0000-7000-8000-000000000701",
    "name": "ask",
    "invoked_name": "ask",
    "schema_version": 3,
    "definition_hash": "sha256:...",
    "options": {
      "prompt": { "type": "string", "value": "hello" }
    }
  },
  "invoker": {
    "user_id": "00000000-0000-7000-8000-000000000101",
    "display_name": "alice",
    "avatar_url": null
  },
  "reply_to": {
    "message_id": "00000000-0000-7000-8000-000000000301",
    "type": "text",
    "status": "normal",
    "text": "original message",
    "sender": {
      "kind": "user",
      "user": {
        "user_id": "00000000-0000-7000-8000-000000000101",
        "display_name": "alice",
        "avatar_url": null
      }
    }
  }
}
```

`reply_to` 仅在用户 invoke 时携带 `reply_to_message_id` 时出现；否则省略或为 `null`。

### 7.4 `message_interaction`

用户点击 Bot 消息上的交互组件时推送：

```json
{
  "type": "delivery",
  "api_version": "lilium.chat.bot.v1",
  "delivery_id": "01J...",
  "kind": "message_interaction",
  "interaction_id": "00000000-0000-7000-8000-000000000a21",
  "channel_id": "00000000-0000-7000-8000-000000000201",
  "message_id": "00000000-0000-7000-8000-000000000301",
  "component": {
    "component_id": "00000000-0000-7000-8000-000000000a01",
    "custom_id": "confirm",
    "value": true
  },
  "actor": {
    "user_id": "00000000-0000-7000-8000-000000000101",
    "display_name": "alice",
    "avatar_url": null
  }
}
```

### 7.5 `message_event`

Stateful command 的 `listen_capability` 匹配时推送频道内新消息。`message` 为完整消息投影；`sender` 为 `user` 或 `bot` 形状。

### 7.6 `delivery_result` 与 `delivery_ack`

Bot 处理完毕后发送：

```json
{
  "type": "delivery_result",
  "api_version": "lilium.chat.bot.v1",
  "delivery_id": "01J...",
  "status": "ok",
  "effects": []
}
```

`status` 为 `ok` 或 `failed`。失败时可带 `error: { code, message }`。

Server 应用 effects 后回：

```json
{
  "type": "delivery_ack",
  "api_version": "lilium.chat.bot.v1",
  "delivery_id": "01J...",
  "status": "applied"
}
```

失败 ack：

```json
{
  "type": "delivery_ack",
  "api_version": "lilium.chat.bot.v1",
  "delivery_id": "01J...",
  "status": "failed",
  "error": {
    "code": "BOT_EFFECT_INVALID",
    "message": "..."
  }
}
```

<a id="stateful-session"></a>
## 8. Stateful Session 帧协议

用户对 `execution.mode=stateful` 的 command 发起 invoke 后，Chat 经 Bot Gateway 下发会话帧（非 `command_invocation` delivery）。

### 8.1 帧一览

| 方向 | 帧类型 | 说明 |
| --- | --- | --- |
| Server → Bot | `session.start` | 会话创建，含 invoker、options、`listen_rules` |
| Bot → Server | `session.start_ack` | 确认 `session.start` 已就绪；此后频道才广播 `stateful_session.started` |
| Server → Bot | `session.input` | 频道内匹配 `listen_capability` 的消息 |
| Bot → Server | `session.input_ack` | 确认已处理到的 `seq` |
| Bot → Server | `session.effects` | 会话期内的 effects（与会话绑定） |
| Server → Bot | `session.effects_ack` | effects 应用结果 |
| Bot → Server | `session.close` | Bot 主动结束会话 |
| Server → Bot | `session.closed` | 会话结束通知 |

Bot → Server 对 Server 推送/命令的确认帧统一 `{…}_ack`（`start_ack`、`input_ack`）。Server → Bot 的会话终结通知用过去式 `session.closed`。频道 timeline 事件 `stateful_session.started` 属于另一命名空间，与 WS 帧类型不同。

`session.start` 除 `command` / `invoker` / `options` 外，还包含 `listen_rules`（来自 catalog 的 `listen_capability`）、`input_seq_start`、`expires_at`，以及可选的 `reply_to`（语义同 §7.3）。

<a id="slash-commands"></a>
## 9. Slash Command 与频道绑定

### 9.1 全局 catalog + 频道 allow/block

1. Bot 通过 `PUT /api/chat/bot/commands` 声明全局 catalog（全站 slash 名称唯一）。
2. 频道 owner/admin 在 ToolBear 频道设置中对单条 command 执行 allow 或 block。
3. 用户仅在 command 被 allow 且 Bot 在线时才能成功 invoke。

Allow 时可配置 `permission_override`（`member` | `admin` | `owner`）与 stateful command 的 `stateful_max_ttl_seconds`。每次绑定变更递增频道 `command_manifest_version` 并 fanout `command.binding_updated`。

### 9.2 平台 `/help`

平台内置 `bot_id` `00000000-0000-7000-8000-000000000600` 提供 `/help`；同步写入 Bot 消息，**不**经 Bot Gateway delivery。第三方 Bot 的 `help_text` 通过 catalog 同步供 `/help <command>` 查询。

<a id="effects"></a>
## 10. Effects 与消息 Components

### 10.1 Effect 类型

Bot 通过 `delivery_result.effects` 或 `session.effects` 改变频道状态。每个 effect 需要 `client_effect_id`（UUIDv7）。

| Effect | 说明 |
| --- | --- |
| `send_message` | 发送 Bot 消息，可带 `components` |
| `update_message` | 更新本 Bot 发送的消息 |
| `disable_components` | 禁用本 Bot 消息上的交互组件 |
| `start_stream` | 创建 `stream_state=streaming` 的消息 |
| `append_stream` | 向 streaming 消息追加 `delta` |
| `finalize_stream` | 将 streaming 消息标为 `final` |

幂等键：`(channel_id, bot_id, client_effect_id)`。同 `client_effect_id` 不同 body → `409 BOT_EFFECT_CONFLICT`。

`append_stream` / `update_message` / `disable_components` 只能操作**本 Bot 创建**的消息。

### 10.2 Bot Actor

Bot 消息的发送者形状：

```json
{
  "kind": "bot",
  "bot": {
    "bot_id": "00000000-0000-7000-8000-000000000601",
    "display_name": "My Bot",
    "avatar_url": "https://example.com/bot.png"
  }
}
```

`display_name` / `avatar_url` 来自 BotRegistry，不查 ToolBear 用户表。

### 10.3 交互 Components

Bot 消息可携带 `button` 或 `select` 组件。用户操作后产生 `message_interaction` delivery。普通用户消息**不能**带 `components`。

Button 示例：

```json
{
  "component_id": "00000000-0000-7000-8000-000000000a01",
  "kind": "button",
  "style": "primary",
  "label": "确认",
  "custom_id": "confirm",
  "disabled": false
}
```

<a id="errors"></a>
## 11. 错误模型

### 11.1 HTTP 错误

与 [Lilium API 错误模型](./lilium-error-model.md) 相同信封：

```json
{
  "error": {
    "code": "BOT_OFFLINE",
    "message": "bot is not connected",
    "retryable": true
  },
  "request_id": "req_..."
}
```

### 11.2 Bot 相关错误码

| HTTP | Code | 说明 | Retryable |
| --- | --- | --- | --- |
| 401 | `UNAUTHORIZED` / `BOT_TOKEN_INVALID` / `BOT_TOKEN_REVOKED` | Token 无效或已撤销 | 否 |
| 403 | `FORBIDDEN` / `BOT_SCOPE_DENIED` / `BOT_DISABLED` | Scope 不足或 Bot 已禁用 | 否 |
| 403 | `COMMAND_PERMISSION_DENIED` | 用户无权 invoke | 否 |
| 404 | `BOT_NOT_FOUND` / `COMMAND_NOT_FOUND` | Bot 或 command 不存在 | 否 |
| 404 | `STATEFUL_SESSION_NOT_FOUND` | 会话不存在 | 否 |
| 409 | `BOT_COMMAND_DISABLED` | Catalog 中 command 已禁用 | 否 |
| 409 | `COMMAND_NAME_CONFLICT` | 全站 slash 名称冲突 | 否 |
| 409 | `BOT_EFFECT_CONFLICT` | 幂等键冲突 | 否 |
| 409 | `STATEFUL_SESSION_BUSY` | 频道已有活跃 stateful 会话 | 否 |
| 409 | `OFFICIAL_COMMAND_AUTO_ALLOWED` | Official command 无需显式 allow | 否 |
| 409 | `UNSUPPORTED_CHANNEL_KIND` | 不支持的频道类型（如 DM） | 否 |
| 409 | `IDEMPOTENCY_CONFLICT` | 同幂等键不同 body | 否 |
| 422 | `INVALID_COMMAND_OPTIONS` / `COMMAND_OPTIONS_INVALID` | 参数校验失败 | 否 |
| 422 | `BOT_EFFECT_INVALID` | Effect 校验失败 | 否 |
| 410 | `STATEFUL_SESSION_EXPIRED` | 会话已过期 | 否 |
| 429 | `STATEFUL_INPUT_BACKLOG_OVERFLOW` | 输入积压超限 | 否 |
| 503 | `BOT_OFFLINE` | Bot 未连接 Gateway | **是** |
| 503 | `CHAT_WORKER_UNAVAILABLE` | Worker 暂不可用 | **是** |

Bot Gateway 帧内错误出现在 `delivery_ack.error` 或 `delivery_result` 的 `status: "failed"` 中，code 语义与上表一致。

<a id="idempotency"></a>
## 12. 幂等与重试

| 键 | 语义 |
| --- | --- |
| HTTP `Idempotency-Key` | 同一 key + 不同 body → `409 IDEMPOTENCY_CONFLICT` |
| `delivery_id` | Bot 侧 delivery 去重 |
| `(channel_id, bot_id, client_effect_id)` | Server 侧 effect 幂等 |
| `invocation_id` / `interaction_id` | 用户侧 invoke 幂等（Bot 只读 delivery 中的对应字段） |

`BOT_OFFLINE` 与 `CHAT_WORKER_UNAVAILABLE` 可重试。Bot 重连时应携带 `last_received_delivery_id`。Server 可能对未完成 ack 的 delivery 重发。

<a id="security"></a>
## 13. 安全要求

- Bot Token 明文只出现一次；存储于密钥管理系统，不得写入客户端或公开仓库
- Bot 进程只需 **出站** 连接 `chat.kuma.homes`，无需开放入站 HTTP
- 按最小权限签发 scope；生产与开发环境使用不同 token
- 日志中脱敏 token 与用户信息
- 不信任 `invoker` / `actor` 以外的隐式权限；业务鉴权由 Bot 自行实现

<a id="checklist"></a>
## 14. 接入清单

- [ ] 从 Bot 所有者处取得 Chat Bot Token（`chat:runtime:connect`、`chat:commands:manage`、`chat:messages:write`）
- [ ] 实现 `PUT /api/chat/bot/commands` 启动同步
- [ ] 实现 Bot Gateway：hello → ready → delivery 循环 → delivery_result
- [ ] 实现 stateful：`session.start_ack` / `session.input_ack` / `session.effects` / `session.close`
- [ ] 请频道管理员在 ToolBear 中为该频道 allow 目标 command
- [ ] 处理 `delivery_id` 去重与 `client_effect_id` 幂等
- [ ] 实现断线重连与 `BOT_OFFLINE` 重试策略
