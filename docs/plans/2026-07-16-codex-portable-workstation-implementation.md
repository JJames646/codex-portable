# Codex Portable Workstation Implementation Plan

> **For Codex:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task.

**Goal:** 将当前静态 UI 原型迁移为基于 Codex++ Fork 的 Windows U 盘便携工作站，并逐步接入启动、数据、供应商、服务器、连接服务、更新和安全弹出。

**Architecture:** 以 Codex++ 的 Rust/Tauri/React 工程为客户端底座，新增模块化 workstation crates；Sub2API 保持独立服务，使用 activation-bridge 处理无账号卡密和移动工作站授权；客户端只使用自己的签名更新源和目标域名白名单化的连接 sidecar。

**Tech Stack:** Rust 2024、Tauri 2、React 19、TypeScript、Go、Gin、PostgreSQL、Redis、Docker Compose、PowerShell 7、GitHub Actions。

---

## Task 1：建立正式 Fork 和基线

**Files:**

- Create: `docs/FORK_SETUP.md`
- Create: `scripts/sync-upstreams.ps1`
- Create: `.github/workflows/client-ci.yml`
- Modify: Codex++ Fork root `Cargo.toml`
- Test: 上游现有 Rust/TypeScript 测试

**Step 1：记录上游版本**

在 `docs/FORK_SETUP.md` 写入：

```text
upstream=https://github.com/BigPizzaV3/CodexPlusPlus.git
baseline_tag=v1.2.37
baseline_commit=2924333c6770497470090235f484519154698651
```

**Step 2：运行基线测试**

Run:

```bash
cargo fmt --all -- --check
cargo test
cd apps/codex-plus-manager
npm ci
npm run check
npm test
npm run vite:build
```

Expected: 全部 PASS；失败项必须在修改前记录。

**Step 3：添加 CI**

CI 必须运行同一组命令，并上传 Windows x64 构建日志，不发布二进制。

**Step 4：提交**

```bash
git add docs/FORK_SETUP.md scripts/sync-upstreams.ps1 .github/workflows/client-ci.yml
git commit -m "chore: 建立 Codex++ Fork 与 CI 基线"
```

## Task 2：实现 Portable Root 与路径模型

**Files:**

- Create: `crates/workstation-core/src/path_layout.rs`
- Create: `crates/workstation-core/src/lib.rs`
- Create: `crates/workstation-core/tests/path_layout.rs`
- Modify: workspace `Cargo.toml`

**Step 1：写失败测试**

测试场景：

```rust
#[test]
fn resolves_usb_root_from_manifest_with_chinese_and_space_path() {
    let root = temp.path().join("我的 U盘").join("Codex 工作站");
    std::fs::create_dir_all(root.join("App/Workstation")).unwrap();
    std::fs::write(root.join("workstation.json"), "{}").unwrap();
    let exe = root.join("App/Workstation/Codex便携工作站.exe");

    let layout = PathLayout::discover_from_executable(&exe).unwrap();

    assert_eq!(layout.root(), root);
    assert_eq!(layout.codex_home(), root.join("Data/Codex/.codex"));
}
```

**Step 2：确认失败**

Run:

```bash
cargo test -p workstation-core --test path_layout
```

Expected: FAIL，`PathLayout` 尚不存在。

**Step 3：最小实现**

实现：

- 从 EXE 父目录向上最多查找 5 层。
- 找到 `workstation.json` 即确定 root。
- 提供 `codex_app_dir`、`codex_home`、`sqlite_home`、`codexpp_home`、`workspace`、`backups`、`staging`。
- 所有路径使用 `PathBuf`。

**Step 4：运行测试**

Run:

```bash
cargo test -p workstation-core --test path_layout
```

Expected: PASS。

**Step 5：提交**

```bash
git add Cargo.toml crates/workstation-core
git commit -m "feat: 新增便携工作站路径模型"
```

## Task 3：实现 CODEXPP_HOME

**Files:**

- Modify: `crates/codex-plus-core/src/paths.rs`
- Modify: `apps/codex-plus-manager/src-tauri/src/commands.rs`
- Test: `crates/codex-plus-core/tests/portable_paths.rs`

**Step 1：写失败测试**

```rust
#[test]
fn codexpp_home_redirects_all_state_paths() {
    let home = temp.path().join("Codex++ 数据");
    std::fs::create_dir_all(&home).unwrap();
    let _guard = EnvGuard::set("CODEXPP_HOME", &home);

    assert_eq!(default_settings_path(), home.join("settings.json"));
    assert_eq!(default_latest_status_path(), home.join("latest-status.json"));
    assert_eq!(default_diagnostic_log_path(), home.join("logs/codex-plus.log"));
}
```

**Step 2：确认失败**

Run:

```bash
cargo test -p codex-plus-core portable_paths
```

Expected: FAIL，仍使用用户主目录。

**Step 3：实现**

`default_app_state_dir()` 优先读取有效 `CODEXPP_HOME`。用户脚本目录从同一根目录派生。

**Step 4：兼容测试**

Run:

```bash
cargo test -p codex-plus-core
```

Expected: 上游默认路径测试和新便携测试均 PASS。

**Step 5：提交**

```bash
git add crates/codex-plus-core/src/paths.rs apps/codex-plus-manager/src-tauri/src/commands.rs crates/codex-plus-core/tests/portable_paths.rs
git commit -m "feat: 支持 CODEXPP_HOME 便携状态目录"
```

## Task 4：建立工作站启动状态机

**Files:**

- Create: `crates/workstation-core/src/state_machine.rs`
- Create: `crates/workstation-core/src/launch_orchestrator.rs`
- Create: `crates/workstation-core/tests/launch_orchestrator.rs`
- Modify: `apps/codex-plus-manager/src-tauri/src/lib.rs`
- Modify: `apps/codex-plus-manager/src-tauri/src/commands.rs`

**Step 1：写状态转换测试**

覆盖：

```text
Idle → Checking → PreparingConnection → StartingCodex → Running
Running → Stopping → FlushingData → ReadyToEject
任何阶段 → Failed(error_code)
```

**Step 2：确认失败**

Run:

```bash
cargo test -p workstation-core launch_orchestrator
```

**Step 3：实现 Tauri Commands**

新增：

```text
workstation_check
workstation_start
workstation_stop
workstation_status
```

`workstation_start`：

- 创建目录。
- 设置子进程环境。
- 传递 `--app-path`。
- 调用 Codex++ 启动链。

**Step 4：Mock 测试**

用 fake launcher 验证调用顺序和失败回滚。

**Step 5：提交**

```bash
git add crates/workstation-core apps/codex-plus-manager/src-tauri
git commit -m "feat: 新增工作站启动状态机"
```

## Task 5：暴露停止与文件释放检查

**Files:**

- Modify: `apps/codex-plus-manager/src-tauri/src/commands.rs`
- Modify: `apps/codex-plus-manager/src-tauri/src/lib.rs`
- Create: `crates/workstation-portable/src/file_locks.rs`
- Test: `crates/workstation-portable/tests/file_locks.rs`

**Step 1：写失败测试**

验证：

- stop 会调用 launcher 和 Codex 停止。
- 仍有 SQLite WAL/SHM 时不能进入 ReadyToEject。
- 超时返回占用详情。

**Step 2：实现**

新增：

```text
stop_codex_plus
check_workstation_locks
```

不得默认强杀非本产品进程。

**Step 3：运行**

```bash
cargo test -p workstation-portable
cargo test -p codex-plus-manager
```

**Step 4：提交**

```bash
git add apps/codex-plus-manager/src-tauri crates/workstation-portable
git commit -m "feat: 新增停止与文件释放检查"
```

## Task 6：实现无 PIN Secret Store

**Files:**

- Create: `crates/workstation-secrets/src/vault.rs`
- Create: `crates/workstation-secrets/src/key_store.rs`
- Create: `crates/workstation-secrets/src/redact.rs`
- Test: `crates/workstation-secrets/tests/vault.rs`

**Step 1：写失败测试**

覆盖：

- 首次运行生成随机 vault key。
- 重启后自动解密。
- vault key 缺失时返回可理解错误。
- nonce 重复检测。
- 密文修改认证失败。
- 序列化文件中没有明文 Key。

**Step 2：实现**

- AES-256-GCM。
- 版本化 envelope。
- vault key 保存在移动盘受控目录，不进入普通配置和日志。
- `SecretString` Debug 不输出内容。
- UI 显示“无密码模式不能防止移动盘丢失后的密钥提取”。

**Step 3：运行**

```bash
cargo test -p workstation-secrets
```

**Step 4：提交**

```bash
git add crates/workstation-secrets
git commit -m "feat: 新增无PIN便携凭据库"
```

## Task 7：改造 Codex++ Provider 为 env_key

**Files:**

- Modify: `crates/codex-plus-core/src/settings.rs`
- Modify: `crates/codex-plus-core/src/provider_import.rs`
- Modify: `crates/codex-plus-core/src/relay_config.rs`
- Modify: `crates/codex-plus-core/src/relay_switch.rs`
- Modify: `apps/codex-plus-manager/src-tauri/src/commands.rs`
- Test: `crates/codex-plus-core/tests/provider_secret_reference.rs`

**Step 1：写失败测试**

断言最终配置：

```toml
env_key = "CODEX_WORKSTATION_API_KEY"
requires_openai_auth = false
```

同时断言不包含：

```text
sk-test
experimental_bearer_token
OPENAI_API_KEY
```

**Step 2：实现**

- 新 provider 使用 `credential_id`。
- Secret Broker 负责运行时注入。
- legacy 模式只用于迁移。

**Step 3：运行**

```bash
cargo test -p codex-plus-core provider_secret_reference
cargo test -p codex-plus-core relay
```

**Step 4：提交**

```bash
git add crates/codex-plus-core apps/codex-plus-manager/src-tauri
git commit -m "feat: 供应商凭据改为环境变量引用"
```

## Task 8：迁移当前 UI 原型

**Files:**

- Create: `apps/codex-plus-manager/src/features/*`
- Create: `apps/codex-plus-manager/src/components/workstation/*`
- Create: `apps/codex-plus-manager/src/features/providers/QuickKeyPaste.tsx`
- Modify: `apps/codex-plus-manager/src/App.tsx`
- Modify: `apps/codex-plus-manager/src/styles.css`
- Test: `apps/codex-plus-manager/src/workstation-navigation.test.tsx`

**Step 1：写导航测试**

验证左侧页面和顶部按钮存在，启动/停止状态正确。

**Step 2：迁移静态原型**

按 feature 拆分：

```text
home providers projects sessions plugins enhancements remote data
connection updates maintenance settings
```

供应商页面增加本产品中转站便捷粘贴框：使用内置 Base URL、协议和模型预设，粘贴 API Key 后自动清理格式、保存 credential reference 并执行连接检测。

**Step 3：接入 Tauri Adapter**

前端不直接散落 `invoke()`，统一通过：

```text
src/lib/workstation-api.ts
```

**Step 4：运行**

```bash
cd apps/codex-plus-manager
npm run check
npm test
npm run vite:build
```

**Step 5：提交**

```bash
git add apps/codex-plus-manager
git commit -m "feat: 迁移便携工作站 UI 到 Codex++"
```

## Task 9：实现工作站概览与可选中转站仪表盘 Adapter

**Files:**

- Create: `crates/workstation-server-client/src/dashboard.rs`
- Create: `crates/workstation-server-client/src/client.rs`
- Create: `crates/workstation-core/src/overview.rs`
- Test: `crates/workstation-server-client/tests/dashboard_contract.rs`
- Modify: `apps/codex-plus-manager/src/features/home/*`

**Step 1：用 Mock Server 写失败测试**

模拟：

```text
/api/v1/user/profile
/api/v1/usage/dashboard/snapshot-v2
/api/v1/subscriptions/summary
```

验证 Codex++ Overview、Workstation Health 和可选中转站用量转换为统一 `DashboardOverview`。未配置中转站时不请求 Sub2API。

**Step 2：实现**

- 明确超时。
- 中转站 401 提示用户重新粘贴 API Key。
- 服务器不可用返回 degraded，不阻塞本地 UI。

**Step 3：运行**

```bash
cargo test -p workstation-server-client
```

**Step 4：提交**

```bash
git add crates/workstation-server-client apps/codex-plus-manager/src/features/home
git commit -m "feat: 对接工作站概览和中转站用量"
```

## Task 10：实现无账号卡密 activation-bridge

**Files:**

- Create: `server/activation-bridge/cmd/server/main.go`
- Create: `server/activation-bridge/internal/license/service.go`
- Create: `server/activation-bridge/internal/license/entitlement.go`
- Create: `server/activation-bridge/internal/installations/service.go`
- Create: `server/activation-bridge/internal/sub2api/client.go`
- Create: `server/activation-bridge/internal/api/routes.go`
- Create: `server/activation-bridge/migrations/001_license_codes.sql`
- Create: `server/activation-bridge/migrations/002_installations.sql`
- Test: `server/activation-bridge/internal/license/service_test.go`

**Step 1：写失败测试**

覆盖：

- 卡密从首次激活开始计算有效期。
- 重复激活保持同一到期时间。
- 最大工作站绑定数量。
- 禁用、到期、解绑和续期。
- 激活接口限速和卡密暴力尝试防护。
- refresh token 重放撤销。
- 管理员 Key 不出现在响应和日志。

**Step 2：实现最小 API**

```text
POST /workstation/v1/licenses/activate
POST /workstation/v1/licenses/refresh
POST /workstation/v1/licenses/deactivate
GET  /workstation/v1/licenses/status
```

**Step 3：运行**

```bash
cd server/activation-bridge
go test ./...
```

**Step 4：提交**

```bash
git add server/activation-bridge
git commit -m "feat: 新增无账号卡密授权桥接服务"
```

## Task 10B：实现独立管理员后台

**Files:**

- Create: `server/admin-web/src/pages/licenses/*`
- Create: `server/admin-web/src/pages/installations/*`
- Create: `server/admin-web/src/pages/service-health/*`
- Create: `server/admin-web/src/api/client.ts`
- Test: `server/admin-web/src/pages/licenses/licenses.test.tsx`

**Step 1：写失败测试**

覆盖：

- 批量生成卡密。
- 禁用、续期、备注和查询。
- 查看首次激活、到期时间、剩余时间和绑定工作站。
- 修改最大绑定数量和手动解绑。
- 查看服务连接状态与客户端版本。
- 普通客户端凭据不能访问管理员接口。

**Step 2：实现**

- 管理后台独立部署，不进入工作站客户端导航。
- 管理操作使用管理员身份认证、MFA、审计日志和最小权限。
- 页面不显示完整模型 API Key、数据库密码或签名私钥。

**Step 3：运行**

```bash
cd server/admin-web
npm run check
npm test
npm run build
```

**Step 4：提交**

```bash
git add server/admin-web
git commit -m "feat: 新增工作站卡密管理员后台"
```

## Task 11：实现受限 Connection Sidecar

**Files:**

- Create: `sidecars/connection-service/src/main.rs`
- Create: `sidecars/connection-service/src/allowlist.rs`
- Create: `sidecars/connection-service/src/session.rs`
- Test: `sidecars/connection-service/tests/security.rs`

**Step 1：写安全失败测试**

验证：

- 非 loopback 绑定拒绝。
- 非白名单 host 拒绝。
- 任意 CONNECT 拒绝。
- 无本地会话 token 拒绝。
- 日志不记录正文和 Authorization。

**Step 2：实现**

只支持工作站所需 HTTP/SSE/WebSocket 转发。

**Step 3：运行**

```bash
cargo test -p connection-service
```

**Step 4：提交**

```bash
git add sidecars/connection-service
git commit -m "feat: 新增受限工作站连接服务"
```

## Task 12：实现签名更新器

**Files:**

- Create: `crates/workstation-update/src/manifest.rs`
- Create: `crates/workstation-update/src/signature.rs`
- Create: `crates/workstation-update/src/apply.rs`
- Create: `apps/workstation-updater/src/main.rs`
- Test: `crates/workstation-update/tests/update_flow.rs`
- Create: `updates/stable.schema.json`

**Step 1：写失败测试**

覆盖：

- 正确签名。
- 错误签名。
- 损坏下载。
- Zip Slip。
- 磁盘不足。
- 健康检查失败回滚。
- Data/Workspace 路径保护。

**Step 2：实现**

先实现本地文件源测试，再接 HTTPS 下载。

**Step 3：运行**

```bash
cargo test -p workstation-update
cargo test -p workstation-updater
```

**Step 4：提交**

```bash
git add crates/workstation-update apps/workstation-updater updates
git commit -m "feat: 新增签名更新与自动回滚"
```

## Task 13：实现恢复点与安全弹出

**Files:**

- Create: `crates/workstation-portable/src/backup.rs`
- Create: `crates/workstation-portable/src/eject_windows.rs`
- Test: `crates/workstation-portable/tests/backup_rotation.rs`
- Test: `crates/workstation-portable/tests/eject_guard.rs`

**Step 1：写失败测试**

验证 latest → previous 轮换、异常中断和占用阻止弹出。

**Step 2：实现**

- 只备份状态和索引，项目大文件不做每次全盘复制。
- 更新前可创建完整关键状态快照。
- Windows 弹出失败返回占用信息。

**Step 3：Windows 实机测试**

Run:

```powershell
pwsh -NoProfile -File scripts/test-portable.ps1 -Scenario SafeEject
```

Expected: 停止、落盘、无占用后才允许弹出。

**Step 4：提交**

```bash
git add crates/workstation-portable scripts/test-portable.ps1
git commit -m "feat: 新增恢复点与安全弹出"
```

## Task 13B：实现完整恢复出厂设置

**Files:**

- Create: `crates/workstation-portable/src/factory_reset.rs`
- Modify: `apps/codex-plus-manager/src/features/settings/*`
- Test: `crates/workstation-portable/tests/factory_reset.rs`

**Step 1：写失败测试**

验证：

- 未停止 Codex、Codex++、sidecar 或 updater 时拒绝执行。
- 未输入指定确认文字时拒绝执行。
- 只允许删除 `USB_ROOT` 下明确列出的工作站目录。
- 删除设置、卡密、凭据、项目、会话、插件、Skills、绘画、备份和缓存。
- 不允许路径穿越、符号链接逃逸和删除移动盘外文件。
- 中途失败时返回已删除和未删除清单，不伪报成功。

**Step 2：实现**

- UI 明确列出不可撤销的数据范围。
- 提供一次可选导出入口，但不默认创建保留副本。
- 二次确认后执行全量清理并重新创建空白出厂目录。

**Step 3：运行**

```bash
cargo test -p workstation-portable factory_reset
```

**Step 4：提交**

```bash
git add crates/workstation-portable apps/codex-plus-manager/src/features/settings
git commit -m "feat: 新增完整恢复出厂设置"
```

## Task 14：总集成与发布候选

**Files:**

- Create: `scripts/package-workstation.ps1`
- Create: `packaging/workstation-layout/manifest.template.json`
- Create: `docs/TEST_REPORT.md`
- Create: `docs/RELEASE_CHECKLIST.md`

**Step 1：组装私有 Release**

```powershell
pwsh -NoProfile -File scripts/package-workstation.ps1
```

**Step 2：两机交替测试**

覆盖：

- 不同盘符。
- 中文和空格路径。
- 运行、停止、弹出。
- 家/公司两台 Windows 电脑。
- 项目、会话、插件和供应商。
- 更新和回滚。
- 断网。

**Step 3：安全扫描**

- 搜索 Release、日志和诊断中的 Key。
- 检查远程 WebView 权限。
- 检查第三方许可证。

**Step 4：记录结果**

所有未通过项必须写入 `docs/TEST_REPORT.md`，不得输出伪成功 Release。

**Step 5：提交**

```bash
git add scripts packaging docs/TEST_REPORT.md docs/RELEASE_CHECKLIST.md
git commit -m "chore: 完成工作站发布候选验证"
```

## 实施顺序约束

- Task 1–5 完成前，不对外接服务器。
- Task 6–7 完成前，不保存真实用户 API Key。
- Task 10 完成前，客户端不调用 Sub2API 管理接口。
- Task 11 完成前，不宣称网络环境功能已实现。
- Task 12 完成前，不开放自动更新。
- 许可证 Gate 未通过前，不公开分发便携 Codex 应用载荷。
