# Codex 便携工作站本地联网启动器完整开发方案

更新时间：2026-07-16

## 1. 本阶段目标

先完成一个可以放入 Windows 移动存储、暂不连接本产品两套业务服务器也能正常运行的“Codex 便携工作站”客户端。

本阶段完成后，用户应当能够：

```text
插入移动存储
→ 双击 Codex便携工作站.exe
→ 自动识别移动盘根目录
→ 检查运行环境和数据目录
→ 从移动盘启动便携 Codex
→ 使用 Codex++ 的供应商、会话、插件、Skills 和页面增强能力
→ 项目和可迁移状态全部写入移动盘
→ 停止当前便携 Codex 进程树
→ 等待数据落盘
→ 安全弹出移动存储
```

中转站数据、工作站卡密、服务连接和管理员后台在下一阶段实现。本阶段只定义稳定接口并提供“未接入”实现，但客户端其他联网能力正常工作。

## 2. “本地联网版”的准确含义

本方案不是断网版。当前只暂不连接用户自己的两套业务服务器：

1. 中转站网站和用量服务器。
2. 工作站卡密、服务连接与网络控制服务器。

当前版本正常允许联网访问：

- Codex/OpenAI 官方服务和登录。
- 用户自己填写的 OpenAI Compatible、DeepSeek 和其他第三方 API。
- GitHub、插件市场、远程插件和用户脚本市场。
- GitHub Release 更新源。
- SSH/Zed 等远程项目。
- 普通 HTTPS 网络诊断。

本产品两套服务器未接入时：

- 工作站客户端仍能启动和使用其他联网功能。
- 项目、会话、配置、Skills、插件和本地状态不依赖本产品服务器。
- 对应页面显示真实的“服务器尚未接入”状态。
- 不允许伪造在线、余额、套餐、卡密或服务连接成功。

完全断网仍然属于需要支持的降级状态：本地项目和界面可用，远程模型请求自然不可用，本地模型可以继续使用。

## 3. 已连接 Windows 开发机审计结果

远程开发机：

```text
主机：LAPTOP-892MM2QA
用户：James
地址：192.168.1.17:22
系统：Windows 11 家庭中文版 x64
系统 Build：22621.4317
开发目录：D:\Dev\codex-portable
```

已确认可用：

```text
Git 2.55.0
GitHub CLI 2.96.0
Node.js 24.18.0
npm 11.16.0
Rust/Cargo 1.97.0 stable-msvc
Python 3.12.10
PowerShell 7.6.3
.NET SDK 8/9
Visual Studio Build Tools 2022
VC Tools x64/x86
Windows SDK 10.0.22621 和 10.0.26100
WebView2 Runtime 150.0.4078.65
7-Zip 26.02
Inno Setup 6
MSBuild 17.14
Windows Long Paths
Git core.longpaths
```

当前状态：

- SSH 已使用 Mac 公钥连接成功。
- `D:\Dev\codex-portable` 已存在，但是空目录且不是 Git 仓库。
- Windows GitHub CLI 尚未登录。
- 当前只有 `C:` 和 `D:` 固定磁盘，尚未检测到 USB 移动盘。
- 官方 Microsoft Store Codex `26.623.11225.0` 已安装并正在运行。
- Windows 仍有第三方 `ZulerCoreTools` 产生的待重启文件重命名标记，但 Build Tools 本身不要求重启。
- CMake 当前未进入 PATH；Codex++ 当前依赖未证明必须使用 CMake，因此不先安装。

## 4. 开发与运行位置

### 4.0 个人电脑保护边界

这台 Windows 是用户日常使用的个人电脑，不需要也不允许为了开发清除其他内容。

- 不卸载或替换当前 Microsoft Store Codex。
- 不删除用户其他项目、文档、下载和桌面文件。
- 不清理 `%USERPROFILE%`、`%APPDATA%` 或系统临时目录中的无关内容。
- 不修改 Windows 全局代理、永久环境变量和自启动项。
- 不按进程名批量关闭全部 Codex。
- 所有源码和构建输出限制在 `D:\Dev\codex-portable`。
- 真实移动盘只有在确认盘符、目录和备份状态后才允许写入。
- 任何删除、格式化、分区、卸载和系统级配置操作必须单独获得用户确认。

### 4.1 不直接在 U 盘编译

源码编译和测试放在：

```text
D:\Dev\codex-portable
```

原因：

- Rust `target/` 和 `node_modules/` 会产生大量小文件。
- 直接在 U 盘编译速度慢、容易占用文件句柄，并增加闪存写入。
- 编译过程中无法可靠执行安全弹出测试。

### 4.2 使用模拟移动盘根目录

开发初期使用：

```text
D:\Dev\codex-portable\.test-usb\Codex-Workstation
```

路径逻辑稳定后，再部署到真实移动盘：

```text
<USB_ROOT>\Codex-Workstation
```

### 4.3 正式移动盘要求

- 推荐 64GB 或更大。
- 推荐 NTFS。
- 路径必须覆盖中文和空格。
- 测试盘必须为空盘或已经完整备份。
- 未经用户再次确认，不执行格式化或删除现有移动盘数据。

## 5. 代码仓库策略

当前仓库：

```text
JJames646/codex-portable
```

当前为私有规划仓库，保存需求、审计、原型和计划。

进入实现时创建：

```text
JJames646/codex-portable-workstation
```

要求：

- 从 `BigPizzaV3/CodexPlusPlus` 正式 Fork。
- 客户端仓库公开并保留 AGPL。
- `origin` 指向用户 Fork。
- `upstream` 指向 Codex++ 原仓库。
- 当前规划文档和原型迁入 Fork。
- 每个二进制 Release 关联准确 commit/tag。

Windows 目录最终克隆该 Fork：

```text
D:\Dev\codex-portable
```

服务器阶段再创建私有：

```text
JJames646/codex-workstation-server
```

更新阶段创建公开：

```text
JJames646/codex-workstation-releases
```

## 6. 客户端总体结构

```text
Codex 便携工作站 UI
  │
  ├─ Workstation Core
  │   ├─ USB Root Resolver
  │   ├─ Environment Checker
  │   ├─ Launch State Machine
  │   ├─ Process Supervisor
  │   ├─ Data Layout
  │   ├─ Backup / Factory Reset
  │   └─ Safe Eject
  │
  ├─ Codex++ Core
  │   ├─ Provider
  │   ├─ Protocol Conversion
  │   ├─ Sessions
  │   ├─ Plugins / Skills
  │   ├─ Page Enhancements
  │   └─ Launcher / CDP
  │
  ├─ PortableCodexRuntimePort
  │   ├─ MissingRuntime
  │   ├─ SmokeTestRuntime
  │   └─ CodexDesktopRuntime（后续接入真实载荷）
  │
  ├─ Local Adapters
  │   ├─ Local Overview
  │   ├─ Local Manifest Update
  │   └─ Server Unavailable State
  │
  └─ Reserved Server Ports
      ├─ RelayServicePort
      └─ WorkstationControlPort
```

## 7. 预留的两个服务器连接接口

### 7.1 RelayServicePort

以后连接中转站：

```text
get_dashboard()
get_usage()
get_subscription()
get_model_catalog()
open_web_detail()
```

当前未接入实现：

- 返回 `NotConfigured`。
- 首页只显示 Codex++ 本地概览。
- 用户仍可在供应商页粘贴自己的 API Key。
- 不显示虚假余额、用量或套餐。

### 7.2 WorkstationControlPort

以后连接工作站服务器：

```text
activate_license()
refresh_entitlement()
get_connection_status()
get_route_manifest()
```

在线更新在本阶段由独立 GitHub Release 更新源实现，不依赖工作站业务服务器。

当前未接入实现：

- 返回 `ServerNotConnected`。
- 网络环境页显示“服务器功能尚未接入”。
- 卡密输入框可以保留，但按钮标记为“后续开放”。
- 不启动 Connection Sidecar。
- 不影响本地启动、项目、会话和插件功能。

### 7.3 GitHubReleaseUpdateSource

GitHub 在线更新属于本阶段正常联网功能，不依赖上述两套业务服务器。

本阶段实现：

```text
check_latest_release()
download_release_asset()
verify_manifest()
stage_update()
apply_update()
rollback_update()
```

- 更新源使用自己的 `codex-workstation-releases` GitHub Release。
- 客户端不执行 `git pull`。
- 下载必须验证 schema、SHA-256 和 Ed25519 签名。
- GitHub 不可访问时继续运行当前版本。
- 后续可以增加自有 CDN 镜像，但不改变客户端更新接口。

### 7.4 PortableCodexRuntimePort

用户后续才提供 U 盘版 Codex 载荷，因此启动器不得把某个固定 Codex 目录或版本写死在 UI 和状态机中。

接口职责：

```text
discover(root)
validate(runtime_manifest)
status()
launch(launch_context)
request_stop(runtime_handle)
wait_for_exit(runtime_handle)
version()
```

本阶段包含三个实现：

1. `MissingRuntime`
   - 找不到 `runtime.json` 或 `Codex.exe` 时使用。
   - UI 显示“U 盘版 Codex 尚未安装”。
   - 启动按钮不可执行真实 Codex 启动。
   - 项目、设置、插件和其他启动器页面仍可使用。

2. `SmokeTestRuntime`
   - 由项目构建一个无业务数据的测试 EXE。
   - 只用于验证启动、PID 树、Job Object、停止、重启、日志和安全弹出。
   - 不冒充 Codex，不调用模型，不读取用户项目内容。

3. `CodexDesktopRuntime`
   - 用户提供 U 盘版 Codex 后接入。
   - 读取 `App/Codex/runtime.json`。
   - 设置 `CODEX_HOME`、`CODEX_SQLITE_HOME`、`CODEXPP_HOME`。
   - 通过 Codex++ `--app-path` 启动真实便携 Codex。

这样 RC1 可以在没有真实 U 盘 Codex 载荷时完成；后续 RC2 只增加运行时载荷和适配测试，不重写 UI、路径和进程状态机。

### 7.5 管理员后台

管理员后台不属于客户端，不在本阶段创建假页面。

后续服务器阶段独立实现：

- 只有管理员身份可以登录。
- 用户/工作站数量。
- 卡密生成、禁用、续期和绑定数。
- 客户端版本和最后在线。
- 服务连接状态。
- 中转站套餐和用量。
- 异常、审计和操作日志。

## 8. 移动盘运行目录

```text
Codex-Workstation/
├─ Codex便携工作站.exe
├─ workstation.json
├─ manifest.json
├─ App/
│  ├─ Workstation/
│  ├─ Codex/
│  │  └─ current/app/
│  ├─ Launcher/
│  └─ Runtime/
├─ Data/
│  ├─ Codex/
│  │  ├─ .codex/
│  │  └─ sqlite/
│  ├─ CodexPlusPlus/
│  ├─ Secrets/
│  ├─ Sessions/
│  ├─ Plugins/
│  └─ State/
├─ Workspace/
├─ Backups/
├─ Updates/
│  ├─ staging/
│  └─ previous/
├─ Diagnostics/
├─ Logs/
└─ Licenses/
```

启动时只对子进程设置：

```text
CODEX_HOME=<USB_ROOT>\Data\Codex\.codex
CODEX_SQLITE_HOME=<USB_ROOT>\Data\Codex\sqlite
CODEXPP_HOME=<USB_ROOT>\Data\CodexPlusPlus
WORKSTATION_ROOT=<USB_ROOT>
```

不得写入 Windows 永久环境变量。

### 8.1 workstation.json

启动器根配置只保存非敏感、本地稳定信息：

```json
{
  "schema_version": 1,
  "product": "codex-portable-workstation",
  "channel": "development",
  "data_directory": "Data",
  "workspace_directory": "Workspace",
  "runtime_manifest": "App/Codex/runtime.json",
  "server_mode": "not_connected"
}
```

规则：

- 所有路径相对 `USB_ROOT`。
- 不保存固定盘符。
- 不保存 API Key、Token、密码和服务器管理员信息。
- `server_mode` 后续可以切换为 `remote`，但必须通过版本化配置和签名更新。

### 8.2 runtime.json

真实 U 盘 Codex 载荷以后通过独立 manifest 接入：

```json
{
  "schema_version": 1,
  "runtime_id": "portable-codex-desktop",
  "version": "pending",
  "entrypoint": "current/app/Codex.exe",
  "sha256": "pending",
  "minimum_workstation_version": "0.1.0"
}
```

RC1 的 SmokeTestRuntime 使用独立 `runtime_id`，不得伪装成 Codex。

## 9. 进程管理方案

真实 Windows 机器当前已经运行安装版 Codex，因此不能按进程名停止全部 `Codex.exe`。

正式实现：

- 启动前记录工作站自己的 session ID。
- 记录 launcher、Codex、helper 的 PID 和实际 EXE 路径。
- 使用 Windows Job Object 管理从便携启动器产生的进程树。
- 停止时先发正常退出请求。
- 只停止属于当前 Job Object 或位于移动盘 Codex 路径中的进程。
- 不停止 Microsoft Store 安装版 Codex。
- helper/debug 端口动态分配，避免与已安装 Codex 冲突。
- 只有正常退出超时后，才允许对当前工作站进程树执行受控终止。

### 9.1 Tauri Command 边界

前端只通过统一 Adapter 调用以下命令，不直接操作文件、进程和 Shell：

```text
workstation_bootstrap
workstation_environment_check
portable_runtime_status
workstation_start
workstation_stop
workstation_restart
workstation_status
workspace_inventory
session_inventory
provider_list
provider_save
provider_test
plugin_inventory
backup_create
factory_reset_preview
factory_reset_apply
safe_eject_preview
safe_eject_apply
relay_service_status
workstation_service_status
update_check
update_download
update_apply
update_rollback
```

### 9.2 统一错误码

```text
ROOT_NOT_FOUND
ROOT_OUTSIDE_ALLOWED_PATH
DATA_DIRECTORY_READ_ONLY
INSUFFICIENT_SPACE
RUNTIME_NOT_INSTALLED
RUNTIME_MANIFEST_INVALID
RUNTIME_HASH_MISMATCH
RUNTIME_ALREADY_RUNNING
RUNTIME_START_FAILED
RUNTIME_STOP_TIMEOUT
FILES_STILL_IN_USE
SERVER_NOT_CONNECTED
PROVIDER_CONFIGURATION_INVALID
UPDATE_SIGNATURE_INVALID
UPDATE_ROLLBACK_REQUIRED
EJECT_NOT_READY
```

UI 必须显示用户可理解的中文说明，同时在日志中保留不含 secret 的技术错误码。

## 10. 功能范围

### 10.1 本阶段必须完成

- 正式 Tauri/React 工作站 UI。
- 深色和浅色主题。
- Codex++ 概览和本地工作站状态。
- 环境检测和首次配置页面。
- 移动盘根目录与路径模型。
- `CODEXPP_HOME`。
- 便携 Codex 路径识别和启动。
- 一键启动、一键停止和重新启动。
- 项目目录和最近项目索引。
- 会话读取、导出和安全删除。
- 绘画文件索引。
- Skills、插件、用户脚本和页面增强便携化。
- 供应商配置。
- 中转站 API Key 便捷粘贴。
- 无 PIN Secret Store。
- 本地日志和脱敏诊断。
- 本地恢复点。
- 完整恢复出厂设置。
- GitHub Release 在线更新。
- 本地更新包导入、校验和回滚。
- 安全弹出。
- 两个业务服务器接口的 `NotConnected` Adapter。

### 10.2 本阶段不实现

- Sub2API 真实仪表盘。
- 卡密服务器。
- activation-bridge。
- 管理员后台。
- 网络 Connection Sidecar。
- 远程 route manifest。
- 在线支付。
- 自有 CDN/域名更新镜像。
- 生产服务器部署。

## 11. 非功能要求

### 性能

- 从双击到工作站主窗口可交互：开发机 SSD 上目标小于 3 秒，USB 上目标小于 8 秒。
- 环境检测和在线更新检查异步执行，不阻塞主窗口。
- 项目和会话列表采用分页或增量读取，避免一次加载全部 SQLite 内容。

### 可靠性

- 正常停止优先等待 10 秒，最大受控等待 30 秒。
- 文件句柄未释放时不得显示“可以拔出”。
- 设置和状态文件使用临时文件 + 原子替换写入。
- 更新、恢复出厂设置和迁移都必须限制在解析后的 `USB_ROOT` 允许目录内。
- 服务器不可用不能阻止启动器打开和读取本地项目。

### 安全

- 不以管理员权限运行日常启动器。
- 不修改全局代理、路由表和永久环境变量。
- API Key 不进入普通 JSON、URL、命令行、日志和诊断包。
- 远程网站未接入前不创建拥有本地权限的远程 WebView。
- 更新资产必须验证 SHA-256 和 Ed25519。

### 兼容性

- Windows 11 x64 为首发目标。
- 覆盖中文路径、空格路径、盘符变化和 NTFS。
- WebView2 缺失时显示修复说明，不静默安装未知组件。
- 同时运行电脑安装版 Codex 时，便携进程操作不能影响安装版。

### 可维护性

- Workstation 新功能放在独立 crates/features 中，减少上游 Codex++ 合并冲突。
- 两套业务服务器 DTO 带 `schema_version`。
- 所有上游同步先进入临时分支并运行完整回归。

## 12. 开发阶段

### 阶段 A：Fork 与 Windows 构建基线

工作：

- 创建公开 Codex++ Fork。
- 把当前文档和 UI 原型迁入 Fork。
- Windows 克隆到 `D:\Dev\codex-portable`。
- 记录 `v1.2.37` 和 commit。
- 运行上游完整测试和 Windows Release 构建。

验证：

```powershell
cargo fmt --all -- --check
cargo test --workspace
Set-Location apps\codex-plus-manager
npm ci
npm run check
npm test
npm run vite:build
npm run build
```

完成标准：

- 未修改前的 Codex++ 能在这台 Windows 上构建。
- 失败项在开发前记录，不把环境问题误判为工作站代码问题。

### 阶段 B：UI 与未接入服务器 Adapter

工作：

- 把现有静态原型迁移到 Codex++ Manager。
- 按 feature 拆分页面。
- 建立统一 `workstation-api.ts`。
- 首页接真实 Codex++ 本地概览。
- 两个业务服务器接口使用 `NotConnected` Adapter。
- 移除所有假在线、假余额和假卡密成功。

完成标准：

- Tauri 开发窗口可打开。
- 全部导航和交互可用。
- 断网时 UI 不崩溃；正常联网时官方服务、第三方 API、GitHub 和插件市场可以使用。
- 服务器模块明确显示未接入。

### 阶段 C：便携路径和数据

工作：

- 实现 `PathLayout`。
- 从 EXE 向上发现 `workstation.json`。
- 实现 `CODEXPP_HOME`。
- 重定向 Codex++ settings、status、logs 和用户脚本。
- 创建移动盘目录和 manifest。

完成标准：

- 模拟路径、中文路径、空格路径和更换盘符测试通过。
- 主状态不写入 `%APPDATA%\Codex++`。
- Codex 数据不写入默认用户 `.codex`。

### 阶段 D1：运行时接口与测试运行时

工作：

- 实现 `PortableCodexRuntimePort`。
- 实现 `MissingRuntime`。
- 构建 `SmokeTestRuntime`。
- 实现 Job Object、PID 跟踪、路径校验、启动、停止和重启。
- 验证安装版 Codex 不受影响。

完成标准：

- 没有真实 Codex 载荷时，启动器能准确显示缺失状态。
- SmokeTestRuntime 可以启动、停止、重启和安全退出。
- 关闭工作站不会关闭 Microsoft Store Codex。
- 启动器主体达到 RC1 运行时框架要求。

### 阶段 D2：真实便携 Codex 接入

该阶段等待用户提供真实载荷，不阻塞阶段 E、F 和使用 SmokeTestRuntime 的阶段 G/RC1。

工作：

- 等待用户提供 U 盘版 Codex 载荷。
- 生成和验证 `App/Codex/runtime.json`。
- 复用 Codex++ `--app-path`。
- 注入 `CODEX_HOME`、`CODEX_SQLITE_HOME` 和 `CODEXPP_HOME`。
- 实现 `CodexDesktopRuntime`。
- 运行真实启动、停止、会话和插件测试。

完成标准：

- 启动的是移动盘路径 Codex。
- 安装版 Codex 同时运行时不会被误关。
- 停止后无当前工作站 launcher/helper 残留。
- 启动失败给出真实原因。

### 阶段 E：本地功能完整化

工作：

- 供应商和协议转换。
- API Key 便捷粘贴和 Secret Store。
- 项目、会话、绘画、Skills、插件和脚本。
- 页面增强和失败降级。
- 本地概览与环境检测。

完成标准：

- 换路径重启后配置仍存在。
- API Key 不进入日志、URL、命令行和普通配置。
- 页面增强失败不影响原始 Codex 启动。

### 阶段 F：维护、更新和弹出

工作：

- GitHub Release 在线检查、下载和签名验证。
- 本地更新 ZIP 和签名清单导入作为备用。
- staging、previous 和回滚。
- 恢复点和异常退出检测。
- 完整恢复出厂设置。
- 文件句柄和 SQLite WAL/SHM 检查。
- Windows 安全弹出。

完成标准：

- 损坏更新不破坏旧版。
- 恢复出厂设置只删除当前移动盘工作站目录。
- Codex 运行时拒绝弹出。
- 弹出失败显示占用进程或文件。

### 阶段 G：真实 U 盘发布候选

前置：

- 用户插入已经备份好的 64GB 移动盘。
- 确认盘符和文件系统。

工作：

- 从 `D:\Dev` 构建 Release。
- 使用 packaging 脚本复制到移动盘。
- 在当前 Win11 上进行冷启动。
- 更换盘符后重复启动。
- 在第二台 Windows 电脑测试。

完成标准：

- 用户只需要运行根目录启动器。
- 不要求安装 Codex++。
- 不要求 Microsoft Store Codex 参与运行。
- 项目、会话和配置跟随移动盘。
- 不连接本产品两套业务服务器时，官方服务、第三方 API、GitHub、插件市场和全部本地功能仍可使用。

## 13. 每阶段验证原则

TypeScript：

```powershell
npm run check
npm test
npm run vite:build
```

Rust：

```powershell
cargo fmt --all -- --check
cargo test --workspace
cargo build --release
```

Windows：

```powershell
pwsh -NoProfile -File scripts\test-portable.ps1 -Scenario SimulatedUsb
pwsh -NoProfile -File scripts\test-process-lifecycle.ps1
pwsh -NoProfile -File scripts\test-portable.ps1 -Scenario SafeEject
```

任何没有实际执行的测试必须明确写“未验证”，不得用 UI 演示代替真实验证。

## 14. 开发机当前还需要的准备

正式开始编码前：

1. Windows 最好再重启一次，确认待重启标记是否仍由 ZulerCoreTools 重建；不要手动删除注册表标记。
2. GitHub 仓库可以由 Mac 当前已登录账号创建。
3. Windows 不必先登录 GitHub CLI；可以先只负责拉取和构建。
4. 到真实移动盘测试阶段，再插入 64GB 测试盘。
5. 不安装 Docker Desktop；本阶段不需要服务器和容器。
6. CMake 只有在真实构建报缺失时才安装。

## 15. 执行方式

推荐工作流：

```text
Mac：需求、代码编辑、Git、文档和变更审查
GitHub：客户端 Fork 和版本历史
Windows SSH：Windows 编译、测试、运行和日志
D:\Dev：构建目录
U盘：只存放打包后的运行版本和用户数据
```

这样既能在 Mac 上持续开发，又能使用真实 Win11 环境完成 Tauri、Rust、WebView2、进程和移动盘测试。

## 16. 三个交付层级

### RC1：启动器完成

- 正式 UI。
- U 盘路径、数据、供应商、插件、更新、恢复和安全弹出。
- 两套服务器 `NotConnected` Adapter。
- MissingRuntime 和 SmokeTestRuntime。
- 不包含真实 U 盘 Codex 载荷。

### RC2：U 盘 Codex 集成完成

- 用户提供真实载荷。
- CodexDesktopRuntime 通过。
- 可以从 U 盘启动、停止和继续会话。
- 不依赖 Microsoft Store Codex。

### RC3：两套服务器集成完成

- 接入 RelayServicePort。
- 接入 WorkstationControlPort。
- 开发卡密、Connection Service 和管理员后台。
- 完成服务器部署、监控和运营功能。

RC1、RC2、RC3 分开验收，不允许用 RC1 的测试运行时宣称真实 Codex 已经接入。

## 17. 开始编码后的第一批任务

第一批只做阶段 A，不同时展开服务器：

1. 创建公开 Codex++ Fork。
2. 配置 `origin` 和 `upstream`。
3. 将 Fork 克隆到 Windows `D:\Dev\codex-portable`。
4. 运行 Codex++ 原始测试。
5. 运行 Windows Release 基线构建。
6. 输出 `docs/BASELINE_BUILD_REPORT.md`。

基线构建通过后，再进入 UI 迁移和便携路径代码。

## 18. 预计新增和修改的核心文件

### 客户端 UI

```text
apps/codex-plus-manager/src/features/home/
apps/codex-plus-manager/src/features/providers/
apps/codex-plus-manager/src/features/projects/
apps/codex-plus-manager/src/features/sessions/
apps/codex-plus-manager/src/features/plugins/
apps/codex-plus-manager/src/features/connection/
apps/codex-plus-manager/src/features/updates/
apps/codex-plus-manager/src/features/maintenance/
apps/codex-plus-manager/src/features/settings/
apps/codex-plus-manager/src/lib/workstation-api.ts
```

### 工作站核心

```text
crates/workstation-core/src/path_layout.rs
crates/workstation-core/src/state_machine.rs
crates/workstation-core/src/environment.rs
crates/workstation-core/src/errors.rs
crates/workstation-runtime/src/port.rs
crates/workstation-runtime/src/missing.rs
crates/workstation-runtime/src/smoke.rs
crates/workstation-runtime/src/codex_desktop.rs
crates/workstation-process/src/job_object_windows.rs
crates/workstation-process/src/process_tree.rs
crates/workstation-portable/src/backup.rs
crates/workstation-portable/src/factory_reset.rs
crates/workstation-portable/src/eject_windows.rs
crates/workstation-secrets/src/vault.rs
crates/workstation-update/src/*
```

### 两套业务服务器接口

```text
crates/workstation-server-client/src/relay_port.rs
crates/workstation-server-client/src/workstation_control_port.rs
crates/workstation-server-client/src/not_connected.rs
contracts/relay-service/v1/
contracts/workstation-control/v1/
```

### 测试与打包

```text
apps/workstation-runtime-smoke/
scripts/test-portable.ps1
scripts/test-process-lifecycle.ps1
scripts/package-workstation.ps1
packaging/workstation-layout/
tests/portable-paths/
tests/process-lifecycle/
tests/server-contract/
```

## 19. 阶段检查点

每个阶段结束时必须提交一份短报告：

```text
docs/reports/PHASE-A-BASELINE.md
docs/reports/PHASE-B-UI.md
docs/reports/PHASE-C-PORTABLE-PATHS.md
docs/reports/PHASE-D1-RUNTIME-SMOKE.md
docs/reports/PHASE-D2-CODEX-RUNTIME.md
docs/reports/PHASE-E-LOCAL-FEATURES.md
docs/reports/PHASE-F-MAINTENANCE.md
docs/reports/PHASE-G-USB-RC.md
```

报告至少记录：

- 修改的文件和 commit。
- 实际执行的测试命令。
- 通过、失败和未验证项。
- Windows 日志和截图位置。
- 是否影响电脑安装版 Codex。
- 是否产生项目目录外写入。
- 下一阶段是否具备进入条件。
