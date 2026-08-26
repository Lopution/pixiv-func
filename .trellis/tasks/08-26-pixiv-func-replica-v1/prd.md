# 完成 Pixiv Func Replica v1 全量复刻

## Goal

以 Flutter 3.47、Android API 36 和现代安全实现，重新实现已停止维护的 Pixiv Func；Replica v1 冻结原版用户可感知体验，达到可真实登录、浏览、交互、下载和在 Android 真机运行的完整复刻状态。

## Background and confirmed facts

- 用户可感知行为以 `svenfuss/pixiv_func_mobile` 的 `c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`（`1.0.0-beta56+62`，2023-06-21）为主要事实来源。
- 可见体验包括页面结构、导航层级、手势、按钮位置、信息密度、颜色、字号、动画、loading/toast、返回与长按行为、收藏/关注交互和页面状态恢复；Replica v1 不主动重新设计这些行为。
- 内部实现允许现代化重写，包括 OAuth、Token 刷新、网络、状态管理、缓存、数据库、下载、Ugoira、WebView、Android 原生桥接、生命周期和安全存储。
- 当前仓库已有 Welcome、Language、Theme、Login、Home 等 UI shell，以及 Riverpod 设置持久化；当前 Login 按钮仍为空操作，Home 业务页仍为占位，`StartupGate` 仍固定按无账号处理。
- `.trellis/tasks/08-26-flutter-android-scaffold` 已建立 Flutter 3.47 / Android API 36 工程基线并通过 `pub get`、`analyze`、`test` 与 debug APK 构建，但尚未完成 commit/archive。
- 当前缺少原版 `assets/icon.ttf`、账号/网络/实体/分页模块、业务集成测试，以及 deep link、`SEND image/*`、FileProvider、MediaStore 等 Android 行为。
- Trellis 使用 `codex.dispatch_mode: inline`；实现和检查由主会话直接完成，不使用 subagent。

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
- 剪贴板账号迁移保留原版可见 UX，但内部必须具有版本、时间戳、有效期、nonce、完整性保护、Android sensitive clipboard 标记和短时间自动清除；不得继续使用原版硬编码 AES 密钥。

### R4. OAuth PKCE 与 WebView 登录

- 使用 Pixiv OAuth PKCE：一次性 verifier、S256 challenge、明确生命周期，并且成功或失败后清除。
- WebView 仅接受 `pixiv://account?code=...` 回调，其他 scheme/host/path 不得完成登录。
- 不允许通过 JavaScript 读取账号或密码，不允许绕过 TLS 或证书错误。
- 注册、登录、帮助展开和剪贴板登录入口保持原版 Login 页面可见行为。

### R5. Pixiv 网络与 Token 刷新

- 集中维护 `PixivClientIdentity`、请求头、API/OAuth host 和必要客户端常量；业务层不得散落身份常量。
- 默认网络路径使用系统 DNS、直接 HTTPS 和严格证书校验；不得全局放宽 TLS 或开启 cleartext。
- Token 刷新按账号 single-flight；20 个并发失效请求只产生 1 次 refresh，每个业务请求最多自动重试一次，无效 refresh 进入重新认证状态。
- 分页必须区分 initial loading、refresh、load more、initial error、load-more error，按实体 ID 去重，并拒绝恶意或未知 host 的 `next_url`。
- 兼容网络模式作为后续独立能力，只允许 Pixiv/pximg 域的 `CONNECT :443` 和端到端 TLS；不得退化成通用代理或固定 IP 安全核心。

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
- WebView 使用现代 AndroidX WebKit 能力；下载通过 MediaStore 写入 `Pictures/PixivFunc`。
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
- 历史图片源 `210.140.92.148` 只能作为 legacy/emergency route，不得成为默认或安全核心。
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

## Acceptance Criteria

- [ ] AC1：Flutter 3.47 / Android API 36 工程基线、原版 icon font、主题和启动引导均通过自动化验证，且无旧版 Gradle、全局 cleartext 或 TLS 绕过。
- [ ] AC2：真实 AccountStore 决定冷启动三态；多账号、切换、退出、re-auth 和安全凭据存储通过测试。
- [ ] AC3：OAuth PKCE 在真实 Android WebView 中完成登录；verifier 一次性、回调白名单和证书失败行为通过测试。
- [ ] AC4：Pixiv 网络客户端和按账号 single-flight Token 刷新通过并发、重试上限、恶意 next URL 与失效 refresh 测试。
- [ ] AC5：在真实账号和 Android 设备上完整走通 `Login → Recommended → Detail → Bookmark`，同一作品跨页面收藏状态一致且交互符合 beta56。
- [ ] AC6：Ranking、New、Search、Profile/Follow、Novel、Comments、History、Settings、Downloads、Ugoira、Reverse image、Profile edit、Live、Widgets、Updater 均有对应可见行为验收和回归测试。
- [ ] AC7：API 36 edge-to-edge、predictive back、双击退出、deep links、`SEND image/*`、FileProvider、WebView 和 MediaStore 在受支持设备上验证。
- [ ] AC8：下载、Ugoira、历史与分页在失败、取消、后台/前台、长列表和大媒体场景下无伪成功、重复回调或无界内存路径。
- [ ] AC9：README、LICENSE/NOTICE、原作者归属、flavor 权限和发布说明与实际实现一致，能够构建经过验证的 debug 与 release 产物；发布远程版本不属于本任务自动授权范围。
- [ ] AC10：所有计划子任务已归档，最终全量 `flutter analyze`、`flutter test`、Android build 和集成验收结果有真实记录，Replica v1 不残留业务占位或空操作。

## Out of Scope

- Replica v1 之后的主动 UX 重设计或 Material 3 风格迁移。
- custom bookmark tags、Live 弹幕、私信、发布作品、pixivision。
- 二维码/公钥配对式账号迁移。
- 原版未完整实现的 Novel save/share。
- 未经单独授权的 Play Store、F-Droid、GitHub Release 发布或其他远程写入。

## Risks and Deferred Items

- Pixiv OAuth、客户端身份、API、Live 与 WebView 行为会变化；相关子任务开始时必须用当前可信实现或运行时证据重新核验，不得盲抄历史常量。
- 原版镜像 commit 是可见行为事实来源，但现代 Android、安全和性能实现以当前平台要求为准；冲突时遵循“体验冻结，内部重写”。
- 当前仓库 `LICENSE` 实际为 GPL v3，而 README/需求要求 AGPL-3.0-only；许可证与归属修正必须在发布验收前完成并单独审查。
- 真实账号、设备、签名材料或外部服务不可用时，对应验收保持明确未完成，不得以 mock 替代业务接受。

## Open Questions

无阻塞性产品决策；后续技术未知项在对应子任务中研究，并且不得改变本 PRD 的 Replica 行为边界。
