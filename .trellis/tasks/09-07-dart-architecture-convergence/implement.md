# 执行计划：Dart 架构收敛（child C）

## 执行前状态

- 当前 task：`.trellis/tasks/09-07-dart-architecture-convergence/`，分支 `task/09-07-dart-architecture-convergence`。
- 开工 gate 已满足：触及 `lib/` 的 09-01 child 全部归档；recount（HEAD `1186ff2`）已写入
  `research/architecture-recount.md`；测试基线 **686**、analyze 0 issues。
- 每个 checkbox 一个提交（消息见各条）；先有新路径测试，再删旧路径。
- 工具链：`export PATH=/opt/flutter-3.47.2/bin:$PATH`；`flutter analyze --no-pub`；`flutter test -j 4`。
- 停止条件：Hero 转场 / Tab 动画 / 下拉刷新 / `PagedFeedController` 三相语义任一被破坏 → 回滚该提交。

## C0 规则先行

- [x] C0 写 `test/architecture/layering_test.dart`：import 图断言（core 不 import features/app；
      features 之间只允许 `lib/app/navigation/routes.dart`；repository/controller/entity 只在 `core/`；
      `features/` 无 `_*Tail/_*Error/_*Empty/_*Card/_*Status`）。白名单初值 = recount §B 的 1+44+1 条
      （抄 HEAD 行号）。首次运行以白名单通过。规则文本写入 `.trellis/spec/frontend/directory-structure.md`
      （先修模板，状态列改真实状态；与 E 协作，C 负责规则文本）。
      提交 `test(arch): layering rules with current violations allow-listed`（f05f485）。

## C1 删除零引用

- [x] C1a 删 `lib/core/network/compat_network.dart`、`lib/core/platform/android_platform.dart`（2 个 0 导入 barrel）。
- [x] C1b `tag_search_repository.dart`：视 `test/illust_detail_controller_test.dart:16` 处理（测试改用 `searchFeedProvider` + `IllustSearchQuery`，文件删除）。
- [x] C1c 12 个 once 公共声明删或私有化：实际删除 9 个（`DnsRecordType`、`PageFetcher`、`StableAnchor`、`ProfileTab`、`MeProfileTab`、`localBlockR18Provider`、`localBlockAIProvider`、`showProfileEditPage`、`showMePage`）；`ProfileFieldContract`/`SearchResultTypeWire`/`UserRestrictWire` 成员仍在使用，保留。
- [x] C1d 删 8 个未用 `AppIcons` 常量（`addFollow/filter/me/toggle/pawoo/twitter/web/blocked`）；放宽 `test/icon_font_test.dart` 到实际 glyph 集。
- [x] C1e `analysis_options.yaml` 开 `unreachable_from_main` → 0 问题。
      提交 `refactor: remove unreferenced files and declarations`（两个提交 5d23552、7045cdb）。

## C2 路由与导航目录合并

- [x] C2a `ReplicaPageRoute` 唯一：删 `replicaRoute()`（3 处调用改 `ReplicaPageRoute`）；
      `lib/core/navigation/route_observer.dart` 保留在 core（被 `history_visibility` 使用）；
      `home_shell_metrics.dart` 移到 `lib/app/navigation/`。
- [x] C2b `lib/app/navigation/routes.dart` 门面：以 id/参数为形参（`openIllust(context, id)` 等），
      28 处 `Navigator.of(context).push(ReplicaPageRoute(...))` 走门面；`search_router.dart` 并入
      `openSearchResults`（数字 ID 分派）；`homeShellTabs` 进门面；`HomeShellMetrics` 静态量改
      Riverpod Notifier（Hero flight 读，无 ProviderScope 时回退常量）；`NovelCard` 顺带迁
      `lib/app/widgets/`（3 处调用点）。提交 `refactor(nav): routes.dart facade and shell metrics provider`（813e9f4）。

## C3 组件层

- [x] C3a `IllustCard` 移 `lib/app/widgets/feed/illust_card.dart`；`BookmarkSwitchButton` 一并移 `lib/app/widgets/`；
      Hero 转场几何（`illustHeroTag`/`illustHeroFlightShuttleBuilder`/clip）移 `lib/app/motion/hero_transition.dart`。
      提交 `refactor(ui): IllustCard, BookmarkSwitchButton, feed grid and hero motion under app`（676cf93）。
- [x] C3b `lib/app/widgets/feed/feed_grid.dart`：`IllustFeedGrid` + `illustColumnsFor`（手机恒 2 列）；8 处统一。
      提交同 676cf93。
- [x] C3c `feed_states.dart`（`FeedTail/FeedEmpty/FeedError`）替换 24 个私有 feed 态 widget
      （recommended×2 页、new、ranking、search、search_result、user、novel、comments、history）。
      保留特殊：`_StartupError`、`_ErrorView`、`_InitialErrorView`、`_ErrorOverlay`、`_InitializationFailure`、
      `_PagePlaceholder`、`_StatusBody`、`_ProfileStatusPage`（整页状态）。视觉保持原样（home_bar golden 逐字节一致）。
      提交 3fb0855、8f87d8b。
- [x] C3d `motion_tokens.dart`（页面 300ms/easeInOutCubic、fast 180ms/easeOut、medium 200ms、imageFade 350ms；
      数据类 Duration 不收拢）；`replica_page_route.dart` 迁 `app/motion/`；`_GlobalRectClip` 公开化留 C8b。
      提交 `refactor(ui): MotionTokens single source...`（74bae8c）。

## C3b PixivImage 变体与 decode 策略

- [x] C3b-1 `PixivImageSize.feed/detail/viewer/avatar` + 命名构造器（含 `.hero(tag:)`）；
      feed/avatar 传 `memCacheWidth`（布局宽 × dpr，上限 1.5× 逻辑像素），detail 按屏宽，viewer 不限；
      13 个调用点切换（recount §D 列表）；`PersonAvatar` 复用 avatar 变体。
      提交 `perf(image): size-aware decode policy for PixivImage variants`。
- [x] C3b-2 widget 测试：断言 feed/avatar 变体 `ImageProvider` 带 `memCacheWidth`。

## C3c 重建边界（静态规则）

- [x] C3c-1 共享组件只 `watch` 所需切片（`select`）；`IllustCard` 不 `watch` 整个 feed state；
      代码走查；layering_test 增加 watch 模式检查（如可能）。

## C4 repository/controller 归位

- [x] C4a 5 个 repository 文件迁 `lib/core/<domain>/`（recommended/illust_detail/related_illust/ranking/
      tag_search）；5 个 `*_controller.dart` 迁 core 且与 repository 分文件（recount §B 清单：
      `RecommendedFeedController`、`IllustDetailController`、`IllustDownloadController`、
      `ProfileIllustFeedController`/`ProfileUserFeedController`、`UserDetailController`）。
- [x] C4b `startup_gate.dart` 上移 `lib/app/`；`widget_feed_loader.dart` 的 core→features 边自然消除。
- [ ] C4c 白名单清零（core→features 0、features a≠b 0、app↔onboarding 0，只留 routes.dart 环）。
      提交 `refactor(core): move repositories and controllers under core`。

## C5 单一 owner

- [x] C5a 主机/头：`PixivDestinationRegistry._allows`、预热列表、`PixivFastRouteStore._bootstrap`、
      `network_probe_page`、`illust_detail_repository.dart:18` 自建 host 从 `PixivClientIdentity` 派生；
      新增 `PixivHeaders.image()` 供 4 个调用点（download_manager/widget_feed_loader/ugoira_repository/
      pixiv_image）。提交 `refactor(net): derive hosts from PixivClientIdentity`。
- [x] C5b `sharedPreferencesProvider` + `PreferenceKeys`（键值不变）替换 7 处直接构造；
      测试 19 处 `InMemorySharedPreferencesAsync` 样板收一个 helper。
- [x] C5c `lib/core/entity/json_read.dart` 替换 `_map`×5、`_firstString`×3、`_positiveInt`×5、
      `_optionalString`×4、`_requiredString`×2、`_nonNegativeInt`×2、`_nextUrl`×5；容器统一
      `Map<String, Object?>`。
- [x] C5d `showAppSnackBar` 替换 44 处；`log()` 替换 23 处 `debugPrint`。
- [x] C5e HTTP 客户端 provider 化：`thirdPartyHttpClientProvider`（翻译/SauceNAO）、
      `resolverHttpClientProvider`（DoH/probe）；生产代码禁止内联 `http.Client()`/`HttpClient()`。
      理由写 `backend/directory-structure.md`。
      提交：每项一个提交。

## C6 i18n（gen-l10n）

- [x] C6a `tool/migrate_i18n.py`：从 `ReplicaStrings._values` 生成四语言 ARB（zh 模板）；`{0}` → 占位符；
      ru 补 `detailQuality`；修 `networkDohEndpointsInvalid`。
- [x] C6b `l10n.yaml`；`flutter gen-l10n` 成功；四语言 key 集合一致。
- [x] C6c 调用点替换：27 个包装与 `ReplicaStrings` 删除；`ReplicaLanguage` 保留为设置枚举；
      `MaterialApp.locale` 由设置驱动（现有行为）；删 `i18n_network_keys_test.dart` 手写 key 列表。
      提交 `i18n: migrate to gen-l10n`（可拆 2–3 个提交：脚本+ARB、调用点、删除）。

## C7 状态范式

- [x] C7a `ProfileEditController`/`ReverseImageSearchController` → Riverpod `Notifier`；
      `NovelReaderController` → `Notifier`/`AsyncNotifier`（.autoDispose/.family 按需）；
      页面 `ref.watch`；3 页 4 处平台适配器经 provider 注入。每项一个提交。
- [x] C7b `history_page` → `HistoryFeedController extends PagedFeedController`；保持 account 切换/outbox。
- [x] C7c `rg "ChangeNotifier" lib` 为 0。

## C7b 持久化增长

- [x] C7b-1 `DownloadRecoveryStore` 记录上限（如 200，淘汰已完成项）或增量写入（code review 决定）；
      单测覆盖。提交 `perf(download): cap DownloadRecoveryStore growth`。
- [ ] C7b-2 history 三条查询 `EXPLAIN QUERY PLAN` 单测断言无 `SCAN TABLE`。

## C8 文件拆分（每文件 ≤600 行）

- [ ] C8a `settings_page.dart`（2,088 行/14 页面类）：`features/settings/pages/<name>_page.dart` 每页一文件；
      共享 `SettingsSection/SettingsTile/SettingsControl` 原语；`MePage` 去重（`user_page.dart` 保留定义，
      设置页本地账号卡改名 `AccountCard`）；`settings_test.dart` 只改 import。
- [ ] C8b `illust_detail_page.dart`（1,388 行）：`_GlobalRectClip` → `motion/hero_rect_clip.dart`
      （`HeroRectClip`，`hero_transition_test` 改 `find.byType`）；`_CaptionRichText` → `caption_rich_text.dart`；
      `_InfoBlock` 等 → `widgets/`。
- [ ] C8c `user_page.dart`（959 行）：3 种 tab feed 各自成文件。
- [ ] C8d `network_policy.dart`（1,198 行）：`network_policy.dart`（策略）+ `pixiv_network_factory.dart` +
      `image_cache.dart`（CacheManager 所有权 + 3.4.x 关闭 bug 绕行注释）；
      `restricted_compat_network_test.dart` 回归。
- [ ] C8e 5 个 >150 行方法拆分（`update_service._checkOnce`、`download_manager._run`、`login_page.build`、
      `IllustCard.build`、`profile_edit_controller.submit`）。
      提交：每文件一个提交。

## C9 颜色与私有化

- [ ] C9a 颜色收敛到 `FuncTokens`/主题（37 处 `Colors.*`；replica 视觉决定的保留并注明）。
- [ ] C9b 141 个仅本文件使用的公共声明按需私有化（随 C8 顺带）；`unreachable_from_main` 0 问题保持。
      提交 `refactor: consolidate colors and privatize local declarations`。

## 验证命令（每阶段后）

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
cd /root/Pixiv-func-C2
flutter analyze --no-pub
flutter test -j 4        # 基线 686，不得减少
dart test test/architecture/layering_test.dart
```

## 完成门槛（PRD Acceptance Criteria）

- layering_test 白名单为空且 CI 通过；`rg "features/" lib/core` 为 0。
- 主机字面量只在 `pixiv_client_identity.dart` 一处（注释除外）。
- `rg "SharedPreferencesAsync()" lib` 为 0；`rg "http.Client()|HttpClient()" lib` 生产路径为 0。
- gen-l10n 生成成功、四语言 key 一致、`ReplicaStrings` 与 27 包装已删。
- `rg "ChangeNotifier" lib` 为 0；history_page 无手写 ScrollController 分页。
- 零引用文件/声明为 0；`unreachable_from_main` 开启且 0 问题。
- 四大文件 ≤600 行；`rg "crossAxisCount: 2" lib/features` 为 0；PixivImage 13 调用点全部用变体构造器；
  `motion_tokens.dart` 外无时长/曲线字面量。
- R8 静态约束测试存在并通过（memCacheWidth 断言、startup_gate_test、history 查询计划、RecoveryStore 上限）。
- `flutter analyze`/`flutter test` 全绿、测试数 ≥686；Hero/Tab/下拉刷新/三相语义测试通过。
- 真机回归（登录/浏览/详情/用户页/历史/设置）由用户执行。
