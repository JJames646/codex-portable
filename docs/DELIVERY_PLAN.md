# Codex 便携工作站分阶段交付计划

## 当前执行顺序调整

用户已确认采用“本地联网客户端优先，暂不连接两套业务服务器”：

1. 先完成客户端 Fork、UI、便携路径、Codex 启停、本地数据、供应商、插件、维护、恢复和安全弹出。
2. 中转站与工作站服务器只保留接口和“未接入”状态，不连接真实服务。
3. Codex 官方服务、用户第三方 API、GitHub、插件市场、远程项目和 GitHub Release 更新保持正常联网。
4. 本地联网客户端在真实 64GB 移动盘和至少两台 Windows 电脑验证通过后，再开发 Sub2API、卡密、Connection Service 和管理员后台。
5. 详细执行方案见 `docs/plans/2026-07-16-local-networked-usb-launcher-development-plan.md`。

## 阶段 0：需求、上游和法律 Gate

已完成：

- 读取原始需求。
- 拉取四个上游。
- 分析 Codex++、Sub2API、便携 Codex 和 OpenAI Codex。
- 建立架构、功能矩阵、安全、更新和许可文档。

已确认：

- 正式客户端技术栈使用 Rust + Tauri + React。
- 目标是把 Codex++ 功能整合进工作站 UI，并且只从移动盘启动。
- Win11 x64 和 64GB 测试移动存储。

已解除：

- 用户接受 Codex++ AGPL 对应源码要求，Codex++ Fork 可以继续作为正式客户端路线。

待确认：

- 便携 Codex 桌面载荷的再分发权仍需单独确认。
- 服务器正式域名和部署条件后续填写。

验收：

- 所有核心结论写入仓库。
- 不凭空假设 Sub2API 存在无账号工作站卡密接口。
- 明确商业阻断项。

## 阶段 1：正式 Codex++ Fork 与集成测试骨架

工作：

- 创建 Codex++ Fork。
- 配置 origin/upstream。
- 将当前 UI 原型迁移到独立 feature 目录骨架。
- 添加 workstation crates。
- 建立 Windows CI。
- 不改变业务功能。

验收：

- 上游原测试通过。
- Fork 可编译出 Windows x64 ZIP。
- UI 能在 Tauri 中打开。
- 首个构建物先供用户本人测试，发布候选阶段再按 AGPL 附带对应源码。

## 阶段 2：Codex++ 便携根目录

工作：

- 实现 `CODEXPP_HOME`。
- 从 EXE 解析 `USB_ROOT`。
- 创建正式数据目录。
- 用户脚本、日志、状态和设置进入移动盘。
- 便携模式禁用安装副作用。

验收：

- 中文/空格路径通过。
- 更换盘符后设置仍可读取。
- 电脑用户目录不出现新的 Codex++ 主状态。
- 普通 Codex++ 安装模式兼容。

## 阶段 3：便携 Codex 启动、停止和安全退出

工作：

- 对接 `--app-path`。
- 设置 `CODEX_HOME`/`CODEX_SQLITE_HOME`。
- 一键启动状态机。
- 暴露 stop command。
- 等待 SQLite 和文件句柄。
- 初步安全弹出。

验收：

- 从移动盘启动 Codex。
- 运行至少 15 秒、停止、再次启动。
- CodexData 位于移动盘。
- 停止后无残留 launcher/helper。
- U盘占用时不伪报“可以拔出”。

## 阶段 4：供应商与加密凭据

工作：

- 实现无 PIN 的本地 Secret Store。
- Codex++ RelayProfile 使用 credential reference。
- 配置使用 `env_key`。
- 增加本产品中转站 API Key 便捷粘贴框和自动预设填充。
- 对接模型列表、测试和 Provider Doctor。
- 迁移旧明文凭据。
- 诊断脱敏。

验收：

- API Key 不出现在 settings/config/auth/log/URL/命令行。
- 重启和换电脑后自动加载同一移动盘凭据。
- UI 明确提示无 PIN 模式不能防止移动盘丢失后的密钥提取。
- 错误 Key、401、403、404、429、TLS、DNS 和超时有明确提示。
- Chat Completions 转 Responses 可用。

## 阶段 5：项目、会话、插件与页面增强

工作：

- Workspace 项目索引。
- 按 cwd 聚合会话。
- 绘画/生成图片索引。
- 插件、Skill、用户脚本便携。
- 页面增强迁移。
- Zed Remote 和 upstream worktree 页面。

验收：

- 同一移动盘换电脑能看到相同项目、会话和插件启用状态。
- 页面增强失败时 Codex 仍能使用。
- 项目移动和会话导出不破坏 SQLite。

## 阶段 6：Codex++ 概览、中转站数据和无账号卡密授权

工作：

- 部署 Sub2API staging。
- 首页接入 Codex++ 概览聚合。
- 实现可选中转站用量 adapter。
- 实现 activation-bridge。
- 卡密激活、移动工作站绑定、续签、撤销和签名 entitlement。
- 远程详情 WebView 隔离。
- 卡密和订阅权益映射。
- 建立独立管理员后台。

验收：

- 工作站授权不要求注册或登录账号。
- 客户端不持有管理员 Key。
- 卡密首次激活、到期、绑定上限、撤销和重放测试通过。
- 首页无需登录即可显示 Codex++ 概览；配置中转站后显示真实用量和套餐。
- 管理后台可以生成卡密并查看绑定工作站和服务状态。
- 远程网站不能调用本地危险 Tauri Commands。

## 阶段 7：工作站服务连接

工作：

- 自有 API 域名和接入网关。
- 签名 route manifest。
- Connection Sidecar。
- 应用级环境注入。
- 启动、停止、过期和故障切换。

验收：

- 不修改 Windows 全局代理。
- 浏览器和其他应用流量不经过 sidecar。
- sidecar 拒绝非白名单目标。
- 权益过期后停止新连接，但不锁死本地项目。
- 当前使用本产品中转站套餐时，到期后停用该 Provider；管理 UI 仍允许切换到用户自己的 API。

## 阶段 8：统一更新与回滚

工作：

- 创建自己的 GitHub Release 更新项目，并为国内可达性保留镜像接口。
- schema、签名和 compatibility matrix。
- 独立 Updater。
- 下载、原子替换、健康检查和回滚。
- GitHub Actions 上游监控。

验收：

- 成功更新。
- 断网继续旧版。
- 损坏下载、错误签名、磁盘不足、健康失败均不破坏旧版。
- `Data/Workspace/Backups` 不被覆盖。

## 阶段 9：恢复点、诊断和发布候选

工作：

- latest/previous/emergency 恢复点。
- 异常拔盘检测。
- 诊断包和 secret scanner。
- 两台 Windows 电脑交替测试。
- 安装维护和恢复出厂设置。

验收：

- 公司电脑停止并弹出，家中电脑继续项目。
- 恢复出厂设置二次确认后删除全部工作站数据并回到首次启动状态。
- 诊断包不包含 secret 和用户代码。
- NTFS/exFAT 行为有明确提示。

## 阶段 10：许可与正式发布

工作：

- AGPL/LGPL 材料。
- 将客户端二进制 Release 与对应源码 commit/tag、构建说明关联。
- 桌面载荷再分发结论。
- 隐私、用户协议和服务边界。
- 产品名称和图标审查。
- 发布签名和最终 Release。

验收：

- 所有商业发布 Gate 有书面结论。
- 对应源码与许可证可获得。
- Release 可复制到移动盘并从零启动。

## 当前下一步

阶段 A 的 Windows 原版基线和阶段 B 的工作站 UI 集成已经完成。客户端当前可以编译，但仍使用电脑本机 Codex++ 数据目录，因此不作为 U 盘测试版运行。

用户明确说“开始阶段 C”后进入 Codex++ 便携根目录开发，对应本计划的阶段 2。

便携 Codex 桌面载荷的商业再分发许可仍是独立发布 Gate，不因 Codex++ AGPL 决策自动解决。
