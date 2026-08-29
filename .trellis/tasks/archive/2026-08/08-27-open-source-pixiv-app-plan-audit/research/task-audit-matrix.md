# Replica v1 17 个直接子任务审查矩阵

## 状态说明

- `archived`：不改写历史任务；新建议写入父任务或 `08-26-replica-v1-integration-release`。
- `planning parent`：本身不实现产品代码；建议下钻到未完成叶子和父级集成 Gate。
- `planning leaf`：可以直接更新 PRD/design/implement，但仍需后续明确 start 批准。

| # | 直接子任务 | 当前状态 | 审查结论 | 决策与规划落点 | 优先级 |
|---:|---|---|---|---|---|
| 1 | `08-26-flutter-android-scaffold` | archived | Flutter 3.47/API 36/AGP 9 基线合理；第三方仓库没有更优的可见体验方案 | 保留现状；集成任务补 SDK/lock/flavor/merged-manifest 可复现证据，不重开 archive | P2 采用 Gate |
| 2 | `08-26-restore-icon-font` | archived | 固定 commit + SHA-256 的资产 provenance 已优于多数样本 | 集成任务检查 APK 内资产、LICENSE/NOTICE 和无 Material fallback；不新增实现 | P2 保留 |
| 3 | `08-26-secure-account-store` | archived | metadata/credential 分离显著优于 pixez/Pix-EzViewer 的账号持久化 | 集成审计普通 prefs/DB/log/crash snapshot，覆盖 Keystore 失效和账号切换；禁止借用第三方账号 schema | P0 采用 Gate |
| 4 | `08-26-oauth-pkce-webview-login` | archived | 一次性 TTL PKCE、严格 callback/TLS 方向正确；外部源码主要提供反例 | 集成复跑 cancel/timeout/dispose/重复 callback 和 release 日志审计；拒绝硬编码 secret、全局 PKCE | P0 采用 Gate |
| 5 | `08-26-pixiv-network-token-refresh` | archived | per-account single-flight 已达到所选项目最佳实践 | 集成补“失败代共享”“旋转 refresh token 原子更新”“账号退出时等待者终止”证据；不改历史计划 | P0 采用 Gate |
| 6 | `08-26-recommended-feed-paging` | archived | ID feed/shared store 正确；存在旧代 response 在 controller 丢弃前先 merge entity 的风险，Recommended 也未向 transport 传 cancel token | 父设计增加 generation-scoped commit；集成任务覆盖 refresh-wins、stale entity/cursor、账号/筛选隔离；首屏 cache 暂缓为 P2 | P0 采用 |
| 7 | `08-26-illust-detail-viewer` | archived | 共享 entity、snapshot→detail merge 和 typed Ugoira route 优于页面私有 store | 集成覆盖旧 feed/detail 交叉返回、viewer/route dispose 和 download group 状态；不复制 pixez page-local mutation | P1 采用 Gate |
| 8 | `08-26-bookmark-state-sync` | archived | non-optimistic、revision 与跨页共享符合 Replica；Shaft durable action queue 不可整套照搬 | 采用 owner/dedupe/Retry-After/superseded failure 测试；拒绝 Replica v1 默认离线自动重放，写入集成 mutation matrix | P1 受限采用 |
| 9 | `08-26-android-platform-parity` | archived | typed intent/FileProvider/MediaStore 边界正确；新的平台风险主要属于尚未实现的 Reverse Image/Widgets/Updater | open leaves 补 WebView challenge、RemoteViews IPC、PendingIntent uniqueness、APK signer；集成检查基础 Manifest 不被污染 | P0 采用 Gate |
| 10 | `08-26-discovery-search-reverse-image` | planning parent；3/4 leaves archived | Ranking/New/Search 复用共享 store 是优势；需补 feed generation；Reverse Image 的 structured API 假设过强 | 更新父协调 Gate；直接更新 `08-26-reverse-image-search` 为 capability-based provider，WebView 差异需再审批 | P0 采用 |
| 11 | `08-26-profile-follow-social` | planning parent；1/2 leaves archived | FollowStore 共享状态合理；Profile edit 应避免字段组合分支和本地密码 | 更新父协调 Gate；`08-26-profile-edit` 加 `ProfileCapabilities`、typed patch、verification-pending 和组合测试 | P0 采用 |
| 12 | `08-26-comments-history-settings` | planning parent；3/3 leaves archived | 分离 repository/schema 的现有规划优于混合大 JSON；需要补恢复/所有权和 mutation retry 边界 | 父任务增加 DB migration/downgrade、restore/dispose、非幂等写操作不自动重放；具体缺陷由集成创建修复任务 | P1 受限采用 |
| 13 | `08-26-novel-reader` | archived | stable anchor、可取消布局已有优势；typed markup 不完整，UI isolate 测量预算仍需证据 | 集成任务列为 P0 hardening：补 newpage/chapter/jump/image tokens、chunk/budget、未知标记与长文测试，不改 archive | P0 采用 |
| 14 | `08-26-downloads-ugoira-media` | planning parent；1/2 leaves archived | streaming/MediaStore 基础正确；task-group、submission snapshot、restart policy 与 archive/pixel caps 可更明确 | 更新父 Gate；直接补强 `08-26-ugoira-player-export`；DownloadManager 跨重启保证先作 P1 决策，至少无幽灵 running/pending item | P0/P1 采用 |
| 15 | `08-26-compat-network-account-migration` | planning parent；2 leaves planning | 大陆无外部代理已成为产品目标；源码表明 ECH/failure taxonomy/分目标诊断值得采用，也暴露当前 strict CONNECT 只解决 DNS、不能单独隐藏 SNI且未覆盖全部 transport。Clipboard 零交互无法提供攻击者级真实性 | network 提升 P0 并前置：shared exact-host transport、direct-first、DoH/direct connector、ECH/WebView capability gate、大陆设备矩阵；拒绝关证书/SNI hack/固定 IP/反代。Clipboard 保持真实性边界 | P0 采用/受决策约束 |
| 16 | `08-26-live-widgets-updater` | planning parent；3 leaves planning | 未找到可信当前 Live 实现；Widgets 和 Updater 有最明确的可借鉴成熟模式 | 更新三个 leaves：Live 两阶段 feasibility；Widget versioned secret-free snapshot + WorkManager/IPC；Updater signed manifest + hash/package/signer + flavor compile-time isolation | P0 采用 |
| 17 | `08-26-replica-v1-integration-release` | planning leaf | 原计划覆盖广，但尚未把上述开源证据转为可执行 release blocker | 增加 feed generation、mutation、Novel markup、download recovery、Ugoira limits、Reverse/Profile、Widget、signed updater 和“不存在 Live 证据”的硬化矩阵 | P0 采用 |

## 下钻后的开放实现叶子写回

| 开放叶子 | 主要修改 |
|---|---|
| `08-26-reverse-image-search` | provider capability 结果、interactive WebView 只是候选且差异需审批、cloud URI/off-main/cleanup 和 challenge 设备证据 |
| `08-26-profile-edit` | server capabilities、typed dirty-field patch、verification pending、密码短生命周期、字段组合矩阵 |
| `08-26-ugoira-player-export` | typed archive/frame budgets、header-before-decode、sliding window、owned output commit、终态 cleanup |
| `08-26-restricted-compat-network` | 大陆无外部代理 P0：覆盖 API/OAuth/image/download/Ugoira/WebView/Widget 的 shared transport；direct-first、failure taxonomy、DoH direct connector、ECH capability、可选 loopback、大陆运营商矩阵；拒绝固定 IP/SNI/证书/反代降级 |
| `08-26-secure-clipboard-account-migration` | checksum 只能防损坏、nonce 只能提供目标设备本地 replay 保护、零交互不提供 authenticity/confidentiality |
| `08-26-live-player` | endpoint/HLS 证据先于 player 依赖；无证据即 blocker，不留 mock |
| `08-26-android-home-widgets` | versioned account snapshot、headless Flutter proof、unique work、last-good/account-invalid、bitmap IPC/PendingIntent limits |
| `08-26-updater-flavors` | signed manifest trust root、mandatory size/hash/package/signer、single-flight/resume、F-Droid 编译期剔除 |
| `08-26-replica-v1-integration-release` | 将已归档任务的新缺口统一变为 release blocker，并保留“发现缺陷后创建 owning fix task”的诚实路由 |

## 计数检查

本文件主表恰好列出父任务 `task.json.children` 的 17 个直接子任务；另下钻原 Replica 树的 9 个未归档实现叶子，其中 `08-26-ugoira-player-export` 为 `in_progress`，其余 8 个仍为 `planning`。新增的 5 个 hardening leaves 在下节单独计数。没有改写任何 archive artifact，也没有将调研建议记为已实现。

## 新增 owning hardening 任务

开源审查后，用户已允许把归档缺口拆成独立规划任务。新增 top-level `08-27-replica-v1-hardening` 不计入原 17 项 direct-child matrix，当前保持 `planning`，并阻塞 `08-26-replica-v1-integration-release` 的对应证据 Gate：

| Hardening leaf | 归档/实现边界 | 依赖与优先级 |
|---|---|---|
| `08-27-feed-generation-commit-hardening` | Recommended/Ranking/New/Search/Profile 的 generation、entity、cursor commit | `08-26-restricted-compat-network`，P0 |
| `08-27-mutation-ownership-hardening` | Bookmark/Follow/Comments/Profile mutation owner、dedupe、429、取消 | `08-26-restricted-compat-network`，P1 |
| `08-27-novel-markup-hardening` | Novel typed markup、unknown preservation、长文预算和取消 | `08-26-novel-reader`，P0 |
| `08-27-media-job-recovery-hardening` | Download/Ugoira group、restart、pending cleanup、资源上限 | Ugoira 完成 + network，P1 |
| `08-27-android-platform-boundary-hardening` | WebView capability/loopback、intent、FileProvider/MediaStore lifecycle | network + OAuth/account，P0 |

叶子都已补齐 `prd.md`、`design.md` 和 `implement.md`，但未运行 `task.py start`；媒体叶子明确等待当前 `08-26-ugoira-player-export` 完成，归档目录继续只读。
