# 完成 Pixiv Func Replica v1 全量复刻

## Goal

以 Flutter 3.47、Android API 36 和现代安全实现，重新实现已停止维护的 Pixiv Func；Replica v1 冻结原版用户可感知体验，达到可真实登录、浏览、交互、下载和在 Android 真机运行的完整复刻状态，并以“中国大陆用户无需外部代理/VPN仍能尽可能使用 Pixiv”作为一等产品目标。

## Background and confirmed facts

- 用户可感知行为以 `svenfuss/pixiv_func_mobile` 的 `c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`（`1.0.0-beta56+62`，2023-06-21）为主要事实来源。
- 可见体验包括页面结构、导航层级、手势、按钮位置、信息密度、颜色、字号、动画、loading/toast、返回与长按行为、收藏/关注交互和页面状态恢复；Replica v1 不主动重新设计这些行为。
- 内部实现允许现代化重写，包括 OAuth、Token 刷新、网络、状态管理、缓存、数据库、下载、Ugoira、WebView、Android 原生桥接、生命周期和安全存储。
- 截至 2026-08-27，原 Replica 树的 27 个实现叶子已有 18 个归档：工程/icon、账号/OAuth/网络、Recommended/Detail/Bookmark、Android platform、Ranking/New/Search、User/Profile/Follow、Settings/Comments/History、Novel 与 DownloadManager；归档表示对应任务流程完成，不自动等于全部真机/真实 API/最终业务验收通过。另有 5 个独立 hardening leaves，不计入这 27 项。
- 当前仓库已具有真实 StartupGate/AccountStore/OAuth/网络、共享实体与分页、主要内容页面、下载/MediaStore 和 Android intent 边界；原树剩余范围中 Ugoira 已进入 `in_progress`，Reverse Image、Profile edit、Compatibility network、Clipboard migration、Live、Widgets、Updater 与最终 Integration Release 共 8 个叶子仍为 `planning`。
- 仍缺少的跨功能证据和完成后发现的 hardening 缺口由开放叶子与最终集成任务承接，不通过改写 archive 记录伪装为已实现。
- 开源审查后已创建独立 top-level `08-27-replica-v1-hardening` 及五个 owning leaves；它们不加入本父任务原有 17 个 direct children，但其证据是本父任务和最终集成的 release blocker。
- Trellis 使用 `codex.dispatch_mode: inline`；实现和检查由主会话直接完成，不使用 subagent。
- 2026-08-27 已以固定 commit 审查 Pixiv-Shaft、pixez-flutter、Pix-EzViewer、Pixeval 与 PixivBiu，并对 17 个直接子任务完成状态感知映射；第三方源码只补强内部实现与验证，不改变 beta56 可见体验。
- 用户在 2026-08-27 新增大陆可用性目标，2026-08-29 收缩为「登录后使用」并修订了手段边界；第三方源码证明 failure taxonomy 与分目标诊断值得采用，也证明它们的直连能力来自"关证书校验 + 省 SNI + 固定 IP"三件套。该目标不等于已经通过大陆真机验证。

## Requirements

### R1. Replica 边界与归属

- Replica v1 必须优先还原原版最终业务版本的可见体验；现代组件不得造成用户可见的导航、布局或交互漂移。
- 项目保持 AGPL-3.0-only，并保留原作者 `git-xiaocao`（小草）的归属和修改说明。
- 新应用使用 `io.github.lopution.pixivfunc`，显示名称保持 `Pixiv Func`；不得复用原作者包名或签名。

### R2. 工程、主题、图标与启动壳

- 保持 Flutter 3.47 stable、API 36、AGP 9、Built-in Kotlin 和现代 Flutter plugin DSL 的可构建基线。
- 迁入并注册原版 `assets/icon.ttf`，Home 前四个图标必须使用 `iconFont` 原始 codepoint，第五个设置图标继续使用 `Icons.settings`。
- 保持 `FuncTokens`、Replica 自有主题、原版颜色与字号，并避免 Material 3 默认表现造成漂移。
- 冷启动严格实现三态：未完成 guide 到 Welcome；已完成 guide 且无账号到 Login；有账号到 Home。
- 首次引导保持 `Welcome → Language → Theme → LoginPage(isFirst: true)`，且仅在 Theme 下一步时持久化 guide 完成状态。

### R3. 账号与凭据

- 提供 `Account`、`AccountStore` 和 `CredentialStore`，支持多账号、当前账号、账号切换、登录、退出和 re-auth required 状态。
- access token、refresh token、cookie 和 OAuth secret 必须进入 Android 安全存储，不得写入 SharedPreferences、日志或测试快照。
- `StartupGate` 必须读取真实账号状态，不再使用固定 `hasAccount: false`。
- 剪贴板账号迁移保留原版可见 UX；信封本身是透明传输格式（版本 + payload 类型 + SHA-256 损坏检测），真实边界是 `TransferCredentialVerifier` 拿凭据到 Pixiv 服务端换权威身份，加上 Android sensitive clipboard 标记与短时间自动清除。不得继续使用原版硬编码 AES 密钥，也不再引入 nonce/TTL/重放存储——信封携带的是明文凭据，过期不提供任何保护，而重放一次导入是幂等的。

### R4. OAuth PKCE 与 WebView 登录

- 使用 Pixiv OAuth PKCE：一次性 verifier、S256 challenge、明确生命周期，并且成功或失败后清除。
- 授权码与活跃 PKCE 会话绑定；`state` 存在时比对，不存在不强求。缺失、为空或重复的 `code` 拒绝。回调路径形状与未知参数不构成拒绝理由——真实回调是 `pixiv://account/login?code=...&via=...`。
- 登录 WebView 不做导航拒绝。能在这个 WebView 里加载出来的本来就是 Pixiv 自己选择的登录端点，包括第三方 IdP 与 `oauth.secure.pixiv.net`；主机白名单只会拒绝真实需求。
- 只有 `AppLifecycleState.detached` 终止 PKCE 会话。读验证码、跳第三方 IdP、全屏输入法都会离开前台，更严的生命周期规则必然堵死登录。
- 不允许通过 JavaScript 读取账号或密码，不允许绕过 TLS 或证书错误。
- 注册、登录、帮助展开和剪贴板登录入口保持原版 Login 页面可见行为。

### R5. Pixiv 网络与 Token 刷新

- 集中维护 `PixivClientIdentity`、请求头、API/OAuth host 和必要客户端常量；业务层不得散落身份常量。
- 默认网络路径使用直接 HTTPS 和严格证书校验；不得全局放宽 TLS 或开启 cleartext。
- **大陆直连的范围是「登录后使用」**：`app-api.pixiv.net`、`oauth.secure.pixiv.net`、`i.pximg.net`、`s.pximg.net` 四个 Pixiv 自有主机，不含任何第三方源站，不需要 WebView 参与。首次注册与首次登录不在范围内，由「在有连通性的环境登录一次 + 剪贴板账号迁移」承担。本项目不提供任何网络服务。
- `NetworkAccessPolicy` 使用 exact-host destination registry 和 shared transport factory；直连失败且属于 eligible transport failure 时，解析出真实候选地址并逐个尝试严格连接。连接候选 IP 时仍以原 URL hostname 完成 `Host`、系统 CA、证书链与主机名校验。
- API/OAuth、`CachedNetworkImage`、download/Ugoira 与 headless Widget 必须逐出口验证；Updater、反向搜图、翻译等非 Pixiv 流量不得进入 compatibility route。
- Token 刷新按账号 single-flight；20 个并发失效请求只产生 1 次 refresh，每个业务请求最多自动重试一次，无效 refresh 进入重新认证状态。
- 分页必须区分 initial loading、refresh、load more、initial error、load-more error，按实体 ID 去重，并校验 `next_url` 的 origin（https + `app-api.pixiv.net` + 无 userinfo + 默认端口）。路径与参数是 Pixiv 自己的游标状态，由各 repository 钉定其期望的端点与身份参数，其余透传——在这里枚举 Pixiv 的参数只会在它下次加参数时挂掉。
- Compatibility 不得关闭证书或主机名校验、改写 `Host`、domain fronting、使用全局 `HttpOverrides`、第三方 API/图片反代或通用代理。**省略 SNI 允许**，前提是仍完成完整证书链验证并自行核对 SAN 命中目标主机名——那只是把主机名检查换个地方做；不做该核对的省 SNI 与"接受一切证书"是同一个缺陷。固定 IP 只能作为解析失败后的兜底，不得成为安全判断依据。Mutation/Token exchange 在请求体可能发送后不得因 route fallback 自动重放。

### R6. 第一条可用纵向链路

- 在扩大功能面之前，完整打通 `Login → Recommended Illust → Illust Detail → Bookmark`。
- Feed 保存作品 ID，作品实体与收藏状态分别由共享 store 管理，使推荐、详情、搜索和排行榜中的同一作品状态一致。
- Recommended 支持真实数据、分页、刷新、错误、空状态、状态保持与滚动位置恢复。
- Illust Detail 支持单图、多页、R18/AI/Ugoira/page count badge、作者、tags、summary、收藏和全屏查看。
- Fullscreen Viewer 使用水平翻页、`n / total` 标题和 `0.9x–6.0x` zoom。
- 未收藏时短按执行 public bookmark；长按打开约 35% 高度的 public/private sheet；请求中显示约 24px 的 `CupertinoActivityIndicator`，仅 API 成功后更新，失败恢复原状态；已收藏时不再弹长按 sheet。

### R7. Android API 36 用户行为

- 实现 Android 16 edge-to-edge、predictive back 和 Flutter `PopScope`；根页面两次 back 间隔不超过约 1 秒才退出，第一次提示“再按一次退出”。
- 支持原版 `pixiv.net`/`www.pixiv.net`、`pixiv://users|illusts|account` 和 `pixivfunc://users|illusts` deep links。
- 接收 Android `SEND image/*` 用于反向搜图，并使用 `${applicationId}.fileProvider` 暴露受控文件。
- WebView 使用平台自带能力，不引入额外的 WebKit 能力探测依赖；下载通过 MediaStore 写入 `Pictures/PixivFunc`。
- Manifest 不得全局启用 cleartext、TLS 绕过或安装 APK 权限；self updater 权限只属于允许该能力的 GitHub flavor。

### R8. Replica 功能面

- 第一条链完成后，完整覆盖 Ranking、New、Search、User/Profile、Follow、Novel、Comments、History、Settings、Downloads、Ugoira、Reverse image、Profile edit、Live、Widgets 和 Updater；实际落地按 `implement.md` 的无循环依赖顺序推进，必要基础能力可先于其消费者完成，但不得改变原版可见行为或缩减范围。
- Search 保持 fake search box、粉色反向搜图入口、两列 trending tags、Illust/Manga、Novel、User tabs、数字 ID 路由和 cancellable debounce。
- Comments 保持头像跳转、emoji、stamp、translate、reply icon、删除本人评论和加载回复，并避免 reply ID 与 parent ID 混用。
- Novel 使用现代详情 API 与稳定分页布局，保持水平阅读、左右 30% 点击翻页和底部阅读百分比。
- Profile 保持 expanded/collapsed action 差异和完全折叠时才显示居中标题。
- Live 保持 16:9 播放、单击控件、双击播放暂停、清晰度、进度、横屏全屏及作者关注区域，不新增聊天 UI。

### R9. 媒体、下载、历史与性能

- 下载使用共享连接池、流式写入和队列，默认并发 3，progress 更新节流，并保持原版 toast/progress/操作顺序。
- Ugoira 使用磁盘 ZIP、有界 frame cache、有界解码窗口和 scheduler；保持封面、约 70px 播放按钮、暂停覆盖层、离屏停止/恢复和保存 GIF 的体验。
- History 使用单例 repository、单一 DB 生命周期、索引化紧凑 schema 和异步事务；Pixiv history 计时使用可见性与生命周期，不得每秒永久轮询。
- 禁止为制造成功而使用完整大文件 `Uint8List`、无界图片缓存、重复完成回调、隐藏 mock、吞错或 TLS bypass。

### R10. 原版默认值与受控扩展

- 默认值保持 max downloads=3、local history=on、Pixiv history=on、preview high quality=on、theme=system。
- 历史图片源 `210.140.92.148` 只能作为解析失败后的兜底地址，不得成为默认路由，也不得成为安全判断依据。
- Updater 区分 GitHub 与 F-Droid flavor；F-Droid 禁用 self updater 且不请求 `REQUEST_INSTALL_PACKAGES`。
- Replica v1 不新增 custom bookmark tags、Live 弹幕、私信、发布作品、pixivision、二维码/公钥配对，也不补做原版未完整实现的 Novel save/share。

### R11. 验证与完成声明

- 每个子任务至少运行与影响范围相称的 `flutter pub get`、`flutter analyze`、`flutter test`；Android 或集成边界变更还需 debug APK 构建和相应的设备/集成验证。
- 必须覆盖 Startup、OAuth PKCE、Token single-flight、Paging、Bookmark、Viewer 和 Android intent/back/deep-link 关键场景。
- 明确区分自动测试、模拟器测试、真机测试和真实 Pixiv 账号/API 验证；没有证据不得声称对应层面完成。
- 最终集成验收前，所有子任务必须完成、相关 specs 更新、工作树边界清晰，并且不存在空操作或占位业务路径。

### R12. 分阶段交付

- 本任务是父任务，管理完整范围、依赖顺序和跨任务验收；不直接承载大规模实现。
- 每个子任务独立规划、审核、实现、验证、提交和归档，一次只推进一个依赖已满足的实现任务。
- 第一条纵向链完成前，不得同时大规模铺开 Ranking、Search、Novel 等后续功能。

### R13. 开源源码审查后的跨任务加固

- Recommended、Ranking、New、Search 和 Profile feeds 的网络、解析、共享 entity merge 与 cursor commit 必须绑定同一 request generation；refresh 取消旧 append，旧代结果不得写回实体或游标。
- 收藏、关注、评论与资料编辑继续以服务端 confirmed 状态为准；吸收 owner/account isolation、dedupe、superseded revision 和 Retry-After 分类，但 Replica v1 不默认持久化离线意图或在重启后自动重放写操作。
- Novel 在最终验收前必须以纯 Dart typed parser 覆盖 `newpage`、`chapter`、ruby、jump URI/page 和 Pixiv/uploaded image block，未知标记保持可观察；长文布局具有 generation cancellation 和单批工作预算。
- 多页下载与 Ugoira 使用显式 task-group、提交时设置快照、owned temporary/pending output、terminal progress flush 和确定的进程重启策略；不得留下幽灵 running task 或未归属 MediaStore pending item。
- Ugoira archive/frame limits 必须同时约束 entry count、压缩/解压字节、重复/路径、frame count、单帧 bytes、dimensions 与 pixel budget，播放和导出仍采用滑动窗口/流式处理。
- Widget 优先消费版本化、account-revision keyed、无秘密 render snapshot；若后台网络刷新需要 headless Flutter，必须复用同一认证/刷新栈并经 API 36 实测。Updater 必须以固定公钥验证 signed manifest，并校验 exact size、SHA-256、package 与 APK signing certificate。
- Reverse Image 与 Live 采用 capability/feasibility gate；外部服务只剩交互式网页或 endpoint 无法证明时保持差异审批或 blocker，不以 HTML replay、fixture、mock 或 TLS 降级制造完成。
- Mainland access 采用 stable failure taxonomy 与脱敏诊断，并提供分出口、分层（系统 DNS / DoH / TCP / TLS / 真实请求）的探测页，使阻断层次可被实测定位。最终证据必须在系统 proxy/VPN 关闭且无外部代理 App 的真机上覆盖 OAuth/API/pximg/download，并按日期、运营商、IP family、Android 版本和实际 route 记录。样本不足时不得宣称大陆普遍可用。

## Acceptance Criteria

- [ ] AC1：Flutter 3.47 / Android API 36 工程基线、原版 icon font、主题和启动引导均通过自动化验证，且无旧版 Gradle、全局 cleartext 或 TLS 绕过。
- [ ] AC2：真实 AccountStore 决定冷启动三态；多账号、切换、退出、re-auth 和安全凭据存储通过测试。
- [ ] AC3：OAuth PKCE 在真实 Android WebView 中完成登录；verifier 一次性、授权码与会话绑定、退到后台再返回会话仍可用、第三方 IdP 跳转不被拦截，均通过测试与真机验证。
- [ ] AC4：Pixiv 网络客户端和按账号 single-flight Token 刷新通过并发、重试上限、恶意 next URL 与失效 refresh 测试。
- [ ] AC5：在真实账号和 Android 设备上完整走通 `Login → Recommended → Detail → Bookmark`，同一作品跨页面收藏状态一致且交互符合 beta56。
- [ ] AC6：Ranking、New、Search、Profile/Follow、Novel、Comments、History、Settings、Downloads、Ugoira、Reverse image、Profile edit、Live、Widgets、Updater 均有对应可见行为验收和回归测试。
- [ ] AC7：API 36 edge-to-edge、predictive back、双击退出、deep links、`SEND image/*`、FileProvider、WebView 和 MediaStore 在受支持设备上验证。
- [ ] AC8：下载、Ugoira、历史与分页在失败、取消、后台/前台、长列表和大媒体场景下无伪成功、重复回调或无界内存路径。
- [ ] AC9：README、LICENSE/NOTICE、原作者归属、flavor 权限和发布说明与实际实现一致，能够构建经过验证的 debug 与 release 产物；发布远程版本不属于本任务自动授权范围。
- [ ] AC10：所有计划子任务已归档，最终全量 `flutter analyze`、`flutter test`、Android build 和集成验收结果有真实记录，Replica v1 不残留业务占位或空操作。
- [ ] AC11：`08-27-open-source-pixiv-app-plan-audit` 的 17 项矩阵全部有落点；R13 的 generation、typed markup、媒体边界、Widget snapshot、signed updater 和 capability gates 均有自动化/设备证据或明确 blocker。
- [ ] AC12：在系统 proxy/VPN 关闭、无外部代理 App 的 API 36 真机上完成真实 OAuth、`Login → Recommended → Detail → Bookmark`、pximg 图片与下载/Ugoira、WebView 路径；每个 transport 出口、route/failure 和大陆网络样本有分层证据。移动/联通/电信样本不齐时，产品只能声明已测范围而不能宣称大陆普遍可用。

## Out of Scope

- Replica v1 之后的主动 UX 重设计或 Material 3 风格迁移。
- custom bookmark tags、Live 弹幕、私信、发布作品、pixivision。
- 二维码/公钥配对式账号迁移。
- 原版未完整实现的 Novel save/share。
- 未经单独授权的 Play Store、F-Droid、GitHub Release 发布或其他远程写入。
- 通用代理/VPN、远程中继、第三方 Pixiv API/图片反代，或承诺所有大陆地区/运营商/时间点 100% 可用。

## Risks and Deferred Items

- Pixiv OAuth、客户端身份、API、Live 与 WebView 行为会变化；相关子任务开始时必须用当前可信实现或运行时证据重新核验，不得盲抄历史常量。
- 原版镜像 commit 是可见行为事实来源，但现代 Android、安全和性能实现以当前平台要求为准；冲突时遵循“体验冻结，内部重写”。
- 当前仓库 `LICENSE` 实际为 GPL v3，而 README/需求要求 AGPL-3.0-only；许可证与归属修正必须在发布验收前完成并单独审查。
- 真实账号、设备、签名材料或外部服务不可用时，对应验收保持明确未完成，不得以 mock 替代业务接受。
- 第三方开源项目中同时存在硬编码 OAuth secret、关闭证书校验、BODY 日志、整文件 bytes、全帧解码和 destructive migration；这些只作为拒绝证据，不能因项目活跃或 star 较多而进入实现。三个参考客户端能在境内直连，靠的是"关证书校验 + 省 SNI + 固定 IP"三件套——**只采纳省 SNI（且必须自行核对证书链与 SAN），固定 IP 降级为兜底，关证书校验继续拒绝**。
- 大陆运营商行为与 Pixiv 主机地址均具有时效性；无法取得真实网络样本时只能报告"实现/自动测试完成，区域接受未验证"。省 SNI 是否仍然有效必须以实测为准，不能以历史结论代替。

## Open Questions

- Q1（已决议 2026-08-27，2026-08-29 修订）：「不使用代理」指用户无需安装或配置外部代理/VPN。范围收缩为**登录后使用**的四个 Pixiv 自有主机；首次注册/登录不在范围内。App 默认 direct-first，eligible failure 后走仅限 Pixiv 域的 DoH 与严格连接。2026-08-29 推翻了原先对"省 SNI / 固定 IP"的一并拒绝：省 SNI 在自行完成链验证与 SAN 核对的前提下允许，固定 IP 降为兜底，关闭证书校验继续拒绝。ECH 与 WebView loopback 两条路径的实现已删除，不再是本项目的方案。
