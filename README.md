# Codex 便携工作站

这是一个面向 Windows 10/11 x64 的 U 盘/移动固态便携 AI 编程工作站项目。

项目不会重新开发 Codex 桌面界面。正式产品计划基于 Codex++ 做受控二次开发，复用其供应商配置、协议转换、会话管理、插件市场、页面增强、Zed Remote 和 Codex 启动能力，再补充 U 盘路径、跨电脑数据、服务器授权、应用级连接、统一更新、安全退出与恢复机制。

> 当前阶段：上游审计、架构设计和 UI 原型。尚未生成可在 Windows 上运行的正式 EXE。

## 当前已有内容

- `prototype/codex-launcher-ui/`：可在浏览器中查看的产品 UI 原型。
- `third_party/upstreams/`：本地浅克隆的上游源码，仅用于审计，不提交到本仓库。
- `PROJECT_SPEC.md`：整理后的完整产品需求。
- `docs/UPSTREAM_AUDIT.md`：Codex++、Sub2API、便携 Codex 和 OpenAI Codex 源码审计。
- `docs/FEATURE_INTEGRATION_MATRIX.md`：每个 UI 页面和按钮应对接的真实功能。
- `docs/ARCHITECTURE.md`：正式软件总体架构。
- `docs/ENGINEERING_DIRECTORY.md`：正式工程目录设计。
- `docs/CODEXPLUSPLUS_INTEGRATION.md`：Codex++ Fork 和改造边界。
- `docs/SUB2API_INTEGRATION.md`：服务器、仪表盘和设备授权对接方案。
- `docs/NETWORK_AND_SERVER_CONNECTIVITY.md`：仅服务于本工作站的合规连接方案。
- `docs/UPDATE_PROTOCOL.md`：自己的更新仓库、签名、兼容矩阵和回滚协议。
- `docs/SECURITY_MODEL.md`：凭据、日志、WebView、更新和服务端安全模型。
- `docs/LICENSE_REVIEW.md`：AGPL、LGPL、第三方载荷和商标风险。
- `docs/DELIVERY_PLAN.md`：分阶段开发计划。
- `docs/DEVELOPMENT_READINESS.md`：Mac、GitHub 和 Windows 的最终开发前预检与 Go/No-Go 结论。
- `docs/PRODUCT_DECISIONS.md`：用户已确认的产品、授权、卡密、数据和更新决策。
- `docs/plans/2026-07-16-local-networked-usb-launcher-development-plan.md`：先完成可在 U 盘运行、正常联网但暂不连接两套业务服务器的启动器，再接服务器的执行方案。

## 查看 UI 原型

在当前目录运行：

```bash
cd prototype/codex-launcher-ui
python3 -m http.server 8787
```

浏览器打开：

```text
http://127.0.0.1:8787/
```

## 已确定的核心路线

1. 正式客户端优先从 Codex++ Fork 演进，继续使用 Rust + Tauri + React。
2. 启动器从自身 EXE 路径推导移动盘根目录，不写死盘符。
3. `CODEX_HOME`、`CODEX_SQLITE_HOME`、Codex++ 状态、项目、插件和恢复点尽量全部落在移动存储中。
4. 客户端不直接从社区仓库执行 `git pull`，只检查自己的签名稳定更新清单。
5. Sub2API 独立部署在服务器；无账号卡密激活、移动工作站绑定和服务 entitlement 由独立 `activation-bridge` 补齐。
6. 所谓“网络环境”仅允许实现为本工作站专用、目标域名白名单化的服务连接，不提供通用 VPN 或任意流量转发。

## 当前最重要的发布约束

- Codex++ 是 `AGPL-3.0-only`。用户已接受对应源码义务；每个客户端发行版必须关联准确源码和构建说明。
- 用户已接受向接收者提供 Codex++ 对应修改源码，正式客户端可以继续采用 Codex++ Fork；每个发行版必须提供完全对应的源码、构建说明、许可证和第三方声明。
- 便携 Codex 上游仓库未声明清晰的仓库级许可证，桌面应用载荷的再分发权必须单独确认。
- Sub2API 使用 LGPL，但 README 同时写有“无商业授权”和上游服务条款风险，商业上线前必须进行独立法律与上游条款评估。
- “从同一 U 盘换电脑继续工作”可以覆盖项目、会话、Skills、插件和本地配置，但第三方 OAuth、系统钥匙串和官方云端状态不能承诺完全无感迁移。

详细结论见 `docs/`。
