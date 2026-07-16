# 工作站服务连接与服务器可达性方案

## 1. 产品边界

本模块的目标是让“从便携工作站启动的 Codex”稳定访问你拥有并授权的模型 API、设备授权和更新服务。

它不是：

- 通用 VPN。
- 机场客户端。
- 任意目标 SOCKS/HTTP 代理。
- 用于绕过法律、监管、账号权限或服务条款的工具。

## 2. 推荐的连接方案

### 方案 A：直接 HTTPS 访问自有 API 域名（第一选择）

```text
Codex
→ https://api.example.com/v1
→ Sub2API
→ 合法授权的上游模型服务
```

如果国内用户可以稳定访问你的域名，就不需要客户端“节点”功能。

优点：

- 架构最简单。
- 不修改电脑网络。
- 无本地通用代理。
- SSE/WebSocket 可以直接由 API 网关处理。

### 方案 B：受限本地 sidecar + 自有接入网关（推荐备用）

```text
Codex
→ 127.0.0.1:<随机端口>
→ Connection Sidecar
→ https://access.example.com
→ Sub2API
```

Sidecar 的价值：

- 不把真实 API Key 写入 Codex 配置。
- 统一附加设备短期令牌。
- 只对子进程生效。
- 对目标域名做固定白名单。
- 可在多个自有接入网关间健康切换。

它仍不是通用 VPN。

### 方案 C：通用隧道/机场节点

不纳入本项目。

原因：

- 范围超出 Codex 工作站。
- 容易产生法规、运营许可、上游条款和安全风险。
- 任意流量转发会扩大攻击面和滥用风险。

## 3. 国内用户访问境外服务器

推荐顺序：

1. 为 Sub2API 和 activation-bridge 使用稳定的自有 HTTPS 域名。
2. 选择支持长连接、SSE 和 WebSocket 的合规 CDN/WAF/API 加速服务。
3. 根据目标市场和法律意见选择服务器区域。
4. 如果使用国内基础设施，按实际业务完成备案、数据和电信业务相关评估。
5. 如果主服务器在境外，可以增加自有边缘接入网关，但该网关只反向代理本产品 API 域名。

客户端无法合法地“保证任何网络环境都能访问所有境外服务”。正式产品应把：

- 自有 API 服务可达性。
- OpenAI/第三方官方网站可达性。

区分开。前者由你的系统负责，后者取决于用户所在网络和相关服务政策。

## 4. Route Manifest

客户端启动后向：

```text
https://updates.example.com/routes/stable.json
```

获取签名清单。

示例：

```json
{
  "schema_version": 1,
  "manifest_version": "2026.07.16.1",
  "issued_at": "2026-07-16T00:00:00Z",
  "expires_at": "2026-07-17T00:00:00Z",
  "routes": [
    {
      "id": "primary",
      "region": "default",
      "api_base_url": "https://api.example.com/v1",
      "activation_base_url": "https://workstation.example.com",
      "health_url": "https://api.example.com/health",
      "priority": 100,
      "protocols": ["https", "sse", "websocket"]
    }
  ],
  "signature": "base64-ed25519-signature"
}
```

清单不得包含：

- 通用代理账号密码。
- 第三方机场订阅。
- 可访问任意目标的节点配置。
- 服务器管理员秘密。

## 5. 选择与故障切换

客户端：

1. 验证清单签名和有效期。
2. 只保留 HTTPS 且 host 在产品白名单中的路由。
3. 并发执行轻量健康探测。
4. 按 priority、延迟和最近失败次数选择。
5. 运行中只在明确连接错误时切换。
6. 不在一次流式响应中间无缝切换上游。

路由选择状态可以在 UI 显示为：

```text
服务连接：已连接
```

普通用户界面不需要显示物理节点、IP 或运营商线路。

## 6. Sidecar 安全规则

- 监听 `127.0.0.1`，随机高位端口。
- 每次启动生成本地会话 token。
- Codex 请求必须带本地会话标识或只允许父子进程连接。
- 目标 host 固定 allowlist。
- 禁止 CONNECT 任意 host。
- 限制请求体大小、并发和空闲时间。
- 不持久化请求正文、代码和图片。
- 日志只记录请求 ID、状态码、耗时和脱敏错误。
- 停止 Codex 后立即停止或锁定 sidecar。

## 7. Codex 进程注入

优先使用 provider `env_key`：

```text
CODEX_WORKSTATION_API_KEY=<临时解密值>
```

如果采用本地 sidecar，则 Codex 配置可指向：

```text
http://127.0.0.1:<port>/v1
```

但本地 sidecar 到服务器必须使用 HTTPS。

不设置：

- Windows 全局代理。
- 用户永久环境变量。
- 浏览器代理。
- 系统路由表。

## 8. 权益逻辑

```text
官方中转站套餐用户
→ activation-bridge 返回 connection_entitled=true

用户自己的 API + 独立卡密
→ activation-bridge 验证无账号工作站卡密
→ 返回签名 entitlement 和有效期

权益过期
→ 停止新的受限连接会话
→ 如果使用本产品中转站套餐，同时停用该 Provider
→ 本地项目和无需该服务的功能仍可使用
```

## 9. 合规与运营提示

UI 声明只能起提示作用，不能替代实际合规设计。

正式上线前应评估：

- 服务器和 CDN 所在地要求。
- 数据跨境和日志留存。
- 网络接入、内容安全和电信业务相关要求。
- 模型供应商服务条款。
- 用户隐私政策和数据处理协议。

本文件是工程设计，不构成法律意见。
