# Pixiv Func Replica v1 技术与任务设计

## 1. Design objective

该父任务定义完整 Replica v1 的架构边界、子任务依赖与集成验收，不直接作为大规模编码目标。实现以小步子任务推进，任何阶段都必须保持仓库可分析、可测试、可构建，并且不以占位路径伪造业务完成。

## 2. Sources of truth

1. 用户可感知行为：`svenfuss/pixiv_func_mobile@c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`。
2. 产品边界与现代化约束：本任务 `prd.md`。
3. 用户原始指令覆盖关系：`research/requirements-traceability.md`；固定源码路径索引：`research/beta56-source-map.md`。
4. 当前实现事实：仓库代码、测试、Android 配置和已完成的 Trellis 子任务。
5. 当前平台/API 事实：对应子任务开始时核验的 Flutter、Android、Pixiv 或可信开源客户端资料。

历史源码只决定 Replica 行为，不决定继续使用不安全、不可维护或不兼容的内部实现。

## 3. Task hierarchy and ownership

- 父任务维护完整范围、任务地图、跨子任务契约和最终集成审查。
- 已有 `08-26-flutter-android-scaffold` 作为工程基线子任务纳入父任务，仍是唯一 `in_progress` 任务。
- 其余 32 个任务已经创建并完成 PRD/design/implement：6 个中间父任务协调范围，26 个叶子任务承担独立实现。
- 所有新任务保持 `planning`；每个叶子必须具有可独立验证的行为边界、自己的 start 审批、质量检查、提交和归档。
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

业务请求不得接受 arbitrary absolute `next_url`。分页层只解析允许 host 的 endpoint 和已知 query 参数，再经统一 client 发起请求。

### 4.4 Entities, stores and paging

- `core/entity/illust_entity.dart` 保存共享作品实体。
- `IllustStore` 以作品 ID 为键，合并来自推荐、详情、搜索和排行榜的数据。
- `BookmarkStore` 维护作品收藏状态及 pending/error 状态，并在 API 成功后更新共享状态。
- `core/paging/page.dart` 与 `paged_feed_controller.dart` 表达 initial/refresh/load-more/error 的独立状态和 ID 去重。
- Feed/controller 尽量保存 ID 列表，不复制可漂移的作品对象。

这一边界保证 Recommended 与 Detail 不会各自维护互相冲突的 bookmark 副本。

### 4.5 Features

各 feature 拥有页面、controller 和 feature-specific repository adapter；跨页面实体、认证、网络、下载和数据库能力来自 core service/store。功能扩展顺序严格遵守第一条链优先，后续 discovery、social、novel/media、platform 和 late-parity 任务不能反向污染基础层。

### 4.6 Media and persistence

- 下载器由共享连接池、并发受限队列、流式文件 sink、节流进度和唯一完成状态组成；Android 文件提交通过 MediaStore adapter。
- Ugoira 将 ZIP 和 frame 数据落盘，内存只保留有界解码窗口；播放器生命周期与页面可见性绑定。
- History/database 采用应用级单例生命周期和紧凑索引 schema；任何 migration 必须有版本和回滚/恢复说明。

### 4.7 Android platform boundary

- Manifest、intent filter、FileProvider、flavor 权限和 MainActivity/native bridge 只处理 Android 平台契约。
- Flutter 层负责解析成受类型约束的导航/业务命令；外部 URI、文件和 intent 数据在边界验证。
- predictive back、root double-back 和页面 PopScope 形成单一回退策略，避免每个页面各自实现冲突计时。
- WebView compatibility proxy 使用 AndroidX WebKit capability detection；能力不足时明确失败，不扩大代理范围。

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

## 6. Compatibility and migration

- 已有 UI shell 作为可见行为起点，小步替换 placeholder，不用 stock Flutter template 覆盖现有 `lib/`。
- 账号数据升级采用版本化 schema；原版剪贴板格式只在受控迁移入口解析，解析后立即转入安全存储。
- compatibility networking 默认关闭，不阻塞 Normal 路径；旧固定 IP 只能作为显式 emergency route。
- GitHub 与 F-Droid flavor 在 Manifest merge 层隔离 updater 权限，基础 Manifest 保持最小权限。

## 7. Trade-offs

- 选择集中 entity/store 而不是页面复制模型，增加初期基础层工作，但换来跨页面状态一致和可测试性。
- 选择逐个子任务完成而不是横向铺页面，早期功能面较窄，但能更早得到真实可用链路并降低伪完成风险。
- 选择严格 TLS 和 capability-based compatibility，可能使部分特殊网络环境明确失败，但不会用安全降级制造表面可用。
- Replica v1 保留旧 UX，即使存在更现代的交互方案；这些改动留到 Evolution 阶段。

## 8. Rollback and operational boundaries

- 每个子任务使用独立小提交；失败时仅回滚该子任务，不重写历史、不覆盖无关用户改动。
- 数据库、账号 schema、Manifest 权限和 flavor 变化必须在子任务设计中单独列出迁移与回滚点。
- 真实账号、设备、签名或发布操作需要对应权限；缺失时记录未验收，不使用 mock 替代。
- 父任务完成只表示代码与本地/设备验收满足 PRD；任何远程发布仍需要用户单独授权。
