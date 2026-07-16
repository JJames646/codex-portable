# AGENTS.md

## 项目目标

本仓库开发“Codex 便携工作站”：从 Windows 移动存储启动 Codex 兼容桌面运行时，并在不同电脑间携带项目、会话、Skills、插件、供应商配置和可迁移状态。

## 当前阶段

当前只允许进行上游审计、架构、接口契约、UI 原型和开发计划工作。正式功能编码必须以 `docs/DELIVERY_PLAN.md` 的阶段为边界，并在阶段开始前确认。

## 技术路线

- 客户端：优先基于 Codex++ Fork，保持 Rust + Tauri 2 + React + TypeScript。
- 便携运行时：复用 `lusipad/unofficial-codex-app-offline` 的构建流程，但不得在许可审查前公开分发其应用载荷。
- 服务端：Sub2API 独立部署；新增能力放在独立 `activation-bridge` 和 adapter 中。
- 更新：客户端只连接自己的签名更新源，不直接拉取社区仓库。

## 强制路径规则

- 禁止写死 `C:`、`D:`、`E:`、`F:`、`U:` 等盘符。
- 移动盘根目录必须由当前 EXE 所在目录推导。
- 所有跨电脑状态必须显式归入 `Data/` 或 `Workspace/`。
- `CODEX_HOME`、`CODEX_SQLITE_HOME` 和未来的 `CODEXPP_HOME` 必须只对子进程设置，不得写入 Windows 永久环境变量。
- 必须测试中文路径、空格路径、盘符变化、只读和空间不足。

## 开发机保护规则

- Windows 电脑是用户日常使用的个人电脑，不得把它当作可随意清理的测试机。
- 开发写入只允许进入 `D:\Dev\codex-portable`、明确指定的构建缓存目录和用户确认过的测试移动盘目录。
- 禁止删除、移动或覆盖用户其他项目、桌面、下载、文档、图片和个人配置。
- 禁止卸载或修改电脑上现有的 Microsoft Store Codex、其他软件和用户数据。
- 不得按进程名批量终止所有 `Codex.exe`；只能管理当前工作站启动并通过路径、PID 树或 Job Object 确认的便携进程。
- 禁止格式化磁盘、调整分区或清理真实移动盘，除非用户看到准确目标并再次明确确认。
- 禁止修改系统全局代理、永久环境变量、自启动项和安全策略；确有必要时必须先说明影响并获得确认。
- 清理构建产物时只能删除项目自己的 `target`、`node_modules`、staging 和临时测试目录，并在执行前确认解析后的绝对路径仍位于允许目录。

## 凭据与日志

- 禁止硬编码 API Key、管理员 Key、JWT Secret、数据库密码和支付密钥。
- API Key 不得出现在命令行参数、URL、日志、诊断包和明文配置中。
- 用户已确认不设置 PIN。客户端凭据使用同盘随机 vault key 和 AES-256-GCM，必须明确提示该模式不能防止移动盘丢失后的专业提取。
- 运行时优先通过 Codex provider 的 `env_key` 引用子进程环境变量。
- 诊断信息必须经过字段级脱敏和最终敏感信息扫描。
- 不得把完整 Codex++ `settings.json` 直接写入诊断包，因为上游当前可能包含 API Key。

## 更新与供应链

- 社区仓库更新只允许在开发机或 CI 中同步。
- 用户端只读取自己的 `stable.json`/`beta.json`。
- 更新清单与组件必须校验 schema、SHA-256 和 Ed25519 签名。
- 更新不得覆盖 `Data/`、`Workspace/`、`Backups/` 和加密凭据。
- 更新失败必须保留旧版本并自动回滚。
- 不得把 GitHub Release 的“最新版本”直接等同于已兼容版本。

## 进程与安全退出

- 启动顺序必须由统一状态机控制。
- “启动 Codex”应先完成目录、凭据、服务连接和兼容性检查，再启动 Codex++ 和 Codex。
- “停止”应先请求正常退出，再等待 SQLite WAL/SHM 和文件句柄释放。
- 默认不得强杀未知进程。
- 安全弹出失败时必须显示占用进程或文件。

## 服务器边界

- 客户端不得调用 Sub2API 管理员接口。
- 管理员 Key 只能存在于服务器端 adapter/activation-bridge。
- 工作站授权不要求用户账号，使用卡密、移动盘 `installation_id`、安装实例公钥、签名 entitlement 和可撤销短期令牌。
- 卡密从首次成功激活开始计时，绑定数量由管理员后台配置。
- 远程线路配置必须签名、白名单化且只能指向本产品自有 API 服务。
- 不实现通用 VPN、任意目标代理、账号共享、额度绕过或上游权限绕过。

## 上游代码规则

- `third_party/upstreams/` 是本地审计目录，不直接修改。
- 需要改 Codex++ 时，先创建正式 Fork 并配置 `origin`/`upstream`。
- 修改 Codex++ 必须保留 AGPL 许可证、第三方声明和对应源码提供方式。
- 用户已接受向接收者提供对应修改源码；每个二进制 Release 必须关联准确源码 commit/tag 和构建说明。
- 不要把上游大型二进制或生成物提交到主产品仓库。

## 验证要求

每次功能修改至少执行与影响范围匹配的：

```text
TypeScript: npm run check / npm test / npm run vite:build
Rust: cargo fmt --all -- --check / cargo test / cargo build --release
Go: go test ./...
Windows: PowerShell 路径、进程、盘符和安全退出测试
```

无法实际运行 Windows 测试时必须明确说明，不得宣称通过。
