# Lilium Chat Bot API 接入规范 v1

适用对象：在 ToolBear Chat 上开发外置 Bot 的第三方开发者
文档状态：草案（对齐 `lilium-chat` 仓库 `docs/api-contract.md` v2.30；权威以 `lilium-chat` 仓库的 `docs/api-contract.md` 为准）

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
| Bot Stream WebSocket | `wss://chat.kuma.homes/api/chat/bot/channels/{channel_id}/streams/{message_id}/ws` |
| Bot 协议版本（Gateway） | `lilium.chat.bot.v1` |
| Bot 协议版本（Stream） | `lilium.chat.bot.stream.v1` |

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
| `GET` | `/api/chat/bot/channels/{channel_id}/streams/{message_id}/ws` | `chat:runtime:connect` + `chat:messages:write` | 单条 streaming message 的 Stream WS（append/finalize，§10.5） |
| `POST` | `/api/chat/bot/channels/{channel_id}/uploads/images/presign` | `chat:messages:write` | Bot 图片上传 presign（§10.6） |
| `POST` | `/api/chat/bot/channels/{channel_id}/uploads/images/{attachment_id}/finalize` | `chat:messages:write` | Bot 图片上传 finalize（§10.6） |

Bot 发消息、改消息、流式输出、components **只**通过 Gateway WS 的 `delivery_result.effects` 或 `session.effects`（见 §7、§10）。Bot Token 的 HTTP 面含 `PUT /api/chat/bot/commands`（catalog 同步）与 `POST /api/chat/bot/channels/{channel_id}/uploads/images/*`（附件上传，§10.6），**无**消息 mutation HTTP 端点。流式追加与 finalize 走专用 Stream WS（§10.5），**不**经主 Gateway effect。

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
| `chat:runtime:connect` | 连接 Bot Gateway WebSocket（主 Gateway 与 Stream WS） |
| `chat:commands:manage` | `PUT /api/chat/bot/commands` |
| `chat:messages:write` | `delivery_result` / `session.effects` 中的发消息类 effect，以及 Bot 附件上传 |

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

Stateful command 的 `listen_capability` 匹配时推送频道内新消息。`event` 含 `event_id` / `type` / `occurred_at`；`message` 为完整 Browser-visible Message 投影；`sender` 为 `user` 或 `bot` 形状。Bot 通常以 observer/responder only 回空 `effects: []`。被动消息投递由 stateful command 的 `listen_capability`（§6 catalog + §8 `session.input`）承担。

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

**Official bot auto-allow**：`visibility: "official"` 的 Bot 其 catalog 中 `status=active` 的命令在**所有非 DM 频道**自动可用，无需 allow binding；频道管理员只能 `blocked` 显式禁用，对 official command 发 `status: "allowed"` 返回 `409 OFFICIAL_COMMAND_AUTO_ALLOWED`。非 official Bot 仍须 allow binding 后才出现在 manifest。第三方接入通常 `visibility=private`，依赖频道 allow。

### 9.2 平台命令 `/help` 与 `/permission`

平台内置 `bot_id` `00000000-0000-7000-8000-000000000600` 提供 `/help` 与 `/permission`，响应以 `format=unsafe-markdown` 同步写入 Bot 消息，**不**经 Bot Gateway delivery。命令名使用 slash command chip 虚拟链接（`` [`/name`](/command:name) ``）。

- `/help`：无 `options.command` 时列出当前 caller manifest 全部命令（按 `bot.display_name` 分组，含 `/help`、owner/admin 可见 `/permission`）；有 `options.command` 时返回该命令的 `help_text`（fallback `description`）。
- `/permission`：内部平台命令，owner/admin 在非 DM manifest 可见；`/permission` 列出当前允许/阻止的命令，`/permission <name> on|off` 写 binding rows。对 official command `on` → `409 OFFICIAL_COMMAND_AUTO_ALLOWED`。

第三方 Bot 的 `help_text` 通过 catalog 同步供 `/help <command>` 查询。

<a id="effects"></a>
## 10. Effects 与消息 Components

### 10.1 Effect 类型与双 WebSocket

Bot 消息 mutation（发消息、改消息、流式、components）分两条 WebSocket 路径：

| 路径 | 允许的 mutation |
| --- | --- |
| 主 Bot Gateway WS（§7） | `send_message`、`update_message`、`disable_components`、`start_stream` |
| Stream WS（§10.5） | 单条 streaming message 的 `append`、`finalize` |

主 Bot Gateway WS 上，Bot 通过 `delivery_result.effects` 或 `session.effects` 改变频道状态。每个 effect 需要 `client_effect_id`（UUIDv7）。

| Effect | 说明 |
| --- | --- |
| `send_message` | 发送 **非 stream** Bot 消息（`stream_state=none`），可带 `components`（§10.3） |
| `update_message` | 更新 Bot 自己发送的消息文本、附件和 components（目标须 `stream_state=none`） |
| `disable_components` | 禁用 Bot 自己消息上的交互组件（目标须 `stream_state=none`） |
| `start_stream` | 创建 streaming registry + 返回 Stream WS URL（**不**写入 canonical `messages`；**禁止** `components`） |

主 Bot Gateway WS **拒绝** `append_stream` / `finalize_stream`（→ `422 BOT_EFFECT_INVALID`，recovery hint 可含 `stream.ws_url`）。流式正文的 `append` / `finalize` 必须在专用 Stream WS 上完成（§10.5）。

幂等键：`(channel_id, bot_id, client_effect_id)`。同 `client_effect_id` 不同 body → `409 BOT_EFFECT_CONFLICT`。

`update_message` / `disable_components` 只能操作**本 Bot 创建**、且 `stream_state=none` 的消息。

`send_message` / `start_stream` 的 `message.format` 允许 `plain` | `markdown` | `unsafe-markdown`；`unsafe-markdown` 仅 **official bot** 可用（BotConnection 在 `delivery_result` 时拦截非 official bot，返回 `delivery_ack{failed, BOT_EFFECT_INVALID}`）。非法 format → `422 BOT_EFFECT_INVALID`。`update_message` 不改 `format`（仅 text / components）。Bot Markdown 渲染规则与安全策略见 **§10.4**。

`delivery_ack` 对成功应用的 effect 可携带 `effect_results[]`。`start_stream` **必须**返回 `message_id` 与 Stream WS URL：

```json
{
  "type": "delivery_ack",
  "api_version": "lilium.chat.bot.v1",
  "delivery_id": "01J...",
  "status": "applied",
  "effect_results": [
    {
      "client_effect_id": "00000000-0000-7000-8000-000000000910",
      "type": "start_stream",
      "status": "applied",
      "message_id": "00000000-0000-7000-8000-000000000301",
      "stream": {
        "channel_id": "00000000-0000-7000-8000-000000000201",
        "message_id": "00000000-0000-7000-8000-000000000301",
        "ws_url": "/api/chat/bot/channels/00000000-0000-7000-8000-000000000201/streams/00000000-0000-7000-8000-000000000301/ws",
        "expires_at": "2026-06-30T12:00:00Z"
      }
    }
  ]
}
```

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

`display_name` / `avatar_url` 来自 BotRegistry，不查 ToolBear 用户表。普通用户消息 `sender.kind="user"`，结构与 Browser API 一致。

### 10.3 交互 Components

Bot 消息可携带 rich interactive UI 组件，由 Bot 生成、Worker 校验后随消息持久化。**仅**非 stream 的 Bot 消息（`send_message` / `update_message`，`stream_state=none`）允许 `components`；stream 消息（`streaming` | `final` | `abandoned`）投影 `components=[]`。普通用户消息**不能**带 `components`。

组件 `kind`：`button` | `select` | `radio` | `checkbox` | `checkbox_group` | `text_input`。

公共字段（所有 kind）：

| 字段 | 说明 |
| --- | --- |
| `component_id` | UUIDv7；组件稳定 id |
| `kind` | 见枚举 |
| `custom_id` | Bot 私有 payload；前端只原样回传，不解析 |
| `disabled` | `true` 时不可提交 |
| `interaction_policy` | 可选；缺省 `multi` |
| `target_user_id` | `interaction_policy=targeted` 时必填；仅该用户可提交 |

`interaction_policy`（Worker 在 `interaction.submit` 事务内原子执行）：

| policy | 平台保证 |
| --- | --- |
| `multi`（默认） | 频道内任意可见成员可提交；同一 `(user, command_id)` 幂等 |
| `per_user_once` | 同一 `(message_id, component_id, actor_user_id)` 只允许一条成功 interaction；重复 → `409 INTERACTION_ALREADY_SUBMITTED` |
| `exclusive` | 全频道该 component 只允许一条成功 interaction；首个成功 submit **同事务**标 `disabled=true`；后续 → `409 COMPONENT_ALREADY_USED` |
| `targeted` | 仅 `target_user_id` 可提交；他人 → `403 INTERACTION_FORBIDDEN_TARGET` |

平台负责 **结构性门禁**（谁能点、能点几次、是否一次性锁死）；**业务冲突**由 Bot 在 `delivery_result.effects` 中自行裁决。Bot 可用 `disable_components` 做 UI 反馈，但 **不能**替代 `exclusive`（后者已在 submit 事务内锁死，无竞态窗口）。

`interaction.submit` 的 `payload.value` 类型按 component `kind` 校验：

| `kind` | `value` 类型 | Worker 额外校验 |
| --- | --- | --- |
| `button` | `boolean`（恒 `true`） | — |
| `select` / `radio` | `string` | 必须命中 `options[].value` |
| `checkbox` | `boolean` | — |
| `checkbox_group` | `string[]` | 每项命中 `options[].value`；长度 ∈ `[min_selected, max_selected]` |
| `text_input` | `string` | 长度 ∈ `[min_length, max_length]` |

Button 示例：

```json
{
  "component_id": "00000000-0000-7000-8000-000000000a01",
  "kind": "button",
  "style": "primary",
  "label": "确认",
  "custom_id": "confirm",
  "disabled": false,
  "interaction_policy": "exclusive"
}
```

用户操作组件后产生 `message_interaction` delivery（§7.4）。

### 10.4 Bot Markdown 与安全渲染

`message.format` 决定 Browser 如何渲染 Bot 消息正文：

| `format` | 谁可以写入 | 校验 |
| --- | --- | --- |
| `plain` | 任意 Bot；用户消息固定为此值 | 无额外限制 |
| `markdown` | 任意已安装 Bot（`send_message` / `start_stream`） | 非法值 → `422 BOT_EFFECT_INVALID` |
| `unsafe-markdown` | 仅 **official bot** 与 platform bot 服务端直写 | 非 official bot → `delivery_ack{failed, BOT_EFFECT_INVALID}`，**不**转发 ChatChannel |

第三方 Bot **只能**使用 `plain` 或 `markdown`；**不得**请求 `unsafe-markdown`。Browser 用 **markdown-it + DOMPurify** 渲染（净化后 `v-html` 输出）。

- `format=markdown`：strict inline HTML 白名单；站外 `http:`/`https:` 链接**不可点击**（渲染为 `<span>`）；`javascript:`/`data:` 等危险协议阻断。
- `format=unsafe-markdown`：放宽为更宽的 HTML 白名单，站外链接可点击。
- 两种 format 均剔除 `script`/`style`/`iframe`/`form` 等危险标签与所有 `on*` 事件属性。
- streaming 中间态按**纯文本**追加（不解析 Markdown）；`stream_state=final` 且 format 为 markdown 类时一次性完整渲染。
- Slash command chip 虚拟链接 `` [`/name`](/command:name) `` 渲染为可点击 chip，打开命令帮助，不离开页面。

详细渲染规则（标签/属性白名单、链接规则、图片规则）见 `lilium-chat` 仓库 `docs/api-contract.md` §3.9。

### 10.5 流式输出：Stream WebSocket

`start_stream` effect 在主 Gateway 上创建 streaming registry 并返回 `ws_url`，**不**写入 canonical `messages`。Bot 必须在 `expires_at` 前连接专用 Stream WS 完成追加与 finalize。

```http
GET /api/chat/bot/channels/{channel_id}/streams/{message_id}/ws
Authorization: Bearer <bot_token>
Sec-WebSocket-Protocol: lilium.chat.bot.stream.v1
```

`{channel_id, message_id}` **成对**出现在 path；主 Gateway 与 Stream WS 使用**同一份** Chat Bot Token，无独立 stream token。Stream WS 一连接服务于单条 streaming message。

帧协议（`api_version` = `lilium.chat.bot.stream.v1`）：

| Direction | Type | Payload |
| --- | --- | --- |
| Bot → Server | `hello` | `{}` |
| Server → Bot | `ready` | `{ channel_id, message_id, expires_at, ack_seq }` |
| Bot → Server | `append` | `{ seq, delta }` |
| Server → Bot | `append_ack` | `{ ack_seq }` |
| Bot → Server | `finalize` | `{ final_seq, attachment_ids? }`（**禁止** `components`） |
| Server → Bot | `finalized_ack` | `{ ok: true, message_id, event_id }` |
| Server → Bot | `stream_error` | `{ code, message, retryable }` |
| Both | `ping` / `pong` | `{}` |

Sequence 规则：

- `seq` 从 `1` 起，严格递增 1。
- `ack_seq` 是 server **已 durable flush** 的最高 seq；重连后从 `ack_seq + 1` 重发。
- `received_seq` 是当前活动连接已接受（未必 durable ack）的最高 seq；gap 与 unacked duplicate 基于 `received_seq`，**不是** `ack_seq`。
- `seq <= ack_seq`：durable no-op；`seq == received_seq + 1`：接受；`seq > received_seq + 1`：`BOT_STREAM_SEQUENCE_GAP`。
- unacked duplicate 判重仅在活动 WS 内存 Map，不持久化；rehydrate 后 `received_seq` 回落到 `ack_seq`。

Finalize 规则：

- `final_seq` 必须 `== received_seq`（`>` gap / `<` conflict）；finalize 前先 flush，使 `ack_seq == received_seq`。
- finalize **禁止**非空 `components`，attachment_ids 当前亦**禁止**。正文仅为 `text`（`format=plain` | `markdown` | `unsafe-markdown`）。
- `finalize_request_hash` = 稳定 hash over 规范化 JSON `{ final_seq, resolved_text, components: [], attachment_ids }`；幂等判断用此 hash。已 finalized 且相同 hash → 返回 stored response；不同 → `BOT_STREAM_CONFLICT`。

中断与 abandon：

- Stream WS / 主 Gateway 断连 **不** finalize；Bot 可在 `expires_at` 前重连。
- Expiry / abandon 时非空 durable partial 写入 canonical **abandoned** message（`stream_state=abandoned`、`status=failed`）并 emit `message.stream_abandoned`；空 buffer 仅发 live-only `message.stream_abandon_cleanup`，不写 history。
- 浏览器对 stream 中间态（`stream_started` / `stream_delta` / `stream_abandon_cleanup`）用 live-only frame，**不**算入 HTTP recovery cursor；`message.stream_finalized` 与 `message.stream_abandoned` 才是 canonical channel event，HTTP history/events 为权威。

### 10.6 Bot 附件上传

Bot 经 HTTP 上传图片附件，供非 stream effect（`send_message` / `update_message`，`stream_state=none`）引用 `attachment_ids`（`type: "image"`）。Stream 路径仍 **禁止** `attachment_ids`。

```http
POST /api/chat/bot/channels/{channel_id}/uploads/images/presign
Authorization: Bearer <bot_token>
Idempotency-Key: <uuidv7>
Content-Type: application/json
```

请求体：

```json
{
  "filename": "photo.png",
  "mime_type": "image/png",
  "size_bytes": 12345,
  "width": 512,
  "height": 512,
  "blurhash": "LKO2?U%2Tw=w]~RBVZRi};RPxuwH"
}
```

响应：`{ attachment_id, upload_url, upload_headers, expires_at }`（presigned PUT 语义同用户上传：浏览器直传 SeaweedFS，5 分钟过期）。

```http
POST /api/chat/bot/channels/{channel_id}/uploads/images/{attachment_id}/finalize
Authorization: Bearer <bot_token>
Content-Type: application/json
```

请求 `{ "etag": "\"abc123\"" }`；响应 `{ "attachment": { attachment_id, url, ... } }`。

规则：

- Required scope `chat:messages:write`；Bot 须 **已安装** 于 `{channel_id}`。
- Attachment 仅可在 presign 同一 `channel_id` 的非 stream effect 中引用；跨 channel 或 user-owned attachment → `BOT_EFFECT_INVALID`。
- 未 finalize 的 pending 行由 ChatChannel alarm GC（删 S3 对象 + SQLite 行）。
- 消息 mutation 仍只经 WS effects；HTTP 上传仅产生可引用的 `attachment_id`。

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
| 409 | `COMPONENT_ALREADY_USED` | `interaction_policy=exclusive` 且该 component 已被他人成功提交 | 否 |
| 409 | `INTERACTION_ALREADY_SUBMITTED` | `interaction_policy=per_user_once` 且该用户已提交过 | 否 |
| 403 | `INTERACTION_FORBIDDEN_TARGET` | `interaction_policy=targeted` 且提交者不是 `target_user_id` | 否 |
| 404 | `COMPONENT_NOT_FOUND` / `COMPONENT_DISABLED` | 组件不存在/不可见或已禁用 | 否 |
| 422 | `INVALID_INTERACTION_VALUE` | interaction value 形状与 `kind` 不匹配 | 否 |
| 409 | `BOT_STREAM_CONFLICT` | 同一 pending stream seq 复用但 delta 不同 | 否 |
| 409 | `BOT_STREAM_SEQUENCE_GAP` | append seq 跳过 `received_seq + 1`（同连接内），可重试 | **是** |
| 410 | `BOT_STREAM_EXPIRED` | stream 在 finalize 前过期 | 否 |
| 410 | `STATEFUL_SESSION_EXPIRED` | 会话已过期 | 否 |
| 429 | `STATEFUL_INPUT_BACKLOG_OVERFLOW` | 输入积压超限 | 否 |
| 503 | `BOT_OFFLINE` | Bot 未连接 Gateway | **是** |
| 503 | `CHAT_WORKER_UNAVAILABLE` | Worker 暂不可用 | **是** |

`BOT_OFFLINE`、`CHAT_WORKER_UNAVAILABLE`、`BOT_STREAM_SEQUENCE_GAP` 可重试。

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
- [ ] 实现流式：主 Gateway `start_stream` effect → 连 Stream WS → `append` / `finalize`
- [ ] （如需图片附件）实现 Bot 附件上传 presign/finalize
- [ ] Rich UI 组件按 `interaction_policy` 处理，回 `delivery_result.effects` 表达交互结果
- [ ] 请频道管理员在 ToolBear 中为该频道 allow 目标 command
- [ ] 处理 `delivery_id` 去重与 `client_effect_id` 幂等
- [ ] 实现断线重连与 `BOT_OFFLINE` 重试策略
