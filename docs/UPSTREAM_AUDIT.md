# 上游源码审计

审计时间：2026-07-16
审计方式：本地浅克隆、源码检索、GitHub Release 元数据和官方 Codex 仓库交叉核对。

## 1. 结论摘要

Codex++ 已经覆盖本产品约一半以上的“Codex 管理与增强”能力，最值得直接复用的是：

- 自定义 Codex 应用路径和 `--app-path` 启动。
- CDP 启动、辅助服务、注入和运行状态。
- 供应商保存、切换、测试、模型列表和 Provider Doctor。
- Responses 与 Chat Completions 的本地协议转换。
- 会话扫描、删除、Markdown 导出、Provider 同步和项目移动。
- 插件市场、用户脚本、页面增强、Zed Remote 和 upstream worktree。
- 应用检测、日志、环境冲突和部分健康检查。

不能原样复用的关键部分：

- Codex++ 自身状态固定写到用户主目录和 `%APPDATA%`，没有 `CODEXPP_HOME`。
- 上游供应商实现会把 API Key 保存在 `settings.json`、`config.toml` 或 `auth.json`。
- 上游更新器直接读取 Codex++ GitHub 最新版本并启动安装程序，没有多组件兼容矩阵、签名验证、数据隔离和自动回滚。
- 上游安装/Watcher 会创建系统快捷方式或启动项，不符合默认纯便携模式。
- Codex++ 的 `protocol_proxy` 是协议转换服务，不是远程网络通道，也不能当作“机场节点”或 VPN。

因此建议维护一个范围受控的 Codex++ Fork，并将当前 UI 原型迁入其管理工具。

## 2. Codex++ 审计

### 2.1 版本与结构

- 仓库：https://github.com/BigPizzaV3/CodexPlusPlus
- 审计 commit：`2924333c6770497470090235f484519154698651`
- Release：`v1.2.37`
- 技术栈：Rust 2024、Tauri 2、React 19、TypeScript、Vite。
- 许可证：`AGPL-3.0-only`。

主要目录：

```text
apps/codex-plus-launcher/          静默启动入口
apps/codex-plus-manager/           React + Tauri 管理工具
crates/codex-plus-core/            启动、供应商、协议、更新、插件等核心
crates/codex-plus-data/            会话、SQLite、备份和 Provider 同步
assets/inject/                     Codex 页面增强脚本
```

### 2.2 U 盘启动可复用点

`apps/codex-plus-launcher/src/main.rs` 已解析：

```text
--app-path
--debug-port
--helper-port
```

`crates/codex-plus-core/src/app_paths.rs` 已支持：

- `Codex.exe`、`ChatGPT.exe`。
- 输入 EXE 文件、`app` 目录或包含 `app` 的便携包根目录。
- 保存自定义 Codex 路径。
- 从便携目录名或 `version` 文件读取版本。

`crates/codex-plus-core/src/launcher.rs` 在非 MSIX 路径下会直接使用 `Command::new(executable)` 启动，因此移动盘中的便携 EXE 可以走现有进程启动链。

结论：不需要重新开发一套 Codex CDP 启动器。正式工作站只需从自身路径计算：

```text
<USB_ROOT>\App\Codex\app
```

然后传给 Codex++ `--app-path`。

### 2.3 Codex 数据目录

`crates/codex-plus-core/src/codex_home.rs` 已读取 `CODEX_HOME`。要求该目录已经存在，否则会回退到用户目录 `~/.codex`。

这意味着工作站必须先创建：

```text
<USB_ROOT>\Data\Codex\.codex
```

再对子进程设置：

```text
CODEX_HOME=<USB_ROOT>\Data\Codex\.codex
CODEX_SQLITE_HOME=<USB_ROOT>\Data\Codex\.codex\sqlite
```

Codex++ 的 SQLite 检索目前仍从 `CODEX_HOME/sqlite` 推导，未单独读取 `CODEX_SQLITE_HOME`。如果 SQLite 目录与 `CODEX_HOME/sqlite` 保持一致，不影响使用；若未来要拆开，Codex++ 也需适配。

### 2.4 Codex++ 自身数据缺口

`crates/codex-plus-core/src/paths.rs` 将以下文件固定放在用户主目录：

```text
~/.codex-session-delete/settings.json
~/.codex-session-delete/latest-status.json
~/.codex-session-delete/codex-plus.log
~/.codex-session-delete/pending-provider-import.json
```

用户脚本配置还会写入：

```text
%APPDATA%\Codex++
```

当前源码中没有 `CODEXPP_HOME`。因此换电脑后，Codex++ 自己的供应商、页面增强开关、用户脚本和状态不会自动跟随 U 盘。

必须改造：

```text
CODEXPP_HOME=<USB_ROOT>\Data\CodexPlusPlus
```

所有 Codex++ 设置、日志、状态、待导入配置、脚本和更新暂存均优先从此目录派生。

### 2.5 供应商与协议能力

可直接复用的 Tauri Commands：

- `load_settings`
- `save_settings`
- `load_ccs_providers`
- `import_ccs_providers`
- `switch_relay_profile`
- `test_relay_profile`
- `diagnose_relay_profile`
- `fetch_relay_profile_models`
- `relay_status`
- `check_relay_environment`
- `apply_relay_injection`
- `apply_pure_api_injection`
- `clear_relay_injection`

`protocol_proxy.rs` 已实现 Responses 与 Chat Completions 的请求、响应和 SSE 转换，可复用到用户自己的 Chat Completions API。

### 2.6 API Key 安全缺口

当前 Codex++：

- `RelayProfile.api_key` 会被序列化到 `settings.json`。
- `provider_import.rs` 会生成带 `experimental_bearer_token` 的 `config.toml`。
- 纯 API 模式会生成包含 `OPENAI_API_KEY` 的 `auth.json`。
- 诊断报告当前会序列化完整 settings。

这与本产品“API Key 不进入明文配置、日志和诊断包”的要求冲突。

OpenAI Codex 当前官方源码支持 provider：

```toml
[model_providers.workstation]
base_url = "https://api.example.com/v1"
env_key = "CODEX_WORKSTATION_API_KEY"
wire_api = "responses"
requires_openai_auth = false
```

因此 Fork 应新增“凭据引用模式”：

- Codex++ 配置只保存 `env_key` 名称。
- 实际 Key 保存在工作站加密凭据库。
- 启动 Codex++ 时由父进程注入环境变量。
- 供应商测试同样从临时环境变量读取。
- 诊断包移除所有 secret 字段。

### 2.7 停止与退出

Codex++ 管理工具没有直接暴露“停止 Codex”命令，但 `restart_codex_plus` 已内部调用：

```text
stop_launcher_processes_and_wait()
stop_codex_processes_and_wait()
```

因此可以新增一个很薄的 `stop_codex_plus` Tauri Command，复用现有进程停止和等待能力。工作站还需在其后检查 SQLite WAL/SHM 与移动盘文件句柄。

### 2.8 上游更新器缺口

`crates/codex-plus-core/src/update.rs` 当前：

- 固定读取 Codex++ GitHub 的 `latest.json`。
- 选择当前平台安装包。
- 下载文件。
- 启动安装程序。

它没有：

- 产品自己的更新源。
- 多组件兼容矩阵。
- SHA-256/Ed25519 强制验证。
- 原子目录替换。
- 用户数据保护规则。
- 健康检查和自动回滚。

正式工作站必须禁用 Codex++ 原更新入口，统一使用工作站更新器。

## 3. Sub2API 审计

### 3.1 版本与结构

- 仓库：https://github.com/Wei-Shaw/sub2api
- 审计 commit：`09c6c6d74050cf49ed2fb864be6c11647798ef53`
- 最新 Release：`v0.1.156`
- 技术栈：Go、Gin、Vue、PostgreSQL、Redis、Docker。
- 许可证：LGPL-3.0-or-later。

### 3.2 可直接使用的用户能力

用户认证后已有：

- `/api/v1/user/profile`
- `/api/v1/keys`
- `/api/v1/groups/available`
- `/api/v1/channels/available`
- `/api/v1/usage/dashboard/stats`
- `/api/v1/usage/dashboard/trend`
- `/api/v1/usage/dashboard/models`
- `/api/v1/usage/dashboard/snapshot-v2`
- `/api/v1/subscriptions`
- `/api/v1/subscriptions/active`
- `/api/v1/subscriptions/progress`
- `/api/v1/subscriptions/summary`
- `/api/v1/redeem`
- `/api/v1/payment/plans`
- `/api/v1/payment/orders`

模型网关已有：

- `/v1/models`
- `/v1/responses`
- `/v1/chat/completions`
- `/v1/messages`
- `/v1/sub2api/billing`
- `/v1/usage`

首页仪表盘可直接复用用户 usage dashboard 接口，不应虚构 `/api/codex-plus/overview`。

### 3.3 缺失能力

源码中没有用于“无账号卡密 + 移动工作站绑定”的产品级授权流程。现有 `device_code` 只属于部分上游 OAuth 实现，不能拿来给本产品客户端授权。

缺少：

- 无账号工作站卡密激活。
- 以移动存储为单位的 `installation_id` 和公钥绑定。
- 卡密首次激活计时和多工作站绑定上限。
- 签名 entitlement、短期服务令牌、续签和撤销。
- 工作站服务连接卡密与中转站套餐权益映射。

因此需要独立 `activation-bridge`。

### 3.4 管理接口边界

Sub2API 管理接口已具备用户、API Key、订阅、卡密、用量、监控和后台能力，但管理接口只能由服务器端桥接服务调用。

客户端不得直接调用：

```text
/api/v1/admin/*
```

### 3.5 商业风险

Sub2API LICENSE 为 LGPL，但 README 明确写有：

- 上游服务条款风险。
- 仅供技术学习与研究。
- “无商业授权”声明。

技术上能够部署不代表商业模式已经获得授权。正式销售前需要独立法律意见和上游服务条款核对。

## 4. 便携 Codex 上游审计

- 仓库：https://github.com/lusipad/unofficial-codex-app-offline
- 审计 commit：`8b39247bd569b98edcf6501a3ec3c2b2d532a604`
- Release：`offline-v26.707.9981.0`

它提供：

- 从 Microsoft Store 包提取 x64 载荷。
- 对 `app.asar` 做便携兼容补丁。
- 生成 portable ZIP、Setup、Web 包和校验文件。
- 包结构中的 `_internal/app` 可作为工作站 `App/Codex/app` 来源。

风险：

- GitHub API 未识别仓库级许可证。
- 生成的桌面载荷包含第三方应用资源。
- 补丁可能随 Store 包结构变化失效。
- 不应在许可结论前公开或商业分发构建产物。

## 5. OpenAI Codex 官方源码核对

- 仓库：https://github.com/openai/codex
- 审计 commit：`cbc83d961e8132bfff4d340ab8342d181b79e95e`
- 最新 Release：`rust-v0.144.5`
- 许可证：Apache-2.0。

已核实：

- `CODEX_HOME` 是正式支持的状态根目录。
- `CODEX_SQLITE_HOME` 是正式支持的 SQLite 目录环境变量。
- 自定义 provider 支持 `env_key`。
- Skills、插件、主题、认证回退文件、日志和大量本地状态位于 `CODEX_HOME`。

注意：该仓库主要是 Codex CLI/app-server 源码，不等同于 Windows 桌面应用全部源码。Apache-2.0 不自动覆盖桌面应用资源。

## 6. 最终复用判断

| 能力 | 结论 |
|---|---|
| 从 U 盘路径启动 Codex | Codex++ 已支持，直接复用 `--app-path` |
| `CODEX_HOME` 便携化 | Codex++ 和官方 Codex 已支持 |
| `CODEX_SQLITE_HOME` | 官方 Codex 支持；Codex++ 检索需保持目录一致或补适配 |
| Codex++ 自身数据便携 | 上游没有，必须增加 `CODEXPP_HOME` |
| 供应商/协议转换 | Codex++ 已有，改造凭据存储后复用 |
| 会话与插件 | Codex++ 已有大部分，补工作站级索引与恢复点 |
| 停止 Codex | 核心已有，需暴露 UI Command 并补磁盘释放检查 |
| 工作站主仪表盘 | 复用 Codex++ Overview，并由 Workstation Health 聚合 |
| 中转站附加仪表盘 | Sub2API 已有用量接口，配置 API Key 后可选接入 |
| 无账号卡密绑定 | Sub2API 没有，新增 activation-bridge |
| 卡密 | activation-bridge 管理工作站卡密，可参考 Sub2API redeem code |
| 应用级连接 | 现有项目没有完整实现，新增受限 sidecar/边缘网关 |
| 工作站统一更新 | 上游更新器不满足，必须重新实现 |
| 安全弹出 U 盘 | 上游没有，必须新增 Windows 模块 |
