# Codex 便携工作站设计说明

状态：Proposed
日期：2026-07-16

## 设计目标

以 Codex++ 为正式客户端底座，构建一个可从 Windows 移动存储启动的工作站。项目、会话、Skills、插件、Codex 状态和 Codex++ 状态尽量保存在移动盘；服务器承担卡密、工作站绑定、套餐、用量、模型 API、更新和受限路由配置。

## 推荐方案

采用 Codex++ Fork：

- React/Tauri 管理工具承载当前 UI。
- Rust Core 复用 Codex++ 供应商、协议、会话、插件和 CDP 启动能力。
- 新增 workstation crates 处理移动盘、加密、服务器、更新和安全弹出。
- Sub2API 独立部署。
- activation-bridge 提供无账号卡密激活和移动工作站绑定。
- Connection Sidecar 仅访问自有 API 域名。

## 关键数据流

```text
用户启动
→ 解析 USB_ROOT
→ 检测和创建 Data/Workspace
→ 自动加载移动盘凭据
→ 验证服务器 route manifest
→ 启动受限 sidecar
→ 写入 env_key provider 配置
→ 注入 CODEX_HOME/CODEX_SQLITE_HOME/CODEXPP_HOME
→ Codex++ --app-path 启动移动盘 Codex
```

## 替代方案

### WPF + 原版 Codex++

适合要求主启动器闭源的场景，但会出现双 UI、双状态和 IPC 复杂度，且 Codex++ 自身状态依然不便携。

### 新 UI 嵌入 Codex++

没有稳定的管理 API，远程 WebView 权限边界和上游 UI 兼容性较差，不采用。

## 安全设计

- API Key 只进入加密 Vault 和子进程环境。
- Codex provider 使用 `env_key`。
- 客户端不持有 Sub2API 管理员 Key。
- 远程网站 WebView 无本地危险权限。
- 更新和路由清单必须签名。
- Connection Sidecar 不提供任意目标代理。

## 失败处理

- 网络不可用：本地项目和已验证程序继续运行。
- 页面增强失败：降级启动原始 Codex。
- 更新失败：删除 staging 或自动回滚。
- SQLite 占用：不允许伪安全弹出，显示占用信息。
- 异常拔盘：下次启动检测 abnormal-exit 并提供恢复点。

## 设计确认点

进入实现前需要用户确认：

1. 接受 Codex++ Fork 的 AGPL 源码提供义务。（已确认）
2. 正式客户端使用 Rust/Tauri/React。
3. 便携 Codex 应用载荷在获得分发许可前只用于内部测试。
