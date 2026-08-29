# Pixiv Func Replica v1 技术与任务设计

## 1. Design objective

该父任务定义完整 Replica v1 的架构边界、子任务依赖与集成验收，不直接作为大规模编码目标。实现以小步子任务推进，任何阶段都必须保持仓库可分析、可测试、可构建，并且不以占位路径伪造业务完成。

## 2. Sources of truth

1. 用户可感知行为：`svenfuss/pixiv_func_mobile@c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`。
2. 产品边界与现代化约束：本任务 `prd.md`。
3. 用户原始指令覆盖关系：`research/requirements-traceability.md`；固定源码路径索引：`research/beta56-source-map.md`。
4. 当前实现事实：仓库代码、测试、Android 配置和已完成的 Trellis 子任务。
5. 当前平台/API 事实：对应子任务开始时核验的 Flutter、Android、Pixiv 或可信开源客户端资料。
6. 开源实现审查：`../08-27-open-source-pixiv-app-plan-audit/research/` 固定 commit 证据与 17 项矩阵；它只能改进内部状态机、失败边界和测试，不覆盖前五项。

历史源码只决定 Replica 行为，不决定继续使用不安全、不可维护或不兼容的内部实现。

## 3. Task hierarchy and ownership

- 父任务维护完整范围、任务地图、跨子任务契约和最终集成审查。
- 原 Replica 树当前共有 17 个直接子任务、6 个中间父任务和 27 个实现叶子；截至 2026-08-27 已归档 18 个实现叶子，`08-26-ugoira-player-export` 为 1 个 `in_progress` 叶子，另有 8 个原树实现叶子仍为 `planning`。独立 hardening parent 另有 5 个 planning leaves。
- 已归档任务保持历史只读；开放叶子继续具有可独立验证的行为边界、自己的 start 审批、质量检查、提交和归档。归档状态与真机/真实 API/最终业务接受分别记录。
- 归档实现的新缺口由 top-level `08-27-replica-v1-hardening` 协调，五个 owning leaves 各自拥有代码和测试；它们不改变本父任务 17 个 direct children，但在最终集成前必须逐项关闭。
- 父子关系只表达范围归属；实际依赖必须同时写进子任务 PRD/implement，不能依赖目录顺序暗示。
- 中间父任务不直接修改产品代码；其叶子归档后仅汇总跨叶子验收并归档。

## 4. Application boundaries

### 4.1 App shell

- `lib/app/` 负责主题、导航、图标和共享 Replica widgets。
- `lib/features/onboarding/` 负责首次引导；导航动画和回退行为由统一 Replica 路由契约控制。
- Home 保持一个 BottomAppBar 和共享 tab 状态，不为每个 tab 引入独立 Navigator；tab 返回时保持数据与滚动位置。

### 4.2 Identity and authentication

- `core/auth/account.dart`：非秘密账号 metadata 与认证状态。
- `core/auth/credential_store.dart`：安全存储访问与凭据生命周期。
- `core/auth/account_store.dart`：多账号、当前账号、切换、退出、凭据更新和 re-auth 状态。
- `core/auth/oauth_service.dart`：PKCE 会话、WebView authorize URL、回调验证和 token exchange。
- `core/auth/token_refresh_gate.dart`：按账号 single-flight refresh 与最多一次请求重试契约。

设置和非敏感 metadata 可以进入 SharedPreferencesAsync 或后续数据库；token、cookie、verifier 等秘密禁止进入普通偏好、日志和 UI 状态快照。

### 4.3 Network

- `core/network/pixiv_client_identity.dart` 集中维护经核验的客户端身份。
- `core/network/pixiv_headers.dart` 生成时间相关请求头，不由 feature 拼接。
- `core/network/network_route_policy.dart` 将 Normal 与 Compatibility 路由隔离。
- `core/network/pixiv_http_client.dart` 负责 TLS、认证、刷新协作、取消、错误归一化与一次重试上限。
- `core/network/api_error.dart` 提供可观察且可测试的错误分类。
- `PixivDestinationRegistry + NetworkAccessPolicy` 以 exact host/purpose 管理 API、OAuth、accounts Web、pximg 和 Pixiv Web；默认 `automatic` direct-first，并提供 `directOnly`。
- `SecureResolver + NativeStrictConnector` 把候选地址、TTL、network revision 与取消绑定；可连接候选 IP，但必须以原 URL host 建立系统信任链 TLS。DoH resolver 只对 registry 允许的 Pixiv 主机生效。
- 一个 shared transport factory 同时注入 `PixivHttpClient`、`OAuthService`、download/Ugoira 与 project-owned CachedNetworkImage file service；禁止全局 `HttpOverrides`，非 Pixiv provider 不受影响。
- `TransportFailureClassifier` 允许 DNS/connect/timeout/reset/TLS-handshake 等 eligible failure 选择另一条严格 route；证书/主机名不匹配是中间人信号，永远终止且不重试。HTTP/auth/429/parse/cancel 同样不触发降级，Mutation/OAuth body 不作不确定自动重放。

业务请求不得接受 arbitrary absolute `next_url`。分页层只解析允许 host 的 endpoint 和已知 query 参数，再经统一 client 发起请求。

### 4.4 Entities, stores and paging

- `core/entity/illust_entity.dart` 保存共享作品实体。
- `IllustStore` 以作品 ID 为键，合并来自推荐、详情、搜索和排行榜的数据。
- `BookmarkStore` 维护作品收藏状态及 pending/error 状态，并在 API 成功后更新共享状态。
- `core/paging/page.dart` 与 `paged_feed_controller.dart` 表达 initial/refresh/load-more/error 的独立状态和 ID 去重。
- Feed/controller 尽量保存 ID 列表，不复制可漂移的作品对象。
- 每次 initial/refresh/append 都携带 generation context；repository 只有在 context 仍活动时才可以把 parsed entity 和 next cursor 作为同一次提交写回。仅在 controller 层丢弃旧 ID 列表不足以阻止旧 entity snapshot 污染共享 store。
- 可选首屏 cache 必须按账号、feed key、响应 schema 和年龄隔离，并有 payload/LRU 上限；它是设备基准证明收益后的优化，不是业务成功路径。

这一边界保证 Recommended 与 Detail 不会各自维护互相冲突的 bookmark 副本。

### 4.5 Features

各 feature 拥有页面、controller 和 feature-specific repository adapter；跨页面实体、认证、网络、下载和数据库能力来自 core service/store。功能扩展顺序严格遵守第一条链优先，后续 discovery、social、novel/media、platform 和 late-parity 任务不能反向污染基础层。

### 4.6 Media and persistence

- 下载器由共享连接池、并发受限队列、流式文件 sink、节流进度和唯一完成状态组成；Android 文件提交通过 MediaStore adapter。
- Ugoira 将 ZIP 和 frame 数据落盘，内存只保留有界解码窗口；播放器生命周期与页面可见性绑定。
- History/database 采用应用级单例生命周期和紧凑索引 schema；任何 migration 必须有版本和回滚/恢复说明。
- 多页作品和 Ugoira 是一个 user-visible job 下的多个 task；命名/输出/设置在提交时冻结，group 后处理恰好一次。临时输出只有成功后 commit，cleanup 只能删除该 task 明确拥有的资源。
- 小型状态文件使用原子替换；大媒体继续流式写入。进程恢复必须把遗留 running 转为明确 queued/failed/canceled 策略，并清理或接管 MediaStore pending item。

### 4.7 Android platform boundary

- Manifest、intent filter、FileProvider、flavor 权限和 MainActivity/native bridge 只处理 Android 平台契约。
- Flutter 层负责解析成受类型约束的导航/业务命令；外部 URI、文件和 intent 数据在边界验证。
- predictive back、root double-back 和页面 PopScope 形成单一回退策略，避免每个页面各自实现冲突计时。
- 登录 WebView 使用平台严格直连，不做导航拒绝，也不承担大陆直连责任——首次登录已不在直连范围内。能在这个 WebView 里加载出来的本来就是 Pixiv 自己选择的登录端点。
- Widget bridge 默认只暴露版本化、账号 revision 绑定的非秘密 render snapshot；RemoteViews bitmap 有 IPC/pixel budget，瞬时网络失败与账号失效采用不同的 last-good 清理策略。
- GitHub updater 的信任根是编译进 flavor 的公钥而非未签名 GitHub JSON；F-Droid flavor 在 build graph/Manifest/component 层没有 updater 能力。

## 5. Critical data flows

### 5.1 Cold start

`SettingsController + AccountStore hydration → StartupGate → Welcome | Login | Home`

在 hydration 完成前只显示明确启动状态；读取失败必须可观察，不得静默假设“无账号”。

### 5.2 Login

`Login action → create one-use PKCE session → WebView → validate pixiv://account callback → token exchange → secure credential write → AccountStore current account → Home`

取消、超时、非法回调、证书错误或 exchange 失败都清理 verifier，并保留在可重试的 Login 状态。

### 5.3 Authenticated request

`Feature → PixivHttpClient(account snapshot) → response/auth failure → TokenRefreshGate(account, old token) → compare current token → await/start one refresh → retry once or mark re-auth`

### 5.4 Shared bookmark

`Card/detail action → BookmarkStore pending → Pixiv API → success updates entity-keyed state | failure restores previous state`，所有订阅同一作品 ID 的页面同步重建。

### 5.5 Download

`User selection → queue → streaming HTTP → temp/MediaStore pending item → successful finalize → one completion event`；取消或失败清理 pending output 并保留明确错误。

### 5.6 Mainland access without external proxy/VPN

读取型原生流量：`exact Pixiv destination → direct HTTPS → eligible failure → DoH candidate + original-host TLS → response/error`。

写操作/OAuth：先用 `(host, route)` 的路由记忆或严格 preflight 选择单一路径，再发送 body 一次；不能在发送状态不确定后自动跨 route 重放。

大陆验收按 API/OAuth/image/download 分出口记录，不能用 API 成功替代图片或下载成功。首次登录不在直连范围内，由剪贴板账号迁移承担。

## 6. Compatibility and migration

- 已有 UI shell 作为可见行为起点，小步替换 placeholder，不用 stock Flutter template 覆盖现有 `lib/`。
- 账号数据升级采用版本化 schema；原版剪贴板格式只在受控迁移入口解析，解析后立即转入安全存储。
- compatibility networking 始终 direct-first，`DirectOnly` 永远可选。省 SNI 在自行完成链验证与 SAN 核对的前提下允许；固定 IP 只作解析失败后的兜底；关闭证书校验与第三方反代继续拒绝。
- GitHub 与 F-Droid flavor 在 Manifest merge 层隔离 updater 权限，基础 Manifest 保持最小权限。

## 7. Trade-offs

- 选择集中 entity/store 而不是页面复制模型，增加初期基础层工作，但换来跨页面状态一致和可测试性。
- 选择逐个子任务完成而不是横向铺页面，早期功能面较窄，但能更早得到真实可用链路并降低伪完成风险。
- 选择严格 TLS 和 capability-based compatibility，可能使部分特殊网络环境明确失败，但不会用安全降级制造表面可用。
- 选择 shared host-scoped transport 与大陆真实网络矩阵会增加平台适配和设备成本，但避免只修 API、登录/图片仍失败却声称“大陆可用”。
- Replica v1 保留旧 UX，即使存在更现代的交互方案；这些改动留到 Evolution 阶段。

## 8. Rollback and operational boundaries

- 每个子任务使用独立小提交；失败时仅回滚该子任务，不重写历史、不覆盖无关用户改动。
- 数据库、账号 schema、Manifest 权限和 flavor 变化必须在子任务设计中单独列出迁移与回滚点。
- 真实账号、设备、签名或发布操作需要对应权限；缺失时记录未验收，不使用 mock 替代。
- 父任务完成只表示代码与本地/设备验收满足 PRD；任何远程发布仍需要用户单独授权。
