# Third-Party Sources

审计时间：2026-07-16（Asia/Shanghai）

| 上游 | 本地审计提交 | 最新发布 | 许可状态 | 用途 |
|---|---|---|---|---|
| BigPizzaV3/CodexPlusPlus | `2924333c6770497470090235f484519154698651` | `v1.2.37`，2026-07-15 | AGPL-3.0-only | 正式客户端、Codex 启动、供应商、协议转换、会话、插件和增强基础 |
| Wei-Shaw/sub2api | `09c6c6d74050cf49ed2fb864be6c11647798ef53` | `v0.1.156`，2026-07-15 | LGPL-3.0-or-later；README 有商业与条款警告 | 用户、Key、套餐、支付、计费、用量、模型转发和后台 |
| lusipad/unofficial-codex-app-offline | `8b39247bd569b98edcf6501a3ec3c2b2d532a604` | `offline-v26.707.9981.0`，2026-07-15 | GitHub 未识别仓库许可证；载荷权利待核实 | Windows 便携 Codex 构建参考 |
| openai/codex | `cbc83d961e8132bfff4d340ab8342d181b79e95e` | `rust-v0.144.5`，2026-07-16 | Apache-2.0，仅覆盖该仓库源码 | 核实 `CODEX_HOME`、`CODEX_SQLITE_HOME`、provider 和 app-server 行为 |

## 来源地址

- https://github.com/BigPizzaV3/CodexPlusPlus
- https://github.com/Wei-Shaw/sub2api
- https://github.com/lusipad/unofficial-codex-app-offline
- https://github.com/openai/codex
- https://developers.openai.com/codex/app

## 使用规则

- 社区最新版本不等于本产品兼容版本。
- 发布时必须记录实际构建 commit、tag、哈希和补丁集。
- 上游同步仅发生在开发机和 CI。
- 未通过许可证、兼容性和安全检查的二进制不得进入稳定通道。
- OpenAI Codex CLI 仓库的 Apache-2.0 不自动授予桌面应用载荷、商标和图标的分发权。

