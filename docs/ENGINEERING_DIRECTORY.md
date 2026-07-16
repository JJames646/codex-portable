# 正式工程目录设计

## 1. 公开客户端仓库

仓库名：`codex-portable-workstation`

```text
codex-portable-workstation/
├─ apps/
│  ├─ workstation-manager/              # Codex++ manager Fork：React + Tauri
│  ├─ workstation-launcher/             # Codex++ silent launcher Fork
│  └─ workstation-updater/               # 独立自更新进程
│
├─ crates/
│  ├─ workstation-core/                  # 状态机、路径、健康检查、错误模型
│  ├─ workstation-portable/              # USB 根目录、文件系统、盘符、弹出
│  ├─ workstation-secrets/               # 同盘 vault key、AES-GCM、Secret Broker
│  ├─ workstation-update/                # 清单、签名、下载、切换、回滚
│  ├─ workstation-server-client/         # activation、dashboard、route config
│  ├─ workstation-connection/            # sidecar 管理与进程环境
│  ├─ codex-plus-core/                   # 来自 Codex++ Fork
│  └─ codex-plus-data/                   # 来自 Codex++ Fork
│
├─ sidecars/
│  └─ connection-service/                # 仅绑定 loopback 的受限服务连接
│
├─ contracts/
│  ├─ activation-api/
│  ├─ update-manifest/
│  ├─ route-manifest/
│  └─ diagnostics/
│
├─ packaging/
│  ├─ workstation-layout/
│  ├─ portable-codex/
│  ├─ release-manifest/
│  └─ licenses/
│
├─ updates/
│  ├─ stable.schema.json
│  ├─ route.schema.json
│  └─ examples/
│
├─ scripts/
│  ├─ bootstrap-windows.ps1
│  ├─ build-client.ps1
│  ├─ build-portable-codex.ps1
│  ├─ package-workstation.ps1
│  ├─ test-portable.ps1
│  ├─ sync-upstreams.ps1
│  └─ generate-update-manifest.ps1
│
├─ tests/
│  ├─ portable-paths/
│  ├─ process-lifecycle/
│  ├─ update-rollback/
│  ├─ secrets-redaction/
│  └─ server-contract/
│
├─ docs/
│  ├─ adr/
│  ├─ plans/
│  ├─ ARCHITECTURE.md
│  ├─ UPSTREAM_AUDIT.md
│  ├─ FEATURE_INTEGRATION_MATRIX.md
│  ├─ CODEXPLUSPLUS_INTEGRATION.md
│  ├─ SUB2API_INTEGRATION.md
│  ├─ NETWORK_AND_SERVER_CONNECTIVITY.md
│  ├─ UPDATE_PROTOCOL.md
│  ├─ SECURITY_MODEL.md
│  ├─ LICENSE_REVIEW.md
│  ├─ PRODUCT_DECISIONS.md
│  └─ DELIVERY_PLAN.md
│
├─ third_party/
│  ├─ patches/
│  └─ README.md
│
├─ .github/workflows/
│  ├─ client-ci.yml
│  ├─ monitor-codexplusplus.yml
│  ├─ monitor-portable-codex.yml
│  ├─ monitor-sub2api.yml
│  ├─ publish-beta.yml
│  ├─ publish-stable.yml
│  └─ rollback-release.yml
│
├─ AGENTS.md
├─ PROJECT_SPEC.md
├─ THIRD_PARTY_SOURCES.md
└─ README.md
```

客户端仓库公开并按 AGPL 提供对应源码。

## 2. 私有服务器仓库

仓库名：`codex-workstation-server`

```text
codex-workstation-server/
├─ activation-bridge/
│  ├─ cmd/server/
│  ├─ internal/api/
│  ├─ internal/auth/
│  ├─ internal/license/
│  ├─ internal/installations/
│  ├─ internal/entitlement/
│  ├─ internal/sub2api/
│  ├─ migrations/
│  ├─ Dockerfile
│  └─ tests/
├─ admin-web/
│  ├─ src/pages/licenses/
│  ├─ src/pages/installations/
│  ├─ src/pages/service-health/
│  └─ Dockerfile
├─ deploy/
│  ├─ docker-compose.yml
│  ├─ nginx/
│  └─ env.example
├─ contracts/
├─ .github/workflows/
│  ├─ server-ci.yml
│  └─ deploy-staging.yml
└─ README.md
```

服务器仓库默认私有。管理员密钥、数据库密码、更新签名私钥和生产 `.env` 不进入仓库。

## 3. 为什么按此拆分

- `apps/` 只放可执行程序。
- `crates/` 将工作站新增能力和 Codex++ 上游能力分开，便于同步上游和审计补丁。
- `sidecars/` 独立承载受限连接进程，避免把网络会话与 UI 生命周期混在一起。
- `codex-workstation-server` 与公开 AGPL 客户端仓库分离，不进入客户端发布包。
- `contracts/` 先定义客户端与服务器、更新源之间的稳定边界。
- `packaging/` 只负责组装，不保存用户数据。
- `third_party/patches` 记录所有上游改动，避免散落在不可追踪目录。

## 4. 自己的更新发布仓库

仓库名：`codex-workstation-releases`

```text
codex-workstation-releases/
├─ updates/
│  ├─ stable.json
│  ├─ beta.json
│  └─ revoked.json
├─ compatibility/
│  └─ matrix.json
├─ release-notes/
├─ public-keys/
├─ THIRD_PARTY_NOTICES.md
└─ README.md
```

Release assets：

```text
workstation-manager-win-x64-<version>.zip
portable-codex-<version>.zip
connection-service-win-x64-<version>.zip
runtime-assets-<version>.zip
workstation-bundle-<version>.zip
SHA256SUMS.txt
SIGNATURES.json
CHANGELOG.md
THIRD_PARTY_NOTICES.md
```

## 5. Codex++ Fork 结构

建议直接 Fork `BigPizzaV3/CodexPlusPlus`，不要复制源码后失去历史。

远端：

```text
origin   = 你的 Codex++ Fork
upstream = https://github.com/BigPizzaV3/CodexPlusPlus.git
```

分支：

```text
main                         跟踪自己的稳定集成
upstream-sync/<date>         上游同步验证
codex/portable-root          CODEXPP_HOME 与路径改造
codex/secret-reference       env_key 与加密凭据适配
codex/workstation-shell      当前 UI 和工作站模块
codex/update-control-plane   统一更新入口
```

## 6. 运行时移动盘目录

正式 Release 目录与源码目录分离：

```text
release/Codex-Workstation/
├─ Codex便携工作站.exe
├─ workstation.json
├─ manifest.json
├─ App/
├─ Data/
├─ Workspace/
├─ Backups/
├─ Updater/
├─ Diagnostics/
├─ Logs/
└─ Licenses/
```

更新器只允许修改 `App/`、自身版本清单和只读运行时资源，不得覆盖 `Data/`、`Workspace/` 和 `Backups/`。
