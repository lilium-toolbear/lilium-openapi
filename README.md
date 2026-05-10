# lilium-openapi

Lilium 面向第三方开发者的公开接口与接入文档仓库。

统一入口：

- `https://lilium.kuma.homes`

本仓库当前提供：

- 平台级认证与授权说明
- 清算 API 接入规范
- 钱包 API 接入规范
- Bot 外部命令接入规范
- Hosted Checkout 与 Webhook 说明
- OpenAPI 及其他公开契约材料

当前文档：

- [Lilium 平台认证与路由规范 v1.1](./docs/lilium-platform-authentication.md)
- [Lilium 开放清算 API 接入规范 v1.1](./docs/lilium-open-clearing-api-design.md)
- [Lilium 钱包 API 接入规范 v1](./docs/lilium-wallet-api-design.md)
- [Lilium Bot External Command 接入规范 v1](./docs/lilium-bot-external-command-api-design.md)

示例第三方接入仓库：

- [lilium-bank-demo](https://github.com/kuma-dzmm/lilium-bank-demo)

文档边界：

- 仅描述第三方需要知道的公开入口、认证方式、字段、状态与安全要求
- 不描述内部服务实现、部署结构、前端技术选型或其他非公开设计决策
- OpenAPI 与说明文档应保持一致
