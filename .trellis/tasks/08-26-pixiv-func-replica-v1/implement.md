# Pixiv Func Replica v1 执行计划

## 1. Parent-task rule

本目录是父任务，只负责完整范围、依赖、任务地图和最终验收，不直接运行 `task.py start` 承载全部实现。原 Replica 树的 27 个实现叶子已创建；截至 2026-08-27，其中 18 个已归档、`08-26-ugoira-player-export` 处于 `in_progress`、8 个仍在 planning。另有 5 个独立 hardening leaves 由 `08-27-replica-v1-hardening` 协调。实现时一次只启动一个依赖已满足且再次获批的叶子任务。仓库保持 Codex inline 模式，不派发 subagent。

## 2. Actual task hierarchy

父任务当前有 17 个直接子任务。其中 6 个是只协调范围和验收的中间父任务，不能承载产品实现；原树总计 27 个实现叶子，补强树另有 5 个实现叶子。任务归档不替代 Milestone 所要求的设备/API证据。

| 直接子任务 | 类型 | 叶子范围 |
|---|---|---|
| `08-26-flutter-android-scaffold` | 已归档叶子 | Flutter 3.47 / API 36 工程基线 |
| `08-26-restore-icon-font` | 叶子 | 原版 `iconFont` 资产与注册 |
| `08-26-secure-account-store` | 叶子 | 账号模型、安全凭据、多账号和 StartupGate |
| `08-26-oauth-pkce-webview-login` | 叶子 | 一次性 PKCE 与严格 WebView 登录 |
| `08-26-pixiv-network-token-refresh` | 叶子 | 统一网络客户端与 per-account single-flight refresh |
| `08-26-recommended-feed-paging` | 叶子 | 推荐流、共享作品实体与分页 |
| `08-26-illust-detail-viewer` | 叶子 | 详情、多页查看器和真实下载入口 |
| `08-26-bookmark-state-sync` | 叶子 | 收藏交互与跨页面共享状态 |
| `08-26-android-platform-parity` | 叶子 | back、deep link、SEND、FileProvider、WebView/MediaStore 基础 |
| `08-26-discovery-search-reverse-image` | 中间父任务 | `ranking-feed`、`new-content-feeds`、`search-catalog`、`reverse-image-search` |
| `08-26-profile-follow-social` | 中间父任务 | `user-profile-follow`、`profile-edit` |
| `08-26-comments-history-settings` | 中间父任务 | `settings-parity`、`comments-replies`、`history-persistence` |
| `08-26-novel-reader` | 叶子 | 当前 Novel API 与水平阅读 |
| `08-26-downloads-ugoira-media` | 中间父任务 | `download-manager-mediastore`、`ugoira-player-export` |
| `08-26-compat-network-account-migration` | 中间父任务 | `restricted-compat-network`、`secure-clipboard-account-migration` |
| `08-26-live-widgets-updater` | 中间父任务 | `live-player`、`android-home-widgets`、`updater-flavors` |
| `08-26-replica-v1-integration-release` | 叶子 | 全量回归、许可、release 构建与最终证据矩阵 |

## 3. Dependency-safe leaf order

以下是单 agent inline 执行顺序。它保留父 PRD 的功能范围；仅把 Settings、Novel、User route 和下载基础提前到其消费者之前，消除循环依赖和业务空操作。

| 顺序 | 实现叶子 | 完成后解锁 |
|---|---|---|
| 0 | `08-26-flutter-android-scaffold` | 可提交工程基线 |
| 1 | `08-26-restore-icon-font` | 原版导航图标、Settings UI 基础 |
| 2 | `08-26-secure-account-store` | 真实 StartupGate、OAuth、账号隔离 |
| 3 | `08-26-oauth-pkce-webview-login` | 真实账号登录与 callback contract |
| 4 | `08-26-pixiv-network-token-refresh` | 所有认证 API 与分页 repository |
| 5 | `08-26-android-platform-parity` | typed route、MediaStore、WebView 和 Android 输入边界 |
| 6 | `08-26-recommended-feed-paging` | 共享 IllustStore/Paging 和首个真实 feed |
| 7 | `08-26-download-manager-mediastore` | Detail 的单页/全部页下载可真实执行 |
| 8 | `08-26-illust-detail-viewer` | 详情、viewer、下载模式 |
| 9 | `08-26-bookmark-state-sync` | Milestone 1 主链 Gate |
| 10 | `08-26-ranking-feed` | Ranking |
| 11 | `08-26-settings-parity` | History/Comments/下载配置 provider |
| 12 | `08-26-user-profile-follow` | 共享 UserStore/FollowStore 与 typed user route |
| 13 | `08-26-novel-reader` | Novel entity/reader，解锁 New/Search 的 Novel 结果 |
| 14 | `08-26-new-content-feeds` | Following/Everyone/My Pixiv 全内容流 |
| 15 | `08-26-search-catalog` | 三类搜索与反向搜图结果路由 |
| 16 | `08-26-comments-replies` | 评论、回复和用户跳转 |
| 17 | `08-26-history-persistence` | 作品/小说历史与生命周期记录 |
| 18 | `08-26-ugoira-player-export` | 有界 Ugoira 播放与 GIF 导出 |
| 19 | `08-26-restricted-compat-network` | 大陆无外部代理 P0：shared direct-first/strict compatibility transport 与真实网络 Gate |
| 20 | `08-26-reverse-image-search` | 实时核验后的图片检索链路 |
| 21 | `08-26-profile-edit` | 原版资料编辑入口与提交；复用已完成的 access policy |
| 22 | `08-26-secure-clipboard-account-migration` | 有残余风险说明的受控账号迁移 |
| 23 | `08-26-live-player` | 当日核验可用时的 Live 体验；否则明确 blocker |
| 24 | `08-26-android-home-widgets` | 无明文凭据的小组件 |
| 25 | `08-26-updater-flavors` | GitHub/F-Droid 权限隔离 |
| 26 | `08-26-replica-v1-integration-release` | Replica v1 最终完成判定 |

每个中间父任务在其全部叶子归档后只执行范围、共享状态和回归证据汇总，然后归档；不另开产品代码实现。

## 4. Milestone gates

### Milestone 0 — Foundation ready

- 收尾并归档现有 scaffold 子任务。
- 完成 icon font、真实账号状态、OAuth 登录和网络/Token 基础。
- Gate：冷启动三态与真实登录可用；`flutter analyze`、`flutter test`、debug APK 构建通过；敏感凭据未落入普通存储。

### Milestone 1 — First usable chain

- 完成 `recommended-feed-paging`、`download-manager-mediastore`、`illust-detail-viewer` 和 `bookmark-state-sync`，并补齐该链需要的 Android 页面行为。
- Gate：在真实 Android 设备和账号上打通 `Login → Recommended → Detail → Bookmark`，覆盖失败、分页、收藏同步和 viewer 行为。
- 在该 Gate 通过前，不开始任务 9–15 的大规模实现。

### Milestone 2 — Replica breadth

- 当前 Ugoira 工作边界完成后，先完成 `restricted-compat-network`：统一 API/OAuth/image/download/WebView 的 strict route，并建立大陆无外部代理真实网络矩阵；Q1 已允许该 App 内部范围，但仍需叶子自身 start approval。
- 随后按上表依赖顺序完成 Reverse Image、Profile edit、clipboard 和 late-parity 功能；Profile/Live/Widget 复用同一 access policy，不自行写 host/proxy fallback。
- 每个功能先核对 beta56 可见行为，再建立现代内部实现和聚焦测试。
- Gate：PRD R8–R10 的每项功能都有实现证据、自动化回归和必要的设备/API 验收；AC12 按 transport 出口记录，样本不足时不宣称大陆普遍可用。

### Milestone 3 — Integrated Replica v1

- 完成任务 16 的跨功能、生命周期、失败恢复、安全、性能和发布构建验收。
- 修正 LICENSE/NOTICE/归属与 flavor 权限，清理所有业务 placeholder/no-op。
- Gate：父任务所有 AC 完成，所有子任务均提交并归档，未验证边界明确为 blocker 而不是伪成功。

## 5. Child-task workflow

对每个待开始条目执行：

1. 打开已有叶子任务，从父任务要求和 beta56 对应源码再次核对可观察行为、实时事实和前置 Gate。
2. 检查当前仓库代码、测试和相关 specs；将依赖与前置 Gate 写入子任务 PRD。
3. 对复杂子任务补齐 design/implement，明确数据流、错误、迁移、Android 边界与回滚点。
4. 向用户提交最终 planning summary；收到后续明确批准才运行 `task.py start`。
5. inline 模式下运行 `trellis-before-dev`，小步实现；不使用 subagent。
6. 运行聚焦测试与 `trellis-check`，再执行本节质量门禁。
7. 按需更新 `.trellis/spec/`，只提交该子任务文件，记录真实验证结果并归档。

## 6. Validation matrix

每个子任务最少执行：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate <child-task>
git diff --check
```

涉及 Android、WebView、插件、Manifest、下载或 release 的子任务增加：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

并记录适用的模拟器/真机、Android API、intent/deep-link、OAuth/API 账号与网络环境证据。最终集成任务至少执行全量测试、debug build、release build，以及 PRD AC3、AC5、AC7 所需的真机验收。

专项测试不可省略：

- Startup：guide false、guide true/no account、account exists、语言/主题持久化。
- OAuth：PKCE challenge、verifier one-use、仅接受 `pixiv://account`、证书错误失败。
- Token：20 并发失败仅一次 refresh、最多重试一次、invalid refresh 进入 re-auth。
- Paging：ID 去重、load-more error 独立、恶意 next URL 拒绝。
- Bookmark：短按 public、长按 public/private、pending spinner、失败恢复、同 ID 跨页面同步。
- Viewer：多页、`n/total`、`0.9x–6.0x` zoom。
- Android：API 36、predictive back、约 1 秒双击退出、deep links、`SEND image/*`、FileProvider 和 MediaStore。
- Mainland access：system proxy/VPN off、无外部代理 App；OAuth/API/pximg/download/WebView 分出口；direct-first/failure taxonomy、original-host TLS、network switch、移动/联通/电信样本范围。

## 7. Research checkpoints

- 每个可见功能开始前，读取 beta56 对应源码并把关键 file/line 或截图证据写入该子任务 `research/`。
- OAuth client identity、Pixiv API、Live endpoint、Android/WebView API 等时效性事实在对应子任务中从当前可信主源重新核验。
- 大陆运营商路径与 Pixiv 主机地址在实现开始时做当日实测；省 SNI 是否仍然有效必须以探测页结果为准，不能由设计推导。
- 研究结论只能决定内部实现细节；任何改变 Replica 可见体验的发现必须回到父 PRD 并重新走用户审阅。
- 开放叶子开始时同时读取 `../08-27-open-source-pixiv-app-plan-audit/research/source-evidence.md` 中自己的采用/拒绝项；所有外部链接固定到 commit，禁止把第三方 client secret、固定 IP 或 current branch 当事实源。

## 8. Open-source audit hardening gates

这些 Gate 不改变 17 个直接子任务的现有树，也不把已归档任务改写为“当时已完成”。已归档实现暴露的缺口由独立的 top-level `08-27-replica-v1-hardening` 承接；该父任务只协调，不计入原 17 项，也不直接承载产品实现。五个 hardening leaf 均需单独审批后启动，完成证据再由最终集成任务消费。

当前补强任务树：

- `08-27-feed-generation-commit-hardening`：Recommended/Ranking/New/Search/Profile 的 generation、entity 和 cursor commit。
- `08-27-mutation-ownership-hardening`：Bookmark/Follow/Comments/Profile mutation 的 account/revision、dedupe、429 和取消。
- `08-27-novel-markup-hardening`：typed Novel markup、unknown preservation、长文 chunk/budget 和取消。
- `08-27-media-job-recovery-hardening`：Download/Ugoira group、submission snapshot、restart、pending cleanup 和资源上限；等待当前 Ugoira 叶子完成。
- `08-27-android-platform-boundary-hardening`：intent、FileProvider、MediaStore 和生命周期。

这些叶子阻塞 `08-26-replica-v1-integration-release` 的对应 hardening matrix，但不替代原有功能任务或其归档记录。

| Gate | 适用任务 | 必须证据 |
|---|---|---|
| Mainland no-external-proxy access | Compatibility、OAuth/API、Download/Ugoira、Profile、Widgets、Integration | exact-host shared transport、direct-first、failure taxonomy、DoH + original-host TLS、分层探测页、无外部代理真机矩阵；省 SNI 需自验证链与 SAN，固定 IP 仅兜底，禁止关证书校验/改写 Host/第三方反代 |
| Feed generation commit | Recommended、Ranking、New、Search、Profile | refresh-wins、旧代响应不 merge entity/cursor、账号/筛选隔离、同 ID 重排/删除/新增 |
| Mutation ownership | Bookmark、Follow、Comments、Profile edit | account owner、dedupe/superseded revision、429 分类、切换/退出取消；无后台隐式重放 |
| Typed Novel markup | Novel | newpage/chapter/ruby/jump/image/unknown fixtures、长文 chunk/budget、取消后不提交 |
| Media job/recovery | Download、Ugoira | group exactly-once、submission snapshot、owned cleanup、restart/pending policy、archive/frame/pixel limits |
| External capability | Reverse Image、Live | 观测日期、endpoint/provider capability、真实失败；WebView 差异需审批，Live 无证据不引依赖 |
| Android background | Widgets | versioned secret-free snapshot、headless proof 或 blocker、unique work、IPC budget、account invalid cleanup |
| Update trust | Updater | signed manifest fail-closed、size/hash/package/signer、single-flight、F-Droid compile-time absence |
| Release evidence | Integration | 17 项审查矩阵逐项关闭；自动化、模拟器、真机、真实 API 分层记录 |

## 9. Risky files and rollback points

- `pubspec.yaml`、`pubspec.lock`：插件引入按子任务最小化，锁文件随同验证。
- `android/app/src/main/AndroidManifest.xml`、Gradle/flavor 配置：权限与 intent 分阶段添加，检查 merged manifest。
- auth secure storage 与 DB schema：必须版本化，并在写入真实数据前验证迁移/失败恢复。
- navigation、StartupGate、Home tab state：保持现有 shell 与引导状态，不跨子任务批量重写。
- 每个子任务独立提交；不得 reset、force push、重写历史或清理无关未跟踪文件。

## 10. Start gate

- 本父任务 `prd.md`、`design.md`、`implement.md` 已完成后，只进入用户审阅，不启动父任务。
- 本次开源审查规划与 Q1 决议不自动启动产品实现；当前 `08-26-ugoira-player-export` 已是独立在途工作，完成其边界后，下一个 planning 候选改为 P0 `08-26-restricted-compat-network`，仍需自己的 final planning review 与明确 start 批准。
- 如果父任务范围、功能边界或验收条件发生实质变化，必须更新三份产物并重新取得批准。
