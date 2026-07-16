# Sub2API 与工作站服务器对接方案

## 1. 角色划分

```text
Codex 便携工作站
  持有移动工作站级签名 entitlement、短期服务令牌和非明文 API Key

activation-bridge
  处理无账号卡密、移动工作站绑定、续签、撤销和 Sub2API 管理接口适配

Sub2API
  用户、API Key、余额、套餐、支付、卡密、用量、模型网关、后台管理
```

客户端不得直接调用 Sub2API 管理接口。

## 2. 首页仪表盘

首页主仪表盘以 Codex++ Overview 和 Workstation Health 为基础，不要求用户登录 Sub2API。

基础数据包括：

```text
Codex/Codex++ 运行状态
移动盘路径、空间和写入状态
版本和更新状态
项目、会话、供应商和插件概览
环境校验和工作站服务连接状态
```

用户已经粘贴并启用本产品中转站 API Key 后，再由 adapter 可选调用：

```text
GET /api/v1/user/profile
GET /api/v1/usage/dashboard/snapshot-v2
GET /api/v1/subscriptions/summary
GET /api/v1/subscriptions/active
GET /v1/sub2api/billing
```

中转站数据统一转换为：

```json
{
  "balance": 0,
  "period_usage": 0,
  "period_quota": 0,
  "requests_today": 0,
  "success_rate": 0,
  "average_latency_ms": 0,
  "model_distribution": [],
  "subscription": {
    "name": "",
    "expires_at": ""
  }
}
```

UI 不直接依赖 Sub2API 原始字段，减少上游升级影响。

## 3. 完整网站详情页

用户要求在软件内部打开完整网站。

推荐方式：

- 新建独立 Tauri `WebviewWindow` 加载服务器域名。
- 该远程 WebView 不注册文件、进程、Shell、更新和 secret 等本地命令权限。
- 域名使用严格 allowlist。
- 禁止远程页面任意跳转到其他域名后仍保留本地能力。
- 登录态使用服务器 HTTPS Cookie，客户端不读取用户网站密码。

不推荐：

- 在主本地 WebView 中直接 `iframe` 远程网站并暴露 Tauri Bridge。
- 把 access token 拼进 URL。
- 将网站账号密码交给启动器保存。

## 4. 无账号卡密授权流程

工作站授权不要求注册或登录工作站账号。Sub2API 当前没有移动工作站卡密绑定接口，因此由 `activation-bridge` 提供。

### 4.1 API 草案

激活卡密：

```text
POST /workstation/v1/licenses/activate
```

请求：

```json
{
  "license_code": "XXXX-XXXX-XXXX-XXXX",
  "installation_id": "ws_...",
  "workstation_name": "My Portable Workstation",
  "client_version": "0.1.0",
  "installation_public_key": "base64..."
}
```

响应：

```json
{
  "license_id": "lic_...",
  "installation_id": "ws_...",
  "activated_at": "2026-07-16T00:00:00Z",
  "expires_at": "2026-08-15T00:00:00Z",
  "max_installations": 3,
  "bound_installations": 1,
  "entitlement": "signed-compact-token",
  "service_access_token": "short-lived",
  "refresh_token": "rotating"
}
```

续签：

```text
POST /workstation/v1/licenses/refresh
```

状态查询：

```text
GET /workstation/v1/licenses/status
```

注销当前移动工作站：

```text
POST /workstation/v1/licenses/deactivate
```

### 4.2 绑定模型

- `installation_id` 绑定移动存储中的工作站实例，而不是当前 Windows 电脑。
- 一个卡密可以绑定多个 `installation_id`。
- 最大绑定数量由管理员后台配置。
- 卡密从第一次成功激活开始计算有效期。
- 达到上限后必须由管理员解绑或按后续规则允许用户自助解绑。
- Windows 电脑变化不消耗新的绑定名额；复制整块移动盘产生相同身份时需要服务端检测并限制并发滥用。

### 4.3 到期行为

- 独立工作站服务卡密到期：停止 Connection Service，不删除本地数据。
- 本产品中转站套餐到期：停止 Connection Service 并停用本产品中转站 Provider。
- 工作站管理 UI 保持可打开，允许用户粘贴自己的第三方 API Key。
- 是否立即关闭正在运行的 Codex 仍需用户最终确认。

### 4.4 安全要求

- 卡密使用密码学安全随机数生成，数据库只保存经过抗暴力破解处理的校验值或安全索引。
- 激活、查询和续签接口必须限速。
- `installation_id` 搭配安装实例公钥。
- entitlement 使用服务器签名，客户端离线验证。
- 短期服务令牌只能访问白名单工作站服务。
- refresh token 轮换，重放旧 token 触发撤销。
- 服务端记录激活、解绑、续签和异常并发审计，不记录完整 API Key。

## 5. Sub2API Adapter

activation-bridge 内部新增：

```text
Sub2APIUserClient
Sub2APIAdminClient
APIKeyProvisioner
LicenseCodeService
InstallationBindingService
EntitlementSigner
DashboardAdapter
```

管理员 Key：

- 仅存在服务器 Secret Manager 或环境变量。
- 不进入镜像、Git、日志和客户端响应。
- 使用最小权限和独立轮换周期。

## 6. 官方中转站供应商

工作站中的“官方中转站”是本产品的默认推荐供应商名称，不应表述为 OpenAI 官方服务。

客户端内置该供应商的非敏感预设：

```text
api_base_url
protocol
推荐模型
供应商显示名称
```

用户从本产品中转站网站复制 API Key，粘贴到工作站便捷输入框。客户端清理空格和常见前缀、自动选择预设、保存 credential reference 并执行连接检测。

卡密授权和模型 API Key 是两套独立凭据：

- 卡密控制工作站服务连接及相关权益。
- API Key 控制模型请求。
- 二者都不得出现在日志、URL 或普通配置中。

## 7. 用户自己的 API

此模式不需要 Sub2API：

- 用户在工作站输入 Base URL、协议、模型和 API Key。
- API Key 加密保存在移动盘。
- Codex++ 负责模型测试、Provider Doctor 和协议转换。
- 若用户选择工作站服务连接套餐，只向 activation-bridge 验证该服务权益。

## 8. 卡密与套餐

工作站卡密由 activation-bridge 管理；可以复用 Sub2API 的兑换码生成思路，但不能强制要求 Sub2API 用户账号。

定义两类权益：

```text
model_service             模型 API 套餐
workstation_connection    工作站服务连接套餐
```

activation-bridge 返回签名结果：

```json
{
  "connection_entitled": true,
  "relay_entitled": true,
  "source": "workstation_code|relay_bundle",
  "max_installations": 3,
  "expires_at": "..."
}
```

客户端不自行计算最终有效期，以服务器签名结果为准。

已确认商业规则：

- 工作站产品/服务暂定约 ¥99，具体包含范围待确认。
- 独立工作站服务卡密暂定 ¥10，有效期单位待确认。
- 有效期从首次激活开始。
- 不支持退款。
- 支持多工作站绑定，但必须设置上限。

## 9. 管理后台

管理员后台保持独立网址，不放进客户端导航。

Sub2API 已具备：

- 用户和 API Key。
- 余额和订阅。
- 卡密生成与兑换记录。
- 用量和模型统计。
- 实时流量、错误、并发、账户可用性和监控。
- 支付订单和套餐管理。

工作站新增后台数据可以放在 activation-bridge：

- 卡密生成、批次、状态、首次激活和到期时间。
- 每张卡密最大绑定数量和已绑定工作站。
- 手动解绑、禁用、续期和备注。
- 客户端版本分布。
- 工作站最后在线时间。
- 服务连接授权状态。
- route manifest 版本。
- 环境检测失败匿名统计。

## 10. 部署建议

```text
Internet
  ↓
CDN/WAF/Load Balancer
  ├─ account.example.com      Sub2API Web
  ├─ api.example.com          Sub2API Gateway
  ├─ workstation.example.com  activation-bridge
  └─ updates.example.com      签名更新与 route manifest
```

生产最少需要：

- HTTPS。
- PostgreSQL。
- Redis。
- 备份。
- 结构化日志和告警。
- 管理接口 IP/身份限制。
- CORS 精确域名。
- CSP 和远程 WebView 域名白名单。

## 11. 上游升级策略

- Sub2API 独立部署，不复制其前后端到客户端。
- 优先通过 adapter 适配上游用户 API。
- activation-bridge 不直接依赖 Sub2API 数据库表结构。
- Sub2API 更新先在 staging 环境跑契约测试。
- 生产升级和回滚独立于工作站客户端更新。
