# Codex 便携工作站开发就绪报告

检查时间：2026-07-16

## 1. Go / No-Go 结论

```text
阶段 A：已完成
阶段 B：等待用户明确开始
```

Codex++ Fork、Windows 克隆、原版基线测试和 Windows Release 构建均已完成，详细结果见 `docs/BASELINE_BUILD_REPORT.md`。

真实移动盘、真实 U 盘版 Codex 和两套业务服务器分别属于后续阶段，不阻塞阶段 A。

## 2. 已锁定开发方案

- 正式 UI 使用当前已经确认的“Codex 便携工作站”原型，不重新设计布局和视觉风格。
- 客户端基于 Codex++ Fork，继续使用 Rust + Tauri 2 + React + TypeScript。
- 开发和编译位于 Windows `D:\Dev\codex-portable`，不直接在 U 盘编译。
- 启动器和用户数据从移动盘运行，路径不写死盘符。
- 只暂不连接中转站网站服务器和工作站卡密/网络控制服务器。
- Codex 官方服务、第三方 API、GitHub、插件市场、远程项目和 GitHub Release 更新正常联网。
- 两套业务服务器使用 `RelayServicePort`、`WorkstationControlPort` 和 `NotConnected` Adapter。
- 用户后续提供 U 盘版 Codex；当前先通过 `PortableCodexRuntimePort` 和 `SmokeTestRuntime` 完成 RC1。
- 正式运行时通过 PID 树、EXE 路径和 Windows Job Object 管理，不影响电脑安装版 Codex。
- 不清除用户电脑上的其他软件、项目和个人文件。

## 3. Mac 开发端

| 检查项 | 状态 | 结果 |
|---|---|---|
| 当前仓库 | 通过 | `/Users/jamesj/Documents/codex便携版项目开发` |
| Git 分支 | 通过 | `main`，跟踪 `origin/main` |
| 当前远端 | 通过 | `JJames646/codex-portable` |
| 仓库可见性 | 已知 | 当前为 Private 规划仓库 |
| GitHub 登录 | 通过 | Mac 已登录 `JJames646` |
| 文档格式 | 通过 | `git diff --check` 无错误 |
| UI JavaScript | 通过 | `node --check` 通过 |
| UI 预览 | 通过 | `http://127.0.0.1:8787/` 返回 200 |
| 上游审计源码 | 通过 | 四个仓库已固定 commit |

私有规划仓库基线已提交并推送。阶段 A 新增的构建报告将在阶段 A 收尾提交中一并推送。

## 4. Windows 开发端

| 检查项 | 状态 | 结果 |
|---|---|---|
| SSH | 通过 | 公钥免密码连接正常 |
| 主机 | 通过 | `LAPTOP-892MM2QA` |
| 用户 | 通过 | `James` |
| 系统 | 通过 | Windows 11 x64，Build 22621.4317 |
| 开发目录 | 通过 | `D:\Dev\codex-portable` 已克隆公开 Fork |
| D 盘空间 | 通过 | 约 396.3GB 可用 |
| Rust | 通过 | rustc/cargo 1.97.0 stable-msvc |
| Node/npm | 通过 | Node 24.18.0，npm 11.16.0 |
| PowerShell | 通过 | PowerShell 7.6.3 |
| C++ Build Tools | 通过 | Visual Studio Build Tools 2022 |
| Windows SDK | 通过 | 10.0.22621、10.0.26100 |
| WebView2 | 通过 | Runtime 150.0.4078.65 |
| Git/打包工具 | 通过 | Git、7-Zip、Inno Setup、MSBuild |
| GitHub CLI 登录 | 不要求 | 仓库和 Fork 由 Mac 已登录会话创建 |
| Docker | 不需要 | 当前客户端阶段不使用 |
| CMake | 按需 | 只有实际构建报缺失时再安装 |

## 5. 实机风险检查

### 安装版 Codex

Windows 当前存在多个安装版 Codex 进程。

开发规则：

- 禁止按进程名批量停止全部 `Codex.exe`。
- 基线构建不操作现有 Codex 进程。
- 便携运行时测试使用 SmokeTestRuntime。
- 后续真实运行时必须通过路径、PID 树和 Job Object 隔离。

### 待重启标记

- CBS `RebootPending` 为 false。
- `PendingFileRenameOperations` 仍由第三方 `ZulerCoreTools` 路径产生。
- 当前不阻塞 Rust/Tauri 基线构建。
- 不手动删除该注册表项。

### 移动盘

当前没有检测到 USB 移动盘。

- 不阻塞阶段 A 到 F。
- 阶段 G 前由用户插入已经备份好的 64GB 测试盘。
- 未确认盘符和备份状态前不得格式化或清理移动盘。

## 6. GitHub 状态

已经存在：

```text
JJames646/codex-portable
JJames646/codex-portable-workstation
```

尚未创建：

```text
JJames646/codex-workstation-releases
```

阶段 A 已完成：

1. 已从 Codex++ 创建公开 Fork。
2. 已建立 `origin` 和只拉取的 `upstream`。
3. 已迁移公开技术文档和脱敏 UI 原型。
4. 已在 Windows 克隆到 `D:\Dev\codex-portable`。
5. 已固定 `v1.2.37` / `2924333c6770497470090235f484519154698651`。
6. 已通过测试和 Release 构建。

服务器私有仓库在 RC3 前不创建。

## 7. 三层交付确认

### RC1

- 完整启动器 UI。
- 便携路径、本地数据、供应商、项目、插件、更新、恢复和安全弹出。
- SmokeTestRuntime。
- 两套服务器显示未接入。

### RC2

- 用户提供真实 U 盘版 Codex。
- 接入 CodexDesktopRuntime。
- 完成真实启动、停止、会话和跨电脑测试。

### RC3

- 用户提供两套服务器。
- 接入中转站、卡密、网络服务和管理员后台。

## 8. 阶段 A 执行结果

```text
[完成] 审查、提交并推送规划仓库基线
[完成] 创建公开 Codex++ Fork
[完成] 配置 origin/upstream
[完成] 迁移公开技术文档和脱敏 UI 原型
[完成] 克隆到 D:\Dev\codex-portable
[完成] 固定 v1.2.37 / 2924333c6770497470090235f484519154698651
[完成] npm ci
[完成] npm run check
[完成] npm test
[完成] npm run vite:build
[完成] cargo fmt --all -- --check
[完成] cargo test --workspace
[完成] npm run build
[完成] 输出 docs/BASELINE_BUILD_REPORT.md
```

## 9. 阶段 A 完成标准

- Codex++ 原始代码能在用户 Win11 电脑上构建。
- 所有失败项有真实日志。
- 没有修改或关闭安装版 Codex。
- 没有写入项目目录之外的产品数据。
- 没有开始服务器开发。
- 没有把测试运行时宣传为真实 Codex。

## 10. 下一阶段授权边界

阶段 A 收尾提交和推送完成后，不自动进入功能编码。用户明确发送：

```text
开始阶段 B
```

才开始把当前静态 UI 迁移到 Codex++ Manager，并建立两个业务服务器的 `NotConnected` Adapter。
