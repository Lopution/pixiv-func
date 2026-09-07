# Research: Dart/Flutter 应用架构与可维护性审计（`lib/` 与 `test/`）

- **Query**: 面向 09-02 全项目重构的 Dart 层静态审计：结构图、分层与依赖方向、重复与散落契约、组件可替换性、测试形态、LLM 可维护性信号，并给出重构候选表。
- **Scope**: internal（仅 `lib/`、`test/`、`pubspec.*`、`analysis_options.yaml`、`.trellis/spec/`；忽略 `build/`、`.dart_tool/`、`plugins/rhttp/rhttp/rust/target/`）
- **Date**: 2026-09-07

统计口径说明：

- 所有计数来自 2026-09-07 的工作树（`wc -l`、`rg -c`、`/tmp` 下一次性 Python 脚本）。工作树含大量未提交改动（`git status` 显示 `lib/`、`test/` 多处 `M`，`test/` 下 9 个 `??` 文件），数字反映当前磁盘状态而非某个 commit。
- `flutter analyze --no-pub`（Flutter 3.47.0）结果：`No issues found! (ran in 1.7s)`；`analysis_options.yaml` 仅 `include: package:flutter_lints/flutter.yaml`，未启用任何额外 lint 规则。
- 无生成代码：`lib/` 下 `*.g.dart` / `*.freezed.dart` 为 0，无 `riverpod_annotation` / `@riverpod`。

---

## A. 结构图

### A1. 顶层布局与规模

| 目录 | Dart 文件数 | LOC | 备注 |
|---|---|---|---|
| `lib/`（合计） | 200 | 51,772 | `lib/main.dart` 34 行 |
| `lib/app/` | 14 | 855 | 根级 5 个文件（`app.dart` 120、`pixiv_image.dart` 206、`person_avatar.dart` 71、`pull_to_refresh.dart` 74、`replica_page_route.dart` 27）+ `icons/` 1/20、`navigation/` 1/16、`theme/` 2/98、`widgets/` 5/223 |
| `lib/core/` | 135 | 33,254 | 22 个子目录，见下表 |
| `lib/features/` | 50 | 17,629 | 13 个子目录，见下表 |
| `test/`（合计） | 72 | 21,903 | 71 个测试文件平铺在 `test/` 根 + `test/helpers/illust_fixtures.dart`（74 行）；`test/goldens/home_bar.png` 1 个 golden；`test/failures/` 4 个 png（未跟踪的 golden 失败产物） |

`lib/core/` 子目录：

| 目录 | 文件 | LOC | 目录 | 文件 | LOC |
|---|---|---|---|---|---|
| `network/` | 16 | 4,827（其中 `compat/` 9 文件 4,061） | `history/` | 6 | 1,163 |
| `download/` | 11 | 3,391 | `user/` | 7 | 1,179 |
| `ugoira/` | 10 | 2,295 | `widget/`（Android 桌面小部件，非 Flutter widget） | 6 | 1,083 |
| `novel/` | 4 | 1,995 | `entity/` | 4 | 1,066 |
| `comments/` | 8 | 1,955 | `settings/` | 5 | 1,000 |
| `i18n/` | 1 | 1,924 | `search/` | 5 | 950 |
| `profile/` | 7 | 1,898 | `bookmark/` | 4 | 540 |
| `updater/` | 6 | 1,838 | `new/`（"New" 页 feed，与关键字同名） | 3 | 300 |
| `reverse_image/` | 7 | 1,742 | `mutation/` | 2 | 255 |
| `auth/` | 10 | 1,702 | `navigation/` | 2 | 27 |
| `platform/` | 9 | 1,290 | `paging/` | 2 | 834 |

`lib/features/` 子目录：

| 目录 | 文件 | LOC | 目录 | 文件 | LOC |
|---|---|---|---|---|---|
| `settings/` | 3 | 2,881 | `search/` | 8 | 1,771 |
| `profile/` | 7 | 2,687 | `home/` | 5 | 1,575 |
| `illust/` | 8 | 2,680 | `comments/` | 4 | 1,051 |
| `novel/` | 3 | 1,885 | `login/` | 3 | 861 |
| `history/` | 1 | 608 | `ranking/` | 2 | 461 |
| `new/` | 1 | 449 | `onboarding/` | 4 | 408 |
| `bookmark/` | 1 | 312 | | | |

`features/` 内文件按后缀：`*_page.dart` 24、`*_controller.dart` 6、`*_repository.dart` 5、其余 15 个（`*_layout.dart`、`*_reader.dart`、`*_delegate.dart`、`*_router.dart` 等）。

### A2. `lib/` 最大的 15 个文件

| # | 文件 | LOC | 内容（据类声明与文件头 `///`） |
|---|---|---|---|
| 1 | `lib/features/settings/settings_page.dart` | 2,088 | 14 个公开页面类挤在一个文件：`SettingsPage`、`MePage`、`AccountSettingsPage`、`ThemeSettingsPage`、`LanguageSettingsPage`、`TranslateSettingsPage`、`TranslationCredentialsPage`、`BrowseSettingsPage`、`DownloadSettingsPage`、`DownloadDestinationPage`、`HistorySettingsPage`、`BlockedTagsPage`、`DownloadTasksPage`、`AboutSettingsPage`（L89–L1827），另含 `_persistSettings` 写入包装（L40）与 `_transferErrorText`（L312） |
| 2 | `lib/core/i18n/replica_strings.dart` | 1,924 | 4 语言（zh-CN/en-US/ja-JP/ru-RU）× 434 个字符串键（ru 433）的内联 `Map` 表；`text()`（L1900）以字符串键运行时查表，缺键走 `_values[zhCN]![key]!` 直接抛出 |
| 3 | `lib/core/novel/novel_entity.dart` | 1,559 | `NovelEntity` + 小说标记语言的 token/block 模型（`NovelMarkupToken` 及 10 个子类、`NovelBlock`）+ `NovelContentMapper`（L693）解析器 + 文件底部私有 JSON 强制转换 helper（L1510–L1535） |
| 4 | `lib/features/illust/detail/illust_detail_page.dart` | 1,388 | 详情页 `IllustDetailPage`（ConsumerStatefulWidget）+ 自定义 RenderObject `_GlobalRectClip`/`_RenderGlobalRectClip`（L344–L406，Hero 裁剪）+ `_PageImage`、`_InfoBlock`、`_CaptionRichText`、`_TagChip` 等 12 个私有 widget；`_CaptionRichText` 内含 pixiv 链接路由判断（L1271） |
| 5 | `lib/core/network/compat/network_policy.dart` | 1,198 | `NetworkAccessPolicy`（路由阶梯、路由记忆、per-route 客户端池、诊断，L30）+ `PixivNetworkFactory`（L1143，含 `imageCacheManager` 即 `flutter_cache_manager.CacheManager` 的所有者，L1148–L1190）+ `_RequestCancelSignal`（L1109） |
| 6 | `lib/core/download/download_manager.dart` | 1,164 | `DownloadManager`（L26–L1006，一个类约 980 行）+ `_Job`/`_DownloadGroup` + 5 个异常类（L1061–L1088）+ `_safeError`（L1130） |
| 7 | `lib/features/profile/user_page.dart` | 959 | `UserPage` + 第二个 `MePage`（L48，与 #1 文件中的 `MePage` 同名）+ 三种 tab feed（插画/用户/小说）+ 10 个私有状态/卡片 widget |
| 8 | `lib/features/novel/novel_layout.dart` | 926 | `NovelLayoutEngine`（L303–L886）文本分页/排版引擎及其 cache/budget/anchor 模型；纯逻辑但位于 `features/` |
| 9 | `lib/core/network/compat/secure_resolver.dart` | 868 | `SecureResolver`/`EchConfigResolver` 接口、`DohResolver`、`SystemSecureResolver`；DoH 传输用 `dart:io HttpClient` + `IOClient`（L790–L823） |
| 10 | `lib/core/ugoira/ugoira_zip.dart` | 741 | `SafeZipIndex` 严格 ZIP 索引解析器（自实现 EOCD/central directory），仅从 `package:archive` 引入 `Inflate, getCrc32`（L5） |
| 11 | `lib/features/profile/profile_header_delegate.dart` | 713 | `ReplicaProfileHeaderDelegate`/`ReplicaProfileTabsDelegate`（两个 `SliverPersistentHeaderDelegate`）+ 7 个私有 widget，替代旧 extended_sliver（L107） |
| 12 | `lib/core/ugoira/ugoira_export.dart` | 698 | `UgoiraExportJob`（L375）GIF 导出任务 + `ImagePackageUgoiraGifEncoder`（L65，`package:image` 编码器，`UgoiraGifEncoder` 接口 L53）+ isolate worker `_UgoiraGifWorker`（L142） |
| 13 | `lib/core/paging/paged_feed_controller.dart` | 629 | `PagedFeedState` + 抽象 `PagedFeedController extends AsyncNotifier`（L103）：初始/刷新/加载更多三相、cursor 校验、C9 本地过滤（L119–L163）、generation 提交 |
| 14 | `lib/core/profile/web_profile_repository.dart` | 627 | `PixivWebProfileEditRepository`：走 `www.pixiv.net/ajax/*` 的资料编辑传输适配器 + `_WebUserAgentClient extends http.BaseClient`（L609） |
| 15 | `lib/core/network/compat/network_probe.dart` | 613 | `NetworkProbe` 分层连通性探测（系统 DNS→DoH→TCP→TLS→ECH→最小请求），I/O 全部注入；`HttpClient()` 直接构造于 L474 |

### A3. 名称暗示过渡/兼容意图的目录与文件

| 路径 | 实际含义（据文件内文档） | 是否仍是过渡态 |
|---|---|---|
| `lib/core/network/compat/`（9 文件，4,061 行）与 barrel `lib/core/network/compat_network.dart`（7 行，全部 `export 'compat/…'`） | "compat" 指 **Pixiv 大陆受限网络兼容策略**，不是对旧代码的兼容层。`network_contracts.dart:23` "The only origins the app may contact through its Pixiv compatibility policy"；`network_fast_route_store.dart:10–17` "PixEz uses for its compatibility transport"；`network_providers.dart:23–26` "PixEz's compatibility transport is an internal performance tier"。对应测试 `test/restricted_compat_network_test.dart`（1,133 行，测试目录最大文件之一） | **永久实现**。`networkAccessPolicyProvider` → `pixivNetworkFactoryProvider`（`network_providers.dart:18–44`）被 `pixivHttpClientProvider`（`pixiv_http_client.dart:418–426`）、`oauthServiceProvider`（`account_store.dart:323–329`）、图片缓存（`pixiv_image.dart:144`）、下载（`download_providers.dart` → `policy_download_transport.dart`）全部消费；没有另一条"非 compat"的 Pixiv 出口 |
| `replica_*` / `Replica*`（`lib/app/navigation/replica_route.dart`、`lib/app/replica_page_route.dart`、`lib/app/theme/replica_theme.dart`、`lib/app/widgets/replica_{button,empty_state,scaffold,switch_tile}.dart`、`lib/core/i18n/replica_strings.dart`、`ReplicaProfileHeaderDelegate`、`replicaRouteObserver`；设置键 `replica.settings.v2`） | "replica" = 对原 Pixiv Func beta56 的**复刻**（`pubspec.yaml:3` "Pixiv Func 的现代化复刻项目"）。`lib/` 中 `beta56` 出现 51 次/39 文件，`PixEz` 26 次/8 文件 | 是项目自有 UI 基件与字符串表的**正式命名前缀**，不是待删的过渡层；但同一职责存在两份实现：`replicaRoute()`（函数，`lib/app/navigation/replica_route.dart`，仅 onboarding 3 处调用）与 `ReplicaPageRoute<T>`（类，`lib/app/replica_page_route.dart`，30 处调用）实现完全相同的 300ms 右滑转场 |
| `lib/core/profile/profile_edit_store_bridge.dart`（42 行） | `ProfileEditStoreCommitter`：把资料编辑结果"先持久化 AccountStore 再刷新 UserStore"的提交器（L6–L9）。"bridge" 指两 store 之间的提交桥 | 正式实现，非过渡 |
| `lib/core/navigation/` vs `lib/app/navigation/` | `core/navigation/` 只有 `route_observer.dart`（7 行，全局 `replicaRouteObserver`）与 `home_shell_metrics.dart`（20 行，两个可写 `static double?` 全局量 L15/L19）；`app/navigation/` 只有 `replica_route.dart` | 两个"navigation"目录合计 43 行 3 文件，无文档说明分工 |
| 命名歧义 | `lib/core/widget/` 是 **Android 桌面小部件**（`widget_channel.dart:12` `MethodChannel('pixivfunc/widget')`），与 Flutter "widget" 同名；`lib/core/new/` 是 "New" 页的 feed（与 Dart 关键字同名，需从 `new_feed_repository.dart:43` 文档推断） | — |

未发现名为 `legacy/`、`old/`、`shim/`、`deprecated/` 的目录或文件。`settings_repository.dart:33–36` 有 `legacyJsonKey/legacyGuideKey/legacyLanguageKey/legacyThemeKey` 四个旧键的迁移读取（`'settings'`、`'replica.guide_completed'`、`'replica.language'`、`'replica.theme'`）。

---

## B. 分层与依赖方向

### B1. 页面如何拿到数据（Riverpod 形态）

典型链路（以推荐页为例）：

```
RecommendedFeedView (ConsumerWidget)                         features/home/recommended/recommended_home_page.dart:151
  └ ref.watch(recommendedFeedProvider(key))                 AsyncNotifierProvider.family  recommended_feed_controller.dart:116
      └ RecommendedFeedController extends PagedFeedController (AsyncNotifier<PagedFeedState>)  L18
          └ ref.read(recommendedIllustRepositoryProvider).fetchPage(...)                       L55–L61
              └ PixivHttpClient.getJson(uri)                  core/network/pixiv_http_client.dart
                  └ PixivNetworkFactory.client(appApi)        network_policy.dart:1151  → NetworkAccessPolicy.runLadder → RhttpClientFactory.create
          └ FeedPage.commit → IllustStore.mergeAll(...)       实体进入 Notifier<Map<int, IllustEntity>>，feed 只持有 id 列表
```

Provider 统计（`^final \w+Provider =` 共 **89** 个，全部手写，无 codegen；`flutter_riverpod 3.4.2`）：

| 种类 | 个数 |
|---|---|
| `Provider<T>` | 63 |
| `AsyncNotifierProvider.family` | 12（10 个是 `PagedFeedController` 子类 + `illustDetailControllerProvider`、`userDetailControllerProvider`） |
| `NotifierProvider` | 6 |
| `AsyncNotifierProvider` | 3（`settingsProvider`、`accountStoreProvider`、…） |
| `FutureProvider` | 2 |
| `FutureProvider.autoDispose(.family)` | 2（`novel_page.dart` 内的 `novelDetailProvider`、`novelSeriesProvider`） |
| `NotifierProvider.autoDispose` | 1（`searchAutocompleteProvider`） |

Provider 分布最集中的文件：`lib/core/settings/settings_controller.dart` 19 个（L143–L289，`settingsProvider` + 18 个派生 `Provider<T>` 选择器），其余文件 1–4 个。

Notifier 类：`Notifier` 7 个（`BlockedTags`、`BookmarkStore`、`CommentStore`、`FollowStore`、`NovelStore`、`UserStore`、`SearchAutocompleteController`）、`AsyncNotifier` 5 个（含抽象 `PagedFeedController` 及 10 个子类：`RankingFeedController`、`NewFeedController`、`ProfileIllustFeedController`、`ProfileUserFeedController`、`SearchFeedController`、`RecommendedIllustController`、`RecommendedFeedController`、`RelatedIllustController`、`CommentFeedController`、`UserNovelFeedController`）、**`ChangeNotifier` 3 个不走 Riverpod**：`ProfileEditController`（`core/profile/profile_edit_controller.dart:65`，在 `profile_edit_page.dart:94` 由 State 直接 `new`）、`ReverseImageSearchController`（`core/reverse_image/reverse_image_controller.dart:62`，`reverse_image_search_page.dart:45`）、`NovelReaderController`（`features/novel/novel_reader.dart:135`，`novel_reader.dart:233`）。

`ref.watch` 168 处、`ref.read` 177 处、`ref.listen` 6 处、`ref.onDispose` 16 处。

一致性观察：

- 分页 feed 一律 `AsyncNotifierProvider.family<PagedFeedController子类, PagedFeedState, Key>`，模式统一。
- Repository/Controller 的**摆放位置不统一**：18 个 `*_repository.dart` 中 13 个在 `core/`、5 个在 `features/`（`features/home/recommended/recommended_repository.dart`、`features/illust/detail/illust_detail_repository.dart`、`features/illust/detail/related_illust_repository.dart`、`features/ranking/ranking_repository.dart`、`features/search/tag_search_repository.dart`）；16 个 `*_controller.dart` 中 10 个在 `core/`、6 个在 `features/`。`RankingFeedController` 与 `RecommendedIllustController` 定义在 `*_repository.dart` 文件内（`ranking_repository.dart:112`、`recommended_repository.dart:87`）。
- 三个 `ChangeNotifier` 控制器 + `StatefulWidget` 手动持有，与其余 Riverpod 状态形成第二套范式（见 B4）。

### B2. 层泄漏：`lib/features/` 直接触碰基础设施

| 文件 | 行 | 泄漏项 |
|---|---|---|
| `lib/features/login/login_intercept_controller.dart` | L3 `package:flutter/services.dart`；L4 `package:http/http.dart`；L41 `MethodChannel(_channelName)` | 唯一在 `features/` 内直接 `new MethodChannel` 的文件；同时把主机→purpose 映射重新写了一遍（L135–L139） |
| `lib/features/settings/network_probe_page.dart` | L3 `flutter/services.dart`；L5 `package:http/http.dart`；L186 `.get(Uri.parse('https://${target.host}…'))` | 页面直接发 HTTP 探测请求；L56–L59 重复声明 4 个 Pixiv 主机 |
| `lib/features/profile/profile_edit_page.dart` | L2 `dart:io`；L174 `MethodChannelReverseImageInputPlatform()` | 页面直接构造平台通道适配器 |
| `lib/features/search/reverse_image_search_page.dart` | L2 `dart:io`；L5 `webview_flutter`；L46 `MethodChannelReverseImageInputPlatform()`；L52 `MethodChannelReverseImageExternalLauncher()` | 同上 + WebView 直接持有 |
| `lib/features/home/home_page.dart` | L4 `flutter/services.dart`；L70 `MethodChannelAndroidIntentSource()` | 页面构造平台 intent 源 |
| `lib/features/login/login_webview_page.dart` | L3 `webview_flutter` | WebView 直接持有 |
| `lib/main.dart` | L3 `package:rhttp/rhttp.dart` | 入口调用 `Rhttp.init()` |

`features/` 中 **没有** 直接 import `sqflite*`、`shared_preferences`、`flutter_secure_storage`、`path_provider`、`rhttp` 的文件。`MethodChannel(` 构造共 13 处，12 处在 `core/`（`widget_channel.dart:12,54`、`widget_background.dart:17`、`web_profile_session.dart:11`、`update_platform.dart:27`、`saf_tree.dart:40`、`android_intent_channel.dart:30,60`、`media_store_channel.dart:30`、`account_transfer_clipboard.dart:63`、`reverse_image_platform.dart:47`、`reverse_image_external.dart:10`），1 处在 `features/`。

### B3. import 图

import 数最多的文件（import+export 行）：`illust_detail_page.dart` 33、`settings_page.dart` 29、`user_page.dart` 25、`home_page.dart` 21、`recommended_home_page.dart` 21、`core/widget/widget_feed_loader.dart` 21、`ugoira_viewer.dart` 20、`search_result_page.dart` 20、`novel_page.dart` 19、`login_page.dart` 18。

层间边数（文件级，相对 import 解析）：`features→core` 269、`features→app` 74、`features→features` 92、`core→core` 392、`app→core` 10、**`core→features` 1**、**`app→features` 1**。

反向依赖：

- `lib/core/widget/widget_feed_loader.dart:21` → `lib/features/home/recommended/recommended_repository.dart`（core 引用 features 内的 repository）。
- `lib/app/app.dart:14` → `lib/features/onboarding/startup_gate.dart`，而 `features/onboarding/*` 又引用 `lib/app/widgets/*`、`lib/app/navigation/replica_route.dart`，构成 `app ↔ onboarding` 互相依赖。

模块级（`lib/core/<x>`、`lib/features/<x>`、`lib/app`）互相依赖对共 17 组，其中 `core/` 内 11 组：`auth↔history`、`auth↔network`、`auth↔platform`、`auth↔user`、`download↔network`、`download↔platform`、`download↔settings`、`download↔ugoira`、`entity↔settings`、`entity↔user`、`network↔download`。Tarjan SCC：`core/` 的 13 个模块（auth、bookmark、download、entity、history、mutation、network、novel、paging、platform、settings、ugoira、user）落在**同一个强连通分量**；`app` + `core/widget` + 12 个 `features/*` 落在另一个。

文件级 import 环（SCC>1）：

1. 3 文件环：`core/auth/account_store.dart` → `core/history/history_repository.dart` → `core/auth/account_store.dart`；`account_store.dart` → `core/network/compat/network_providers.dart`，`core/network/pixiv_http_client.dart` → `account_store.dart`/`oauth_service.dart`/`credential_store.dart`。
2. 12 文件环（全部页面）：`comments/comment_item.dart`、`comments/comments_page.dart`、`home/recommended/recommended_illust_page.dart`、`illust/detail/illust_detail_page.dart`、`illust/detail/related_illusts_section.dart`、`novel/novel_page.dart`、`profile/user_page.dart`、`search/reverse_image_search_page.dart`、`search/search_page.dart`、`search/search_result_page.dart`、`search/search_router.dart`、`search/tag_search_page.dart`。成环主因：`IllustCard` 定义在 `features/home/recommended/recommended_illust_page.dart:236`，被 ranking/new/profile/search/history/related 等 6 个其它 feature 模块 import；各页面又互相 push 对方（详情页→用户页→小说页→用户页…）。

### B4. 状态范式混用

| 基类 | `lib/` 计数 |
|---|---|
| `extends StatelessWidget` | 95 |
| `extends ConsumerWidget` | 41 |
| `extends ConsumerStatefulWidget` / `ConsumerState<` | 23 / 23 |
| `extends StatefulWidget` / `State<` | 14 / 14 |
| `HookWidget` 等 | 0 |

`setState(` 匹配 149 处/27 文件；其中 25 处是两个 `ChangeNotifier` 控制器的私有 `_setState(`（`reverse_image_controller.dart` 11、`profile_edit_controller.dart` 14），真正的 Flutter `setState` 为 **124 处/25 文件**。热点：`settings_page.dart` 25、`ugoira_viewer.dart` 12、`comment_input.dart` 9、`search_filter_sheet.dart` 8、`history_page.dart` 7。12 个 `features/` 文件同时含 `StatefulWidget`；`history_page.dart`、`new_page.dart`、`profile_edit_page.dart`、`settings_page.dart` 四个文件在同一文件内混用 `StatefulWidget` 与 `ConsumerStatefulWidget`。

其它全局可写状态：`HomeShellMetrics.bottomNavTop/bottomNavHeight`（`core/navigation/home_shell_metrics.dart:15,19`，静态可写）、`RhttpGate.ready`（`core/network/rhttp_gate.dart:10`，静态可写 `Future?`）。`lib/app/pixiv_image.dart:137–143` 用 `try { ProviderScope.containerOf(context) } catch` 探测是否存在 ProviderScope 并降级（生产 widget 内含为测试便利而写的防御分支）。

---

## C. 重复与散落契约

### C1. 硬编码 Pixiv 主机 / URL / 头

| 字面量 | 出现次数 / 文件数 | 文件 |
|---|---|---|
| `i.pximg.net` | 13 / 11 | `network_contracts.dart:61`、`network_policy.dart:179`、`network_fast_route_store.dart:33`、`pixiv_client_identity.dart:52`、`policy_download_transport.dart:68`(注释)、`login_intercept_controller.dart:139`、`network_probe_page.dart:58`、`app_settings.dart:11,189`（作为 `ImageSourceMode.normal` 的枚举值）、`ugoira_export.dart:685`（合成一个假的 `https://i.pximg.net/img-ugoira-export/<id>.gif` 作为导出资产 URL）、`widget_feed_loader.dart:75,271`(注释)、`pixiv_image.dart:11`(注释) |
| `s.pximg.net` | 8 / 8 | `network_contracts.dart:61`、`network_policy.dart:180`、`network_fast_route_store.dart:34`、`pixiv_client_identity.dart:52`、`login_intercept_controller.dart:139`、`network_probe_page.dart:59`、`network_probe.dart:312`(注释)、`policy_download_transport.dart:68`(注释) |
| `app-api.pixiv.net` | 13 / 8 | `pixiv_client_identity.dart:15,29,33,45`、`network_contracts.dart:55,58`、`network_policy.dart:176`、`network_fast_route_store.dart:27`、`next_page_parser.dart:109`、`login_intercept_controller.dart:135`、`network_probe_page.dart:56,134`、**`features/illust/detail/illust_detail_repository.dart:18`**（自建 `Uri(scheme:'https', host:'app-api.pixiv.net')`，未用 `PixivClientIdentity.appApiBase`；其余 repository 用 `appApiBase.replace/resolve` 共 19 处） |
| `oauth.secure.pixiv.net` | 6 / 6 | `pixiv_client_identity.dart:17`、`network_contracts.dart:56`、`network_policy.dart:177`、`network_fast_route_store.dart:28`、`login_intercept_controller.dart:136`、`network_probe_page.dart:57` |
| `www.pixiv.net` | 14 / 8 | `network_contracts.dart:59`、`network_policy.dart:178`、`network_fast_route_store.dart:32`、`login_intercept_controller.dart:138`、`web_profile_repository.dart:40,41,43,423`、`intent_router.dart:133`、`illust_detail_page.dart:1271`、`user_page.dart:937` |
| `accounts.pixiv.net` | 3 / 3 | `network_contracts.dart:58`、`login_intercept_controller.dart`、`login_webview_page.dart:64` |
| `'Referer'` | 4 / 4 | `download_manager.dart:563`、`widget_feed_loader.dart:275`、`ugoira_repository.dart:138`、`pixiv_image.dart:67` —— 值均取 `PixivClientIdentity.downloadReferer`，但 header map 的拼装重复了 4 次（`pixiv_headers.dart` 只有 `api()`/`oauth()`，没有 image/download 头的构造函数） |
| `'User-Agent'` | 6 / 4 | `pixiv_headers.dart:11,23`、`download_manager.dart:562`、`ugoira_repository.dart:137`（均取 `PixivClientIdentity.userAgent`）；`comment_translation.dart:209,428` 使用独立字面量 `'pixiv-func/1.0'`（非 Pixiv 主机） |
| `'Accept-Language'` | 2 / 1 | 仅 `pixiv_headers.dart:15,27`（默认 `'zh-CN'` 写了两次） |

结论：`lib/core/network/pixiv_client_identity.dart`（56 行，`abstract final class PixivClientIdentity`）是**名义上的单一所有者**（文档 L5 "Feature code must never keep copies of these values"），提供 `appApiBase`、`oauthHost`、`apiHosts`、`downloadHosts`、`downloadReferer`、`userAgent`。但主机 allowlist 在以下 5 处以字面量**再声明**：`PixivDestinationRegistry._allows`（`network_contracts.dart:53–63`）、`NetworkAccessPolicy` 预热列表（`network_policy.dart:176–180`）、`PixivFastRouteStore._bootstrap`（`network_fast_route_store.dart:27–35`）、`network_probe_page.dart:56–59`、`login_intercept_controller.dart:135–139`（主机→purpose 映射与 `_allows` 互为镜像）。

### C2. 重复 helper 模式

| 模式 | 计数 | 代表位置 |
|---|---|---|
| 手写 JSON 强制转换 helper（`_positiveInt`、`_optionalString`、`_requiredString`、`_nonNegativeInt`…） | `_positiveInt` 5 份、`_optionalString` 4 份、`_requiredString` 2 份、`_nonNegativeInt` 2 份 | `core/entity/comment_entity.dart:176,182`；`core/user/user_entity.dart:254–272`；`core/novel/novel_entity.dart:1510–1535`；`core/novel/novel_repository.dart:319–330`；`core/updater/update_manifest.dart:256,263`；`core/entity/illust_entity.dart:424` |
| `_nextUrl(Object? value)` 分页游标提取 | 5 份 | `new_feed_repository.dart:191`、`user_repository.dart:355`、`comment_repository.dart:216`、`search_repository.dart:351`、`novel_repository.dart:330` |
| `final dynamic decoded;` + `jsonDecode` 手工解包 | `jsonDecode(` 19 处/15 文件；`final dynamic decoded` 5 处 | `bookmark_repository.dart:59`、`comment_repository.dart:230`、`comment_translation.dart:125,249,484` |
| `fromJson` 工厂 | 9 个手写 `factory X.fromJson`；`as Map<String, dynamic>` 6 处/3 文件；`Map<String, dynamic>` 105 行 | `illust_entity.dart`(3)、`oauth_service.dart`(2)、`credential_store.dart`(1) |
| 异常类型 | 56 个：`api_error.dart` 中 `sealed class ApiError` + 7 子类；其余 **49 个各自 `implements Exception`**，不共享基类（`download_manager.dart` 5 个、`pixiv_download_transport.dart` 3、`network_contracts.dart` 4、`ugoira_*` 4、`updater/*` 3、`widget/*` 3 …） | `catch (` 209 处；`on Api*` 47 处；`is Api*` 15 处 |
| 错误→文案映射 | 每页各写一份 | `comment_item.dart:160 _translationFailureText`、`settings_page.dart:312 _transferErrorText`、`login_page.dart:130 _loginTransferErrorText`、`related_illusts_section.dart:149 _errorText`、`download_manager.dart:1130 _safeError` |
| SnackBar | `ScaffoldMessenger` 44 处/17 文件，`SnackBar(` 79 处；无共享 toast/snack helper（仅 `settings_page.dart:40 _persistSettings`、`network_settings_page.dart:18 _persistNetwork` 两个"写入+失败提示"包装） | `settings_page.dart` 9、`network_settings_page.dart` 6、`login_page.dart` 4、`illust_detail_page.dart` 4、`comments_page.dart` 4 |
| i18n 取字包装 | **17 个**私有包装函数，全部等价于 `ReplicaStrings.fromTag(Localizations.localeOf(context).toLanguageTag(), key)`；该表达式在 26 文件出现 38 次 | `settings_page.dart:33 _settingsText`、`history_page.dart:25 _historyText`、`network_settings_page.dart:11 _networkText`、`network_probe_page.dart:14 _probeText`、`new_page.dart:446 _newText`、`novel_page.dart:410 _novelText`、`recommended_home_page.dart:24 _recommendedText`、`profile_edit_page.dart:21 _profileEditText`、`user_page.dart:113 _profileText`、`profile_header_delegate.dart:269,488,633`（同一文件三份）、`follow_switch_button.dart:28`、`login_page.dart:125`、`login_webview_page.dart:51`、`theme_page.dart:43` |
| feed 尾部/错误/空态 widget | **32 个**私有 `_*Tail/_*Error/_*Empty/_*Status/_*Placeholder` 类分布在 14 个 feature 文件 | `new_page.dart` 5 个（`_NewStatus/_NewEmpty/_NewError/_NewRefreshError/_NewFeedTail`）、`search_result_page.dart` 4、`user_page.dart` 4、`recommended_home_page.dart` 4、`ranking_page.dart` 2、`comments_page.dart` 2、`novel_page.dart` 2 … |
| 瀑布流网格 | `SliverMasonryGrid.count(` 8 处，7 处 itemBuilder 直接构造 `IllustCard`，1 处 `_HistoryCard` | `history_page.dart:253`、`ranking_page.dart:162`、`user_page.dart:461`、`new_page.dart:273`、`recommended_illust_page.dart:107`、`recommended_home_page.dart:264`、`search_result_page.dart:143`、`related_illusts_section.dart:127` |
| 日期格式化 | 无 `intl`/`DateFormat`；`padLeft(2, '0')` 手拼 8 处/5 文件 | `naming_rule.dart:150–151`、`search_models.dart:264–265`、`search_filter_sheet.dart:49–50`、`history_page.dart:605`（`two()`） |
| 图片 URL 构造 | 无 `img-master`/`c/600x1200` 等路径拼接（0 处）；由实体的 `image_urls` 直接取用；`PixivImage` 25 处/13 文件是唯一图片 widget，`Image.network` 0 处 | `lib/app/pixiv_image.dart` |
| 释放/取消样板 | `void dispose()` 33 + `Future<void> dispose()` 19；`_disposed` 标志 92 处/14 文件；`mounted` 135；`unawaited(` 62；3 个 cancel 信号实现（`CancelToken` `pixiv_http_client.dart:23`、`DownloadCancelToken` `download_transport.dart:6`、`_RequestCancelSignal` `network_policy.dart:1109`）均 `implements NetworkCancelSignal` | — |
| 路由 | 同一 300ms 右滑转场两份实现：`replicaRoute()`（3 处）与 `ReplicaPageRoute`（30 处）；`Navigator.push` 28 处；`MaterialPageRoute` 0；`go_router 17.5.0` 在 `pubspec.yaml:21` 声明但 **0 处 import** | `lib/app/navigation/replica_route.dart`、`lib/app/replica_page_route.dart` |
| 同名类 | `class MePage` 定义两次：`settings_page.dart:349`（只读本地账号卡）与 `user_page.dart:48`（当前账号远程资料页）；`settings_page.dart:299` 需用前缀 `profile.MePage` 区分 | — |

### C3. 常量 / 设置

- 设置模型：`AppSettings`（`core/settings/app_settings.dart`，524 行，26 个 `final` 字段，`currentSchemaVersion = 3` L184，手写 `fromJson` L253 / `toJson` L327 / `copyWith` L438），整体序列化为**一个 JSON 字串**存于键 `replica.settings.v2`（`settings_repository.dart:32`）；4 个 legacy 键 L33–L36。
- 读取入口：`settingsProvider` + 18 个派生 `Provider<T>`（`settings_controller.dart:143–289`）；`features/` 中 `settingsProvider` 直接 `watch/read` 51 处/5 文件。
- **直接构造 `SharedPreferencesAsync()` 的位置 6 处（全在 `core/`）**：`settings/blocked_tags.dart:18,36`（键 `'blocked_tags'`，每次读写各 new 一次，不可注入）、`settings/settings_repository.dart:30`、`updater/update_download.dart:163`、`network/compat/network_fast_route_store.dart:40`（键 `'pixiv.network.fast_routes.v1'`）、`download/download_recovery.dart:372`（键 `'pixivfunc.download.recovery.v1'`）、`auth/account_repository.dart:38`。后 5 处构造器可注入。
- 键命名空间不统一：`replica.*`、`pixiv.*`、`pixivfunc.*`、裸 `blocked_tags`、裸 `settings`；`translation_credentials.dart:90–94` 另有 `_keyNamespace` 前缀的 secure-storage 键。
- 没有项目自有的 KV 抽象；抽象只存在于各业务仓储层：`SettingsRepository`（`settings_repository.dart:18`）、`AccountMetadataRepository`（`account_repository.dart:28`）、`DownloadRecoveryStore`（`download_recovery.dart:337`）、`UpdateDownloadStateStore`（`update_download.dart:134`）。

---

## D. 组件可替换性

版本来自 `pubspec.lock`。"import 文件数"按 `lib/` 内 import 行统计。

| 组件 | import 文件数（lib） | 项目自有抽象 | 替换难度与理由 |
|---|---|---|---|
| `package:http` 1.6.0 | 13 文件 14 行：`pixiv_http_client.dart`、`oauth_service.dart`、`pixiv_download_transport.dart`、`web_profile_repository.dart`、`comment_translation.dart`、`sauce_nao_provider.dart`、`widget_feed_loader.dart`、`network_contracts.dart`、`network_policy.dart`、`rhttp_client_factory.dart`、`secure_resolver.dart`(+`io_client.dart`)、`features/login/login_intercept_controller.dart`、`features/settings/network_probe_page.dart` | `http.Client` 接口本身就是接缝：`NetworkClientFactory` typedef（`network_policy.dart:20`）+ `RhttpClientFactory.create`（`rhttp_client_factory.dart:63`）；`PixivHttpClient`/`OAuthService`/`HttpDownloadTransport` 构造器接受注入的 `http.Client` | **中**。`http.Client` 类型散布 13 文件，但 Pixiv 流量的真实实现只在 `RhttpClientFactory` 一处决定。注意实际有 **3 套 HTTP 栈并存**：(a) rhttp（全部 Pixiv 出口）；(b) `http.Client()` 默认 IOClient 用于非 Pixiv 主机（`comment_translation.dart:535`、`sauce_nao_provider.dart:33`，生产路径）及 3 个构造器默认值（`oauth_service.dart:98`、`pixiv_http_client.dart:60`、`pixiv_download_transport.dart:55`；生产 provider 均显式注入策略客户端，见 `pixiv_http_client.dart:426`、`account_store.dart:325–327`）；(c) `dart:io HttpClient()` 直接使用：`secure_resolver.dart:790`（DoH）、`network_probe.dart:474`、`update_service.dart:433`（updater manifest） |
| `package:rhttp` 0.18.0（vendored `plugins/rhttp/rhttp`） | 3 文件 4 行 + `main.dart:3`：`rhttp_client_factory.dart`、`network_contracts.dart`（L366–L477 `_classifyRhttp` 异常分类）、`widget_background.dart:6,35`（后台 isolate 再 `Rhttp.init()`） | `RhttpClientFactory`（文档 L10 "This is the transport replacement point"）+ `TransportFailureClassifier` | **低**：3 个文件 + 2 个 init 点（`main.dart:24`、`widget_background.dart:35`）+ `RhttpGate` |
| `cached_network_image` 3.4.1 + `flutter_cache_manager` 3.4.2 | CNI 1 文件（`lib/app/pixiv_image.dart`）；cache_manager 2 文件（`pixiv_image.dart`、`network_policy.dart:5`） | `PixivImage`（`pixiv_image.dart`，25 处/13 文件唯一图片入口，`Image.network` 0 处）；`CacheManager` 实例由 `PixivNetworkFactory.imageCacheManager` 持有（`network_policy.dart:1148–1190`，含对 3.4.x 关闭 bug 的绕行注释 L1190） | **低-中**：2 个文件；但 `PixivImage.provider()` 公开返回类型是 `CachedNetworkImageProvider`（L70，外部无调用者），且缓存实例所有权嵌在 1,198 行的网络策略文件里 |
| `sqflite` 2.4.3 **和** `sqflite_common_ffi` 2.4.2+1（附带 `sqlite3` 3.5.2 Dart FFI 绑定；`sqlite3_flutter_libs` 不在 lock 中） | `history_database.dart`（两者，L4–L5）、`history_repository.dart`（`sqflite` 仅为 `Database/Transaction` 类型 L4） | `HistoryDatabase`（`history_database.dart:12`，唯一 DB `history.db`，2 表，schema v2）；上层 `HistoryRepository` 通过 `historyDatabaseProvider`（`history_repository.dart:382`）取连接 | **两套栈都被编译进产品**：`_platformDatabaseFactory()`（L27–L31）在 Android/iOS 返回 `sqflite.databaseFactory`，否则 `sqfliteFfiInit(); return databaseFactoryFfi`。两者都列在 `dependencies`（`pubspec.yaml:32–33`），非 `dev_dependencies`。运行时每平台只用一套；FFI 分支的生产消费者只有桌面平台，测试用 `databaseFactoryFfi`（`test/history_persistence_test.dart:25,255`）。**全仓库 DB 访问点只有 `history_repository.dart` 一处**（`query/insert/rawQuery/transaction` 集中在 L53–L330） |
| `shared_preferences` 2.5.5 | 6 文件（见 C3） | 无统一 KV 抽象；5/6 可注入 `SharedPreferencesAsync?` | **中**：6 个构造点 + 键命名空间 3 种；测试端 19 个文件各自 `SharedPreferencesAsyncPlatform.instance = InMemorySharedPreferencesAsync.empty()` |
| `flutter_secure_storage` 10.3.1 | 2 文件：`auth/credential_store.dart:3`、`comments/translation_credentials.dart:2` | `abstract class CredentialStore`（L24）→ `SecureCredentialStore`（L32）；`abstract class TranslationCredentialStore`（L74）→ `SecureTranslationCredentialStore`（L86） | **低**：两个具体类各接受 `FlutterSecureStorage?` 注入 |
| `webview_flutter` 4.14.1 | 2 文件（均在 `features/`）：`login/login_webview_page.dart`、`search/reverse_image_search_page.dart:367–473` | 无 widget 层抽象；登录拦截**决策**已抽到 `login_intercept_controller.dart`（L128 注释 "webview_flutter adapter around the shared decision"）；测试用 `WebViewPlatform.instance = _FakeWebViewPlatform()`（`login_navigation_test.dart:185`） | **中**：`WebViewController/NavigationDelegate/WebViewWidget` 直接出现在两页 State 中 |
| `easy_refresh` 3.5.1 | 2 文件：`lib/app/pull_to_refresh.dart`（包装 `PullToRefresh`，14 处/9 文件使用）、`features/profile/user_page.dart:4` | `PullToRefresh` 包装；但 `user_page.dart:541,718` 直接使用 `HeaderLocator`，绕过包装 | **低-中**：1 个包装 + 1 处泄漏 |
| `flutter_staggered_grid_view` 0.7.0 | 8 文件（`SliverMasonryGrid.count` 直接调用，见 C2） | 无 | **中**：8 处调用点形态一致，参数重复 |
| `visibility_detector` 0.4.0+2 | 1 文件：`features/illust/detail/ugoira_viewer.dart:6,115` | 无 | **低** |
| `archive` 3.6.1 | 1 文件：`ugoira_zip.dart:5`，仅 `show Inflate, getCrc32` | `SafeZipIndex` 自实现 ZIP 目录解析，只借用 inflate + crc32 | **低**：两个函数 |
| `image` 4.3.0 | 1 文件：`ugoira_export.dart:6` | `UgoiraGifEncoder` 接口（L53）+ `ImagePackageUgoiraGifEncoder`（L65） | **低**：接口已在，测试用 `_FakeGifEncoder` |
| 已声明未使用 | `go_router` 17.5.0（0 import）、`cupertino_icons` 1.0.9（0 处 `CupertinoIcons`） | — | 删除只需改 `pubspec.yaml`（需按 PRD R3 做消费者搜索后确认） |
| 其它 | `http_parser` 1 文件（`sauce_nao_provider.dart:6`）；`crypto` 7 文件；`meta` 11 文件；`path_provider` 3 文件（`widget_snapshot_store.dart:19`、`ugoira_repository.dart:83` 可注入、`update_providers.dart:37`） | — | — |

---

## E. 测试

### E1. 布局与组织

- 71 个测试文件全部**平铺**于 `test/` 根，无按 `core/`/`features/` 的子目录镜像；文件名前缀聚类：`illust_*` 5、`updater_*` 4、`account_*` 4、`widget_*` 3、`ugoira_*` 3、`download_*` 3、`bookmark_*` 3、其余各 1–2。
- 规模：`test(`+`testWidgets(` 共 626（其中 `testWidgets` 95），`group(` 65。最大文件 `download_manager_test.dart` 1,212 行、`restricted_compat_network_test.dart` 1,133、`illust_detail_page_test.dart` 814、`doh_resolver_test.dart` 768、`settings_test.dart` 695。
- `dart_test.yaml` 全局 `timeout: 60s`（注释：全套约 600 测试）。
- 共享 fixture 仅 `test/helpers/illust_fixtures.dart`（74 行，`illustJson()`/`parseIllust()`），被 10 个文件 import。
- Golden：1 个（`icon_font_test.dart:153` → `test/goldens/home_bar.png`）；`test/failures/` 下 4 个 png 为未跟踪的失败产物。
- `test/zz_diag_tabbar_geometry_test.dart` 文件头写 "One-off diagnostic (not committed)"，但 `git ls-files` 显示已被跟踪。
- 真实 socket：`download_manager_test.dart:908` group `'HttpDownloadTransport over real sockets (environment-flaky)'`；`tls_sni_behaviour_test.dart` 用 loopback `ServerSocket`；`ech_https_rr_real_test.dart` 名带 real 但实际是离线抓包字节回放。

### E2. 假实现重复

`test/` 中私有类 141 个。**同名假实现跨文件重复**（按类名统计）：

| 假实现类名 | 重复文件数 | 文件 |
|---|---|---|
| `_FakeMetadataRepository` | 8 | `bookmark_flow`、`illust_detail_controller`、`illust_detail_page`、`ranking_feed`、`recommended_feed`、`recommended_home`、`related_illust_repository`、`user_profile` |
| `_FakeCredentialStore` | 8 | 同上 8 个文件 |
| `_CredentialStore` | 5 | `comments_replies`、`pixiv_http_client`、`search_catalog`、`settings`、`widget_feed_loader` |
| `_StubAccountStore` | 4 | `bookmark_store`、`bookmark_switch_button`、`comments_replies`、`feed_generation_commit` |
| `_FakePlatform` | 3 | `reverse_image_search_page`、`updater_download`、`updater_manifest` |
| `_StaticMetadataRepository` / `_StaticCredentialStore` | 2 / 2 | `home_page`、`startup_gate` |
| `_MetadataRepository` / `_AccountMetadataRepository` | 2 / 2 | `pixiv_http_client`+`widget_feed_loader` / `comments_replies`+`search_catalog` |
| `_FakeClient`、`_FakeNewFeedRepository`、`_FakeRepository`、`_ScriptedHop`、`_ApiFixture`、`_SwitchableAccountStore`、`_UnusedTransport` | 各 2 | — |

按 `^class _\w*(CredentialStore|MetadataRepository)` 统计：凭据存储假实现出现在 16 个文件、账号元数据仓储假实现出现在 15 个文件，并集 **16 个文件**，共 10 种类名（`_FakeCredentialStore`、`_CredentialStore`、`_StaticCredentialStore`、`_FailingCredentialStore`、`_NoCredentialStore`、`_FakeMetadataRepository`、`_MetadataRepository`、`_AccountMetadataRepository`、`_StaticMetadataRepository`、`_EmptyMetadataRepository`）。对应的 provider override 频次：`credentialStoreProvider.overrideWith` 32、`accountMetadataRepositoryProvider` 31、`oauthServiceProvider` 17、`accountStoreProvider` 15、`settingsRepositoryProvider` 9、`searchRepositoryProvider` 9、`pixivHttpClientProvider` 9（`overrides:` 共 62 处/27 文件，`ProviderContainer(` 36 处）。`SharedPreferencesAsyncPlatform.instance = InMemorySharedPreferencesAsync.empty()` 在 19 个文件重复。

### E3. 平台/网络假实现的使用方式

- `network_image_mock`：`mockNetworkImagesFor(...)` 在 8 个文件使用，`illust_detail_page_test.dart` 单文件 19 次逐用例包裹。
- 平台通道：`TestDefaultBinaryMessengerBinding…setMockMethodCallHandler` 仅 `account_transfer_clipboard_test.dart`（4 处）；其余通道通过项目自有接口注入假实现（`_FakeReverseImageInputPlatform`、`_FakeMediaStoreSession`、`_FakeSafDocument` 等）。
- WebView：`WebViewPlatform.instance = _FakeWebViewPlatform()`（`login_navigation_test.dart:185`）+ `_FakeWebViewController`/`_FakeNavigationDelegate`/`_FakeWebViewWidget`。
- SQLite：`databaseFactoryFfi`（`history_persistence_test.dart:25,39,255`，依赖宿主机 sqlite3 动态库）。

### E4. 锁内部名称而非行为的用例（示例）

1. `test/hero_transition_test.dart:107,191,282,403,417,426`：`find.byWidgetPredicate((w) => w.runtimeType.toString() == '_GlobalRectClip')` —— 以**私有**渲染类名字符串定位。
2. `test/illust_detail_page_test.dart:547–548`：`widget.runtimeType.toString() == 'PersonAvatar' && (widget as dynamic).imageUrl != null` —— 类名字符串 + `dynamic` 反射字段。
3. `test/related_illust_repository_test.dart:121–125`：`isA<Object>().having((e) => e.runtimeType.toString(), 'type', 'ApiParseError')` —— 有 `ApiParseError` 类型可用却比对类名字符串。
4. `test/updater_flavor_contract_test.dart`：断言 Gradle/Kotlin/Manifest **源文本**（`contains('object DistributionUpdaterChannel')`、`isNot(contains('HttpURLConnection'))`、`contains('UPDATE_PUBLIC_KEY_DER_B64')`），锁定文件内容而非构建行为。

---

## F. LLM 可维护性信号

| 信号 | 观测 |
|---|---|
| 模块级文档 | `lib/` 内 **0** 个 README/`*.md`；`library;` 指令 3 个（`android_platform.dart:5`、`illust_caption.dart:15`、`dns_message.dart:4`）；200 个文件中 144 个的首个非 import 行是 `///`（文件级或首类文档） |
| 公开声明文档密度（`class/enum/mixin/typedef` 上方紧邻 `///`） | `lib/core/` 517 个公开声明，296 有文档（**57%**）。分模块：`bookmark` 88%、`auth` 86%、`mutation` 85%、`widget` 84%、`paging` 80%、`settings` 78%、`download` 73%、`ugoira` 72%、`network`/`platform`/`entity` 69%、`history` 62%、`comments`/`user` 55%、`new` 50%、`profile` 34%、`reverse_image` 31%、`search` 30%、`novel` 28%、**`updater` 13%**、`i18n` 0%。`lib/features/` 115 个公开声明 57%；`lib/app/` 12 个 50%。`///` 行 2,407，`//` 行 912 |
| 命名语言 | 标识符全部英文（未检出含 CJK 的标识符）。注释：817 行含 CJK/20 文件，其中 766 行是 `replica_strings.dart` 数据；其余 51 行散布 19 文件，例如 `illust_caption.dart:3–14` 整段模块文档为中文、`network_probe_page.dart:42–46` 中文、`illust_detail_page.dart:1036–1037`；大量文档以 PRD 编号引用需求（`R4`/`C5`/`C9`/`C10`/`C12`/`U2`、"三档之详情档"）而未指向具体 task 文件；`beta56` 51 处/39 文件、`PixEz` 26 处/8 文件作为行为依据被引用 |
| `// ignore:` | `lib/` **1** 处（`network_contracts.dart:259 avoid_positional_boolean_parameters`）；`test/` 1 处（`avoid_print`）。`flutter analyze` 0 问题，但仅默认 `flutter_lints` 规则集 |
| `dynamic` | 126 个 token/32 文件，其中 105 行是 `Map<String, dynamic>`；显式 `final dynamic decoded` 5 处（见 C2）；`(widget as dynamic)` 仅出现于测试 |
| 后缀 `!` 非空断言（近似正则） | 约 311 处。热点：`widget_snapshot.dart` 12、`reverse_image_search_page.dart` 11、`update_platform.dart` 11、`novel_entity.dart` 11、`ugoira_export.dart` 10、`intent_router.dart` 10、`comment_translation.dart` 9、`network_policy.dart` 8、`web_profile_repository.dart` 8、`illust_entity.dart` 8。`late` 52 处/26 文件 |
| TODO/FIXME/HACK | 0 |
| 可写静态全局 | `HomeShellMetrics.bottomNavTop/bottomNavHeight`、`RhttpGate.ready`（见 B4） |
| 未提交/诊断遗留 | `test/zz_diag_tabbar_geometry_test.dart` 自述"one-off, not committed"却被跟踪；`test/failures/` 4 个未跟踪 png |

### F2. `.trellis/spec/` 是否描述了真实约定

| 文件 | 行数 | 状态 |
|---|---|---|
| `backend/index.md` | 39 | 表格把 5 个文档全部标 "To fill"（L17–L21） |
| `backend/directory-structure.md`、`database-guidelines.md`、`error-handling.md`、`logging-guidelines.md`、`quality-guidelines.md` | 51–54 | **模板**：`(To be filled by the team)` + `src/ ├── ...` 占位 |
| `backend/in-app-web-profile.md` | 126 | **已填**（资料编辑 Web 传输契约，7 节） |
| `frontend/index.md` | 39 | 表格把 6 个文档全部标 "To fill"（L17–L22），**与实际不符** |
| `frontend/state-management.md` | 1,259 | **已填**：17 个真实契约（`IllustStore.mergeAll`、`BookmarkStore`、`PagedFeedController`、`FeedRequestContext`、`NetworkAccessPolicy`、`DownloadManager`、`IntentRouter`、Widget 快照、Updater 等） |
| `frontend/component-guidelines.md` | 294 | **已填**：详情页转场、Tab 动画、共享下拉刷新 3 个契约（前 55 行仍是模板段） |
| `frontend/quality-guidelines.md` | 148 | **已填**：工具链、禁止/必需模式、测试要求、WSL loopback 说明 |
| `frontend/directory-structure.md`、`hook-guidelines.md`、`type-safety.md` | 51–54 | **模板** |
| `guides/*.md` | 97–327 | 已填（通用思维指南，非项目专属） |

即：真实约定几乎全部集中在 `frontend/state-management.md` 一个 1,259 行文件里；两份 `index.md` 的状态列已过期；目录结构、错误处理、数据库、类型安全 4 个最贴近本 task 的主题仍是空模板。

---

## G. 重构候选

按"证据强度 × 影响面 × 与四个目标的相关度"排序；行号/计数均见上文。

| 优先级 | 候选 | 证据（文件:行、计数） | 目标形态（一句话） | 影响面（大致文件数） | 风险 | 相关目标 |
|---|---|---|---|---|---|---|
| 1 | 拆分 `settings_page.dart` | 2,088 行、14 个公开页面类（L89–L1827）、`setState` 25 处、`ScaffoldMessenger` 9 处、第二个 `MePage`（L349） | 每个设置子页一个文件，`MePage` 只保留一处定义 | 1 → 约 12–14 个文件；调用方 `home_page.dart`、`user_page.dart` 改 import | 低（纯搬移，无逻辑变化；`settings_test.dart` 695 行需改 import） | 可维护性 / 简洁性 |
| 2 | 打破 12 文件页面 import 环：把 `IllustCard` 迁出 `recommended_illust_page.dart` | `IllustCard` 定义于 `features/home/recommended/recommended_illust_page.dart:236`，被 ranking/new/profile/search/history/related 6 个模块 import；SCC 含 12 个页面文件；`SliverMasonryGrid.count` 8 处形态一致 | 共享卡片与瀑布流 sliver 移到 `lib/app/`（或 `features/illust/widgets/`），feature 间只经路由入口互相引用 | 8–10 | 低-中（Hero tag、缓存键契约见 `component-guidelines.md` 转场契约，需保持） | 可维护性 / 简洁性 |
| 3 | 主机/purpose 契约收敛到 `PixivClientIdentity`/`PixivDestinationRegistry` 单点 | 主机字面量 5 处再声明：`network_contracts.dart:53–63`、`network_policy.dart:176–180`、`network_fast_route_store.dart:27–35`、`network_probe_page.dart:56–59`、`login_intercept_controller.dart:135–139`；`illust_detail_repository.dart:18` 自建 host；`PixivHeaders` 缺 image/download 头构造，`'Referer'` 拼装 4 处 | registry 从 identity 常量派生，probe 页与登录拦截从 registry 枚举；`PixivHeaders.image()` 供 4 个下载/图片调用点 | 8–9 | 中（触及 allowlist 与 TLS 语义边界，PRD R1/R5 要求不改变语义；只能做等价重排） | 可维护性 / 可迭代性 |
| 4 | 统一 i18n 取字入口，消除 17 个 `_xxxText` 包装 | 17 个等价包装函数（C2 表）；`Localizations.localeOf(context).toLanguageTag()` 38 处/26 文件；`ReplicaStrings.text` 缺键运行时 `!` 抛出（L1906）；434 键 × 4 语言 1,924 行单文件 | 一个 `BuildContext` 扩展或 `ReplicaStrings.of(context)`；键表按语言拆文件或生成常量键 | 26+1 | 低（纯调用替换）；键表拆分中（`i18n_network_keys_test.dart` 已在校验键） | 可维护性 / 简洁性 / 升级 |
| 5 | 统一 feed 尾部/空态/错误 widget | 32 个私有 `_*Tail/_*Error/_*Empty/_*Status` 类/14 文件；`PagedFeedState` 已是统一状态模型 | 在 `lib/app/widgets/` 提供 `FeedTail`/`FeedEmpty`/`FeedError` 3 个组件消费 `PagedFeedState` | 14 | 低-中（各页文案/按钮略有差异，需先列差异矩阵） | 简洁性 / 可维护性 |
| 6 | 抽取共享测试假实现 | 凭据/元数据假实现 10 种类名分布在 16 个测试文件（`_FakeCredentialStore`/`_FakeMetadataRepository` 各 8 份、`_CredentialStore` 5、…）、`_StubAccountStore` 4；`credentialStoreProvider.overrideWith` 32 处；`InMemorySharedPreferencesAsync.empty()` 19 文件 | `test/helpers/` 增加 `fake_account.dart`（凭据/元数据/账号 store 假实现 + 标准 overrides 列表）与 `test_prefs.dart` | 约 25 个测试文件 | 低（仅测试代码；PRD R6 禁止删测试，此为合并不删） | 可维护性 |
| 7 | 合并两份右滑路由实现 | `replicaRoute()`（`lib/app/navigation/replica_route.dart`，3 处）与 `ReplicaPageRoute`（`lib/app/replica_page_route.dart`，30 处）同一 300ms/easeInOutCubic 转场；`go_router` 声明未用（0 import） | 保留 `ReplicaPageRoute`，删函数版；`pubspec.yaml` 移除 `go_router`、`cupertino_icons`（需 R3 消费者搜索确认） | 3 + pubspec | 低 | 简洁性 / 升级 |
| 8 | 抽出 JSON 强制转换与 `_nextUrl` 公共 helper | `_positiveInt` 5 份、`_optionalString` 4 份、`_requiredString` 2 份（`comment_entity.dart:176`、`user_entity.dart:254–272`、`novel_entity.dart:1510–1535`、`update_manifest.dart:256–263`、`novel_repository.dart:319–330`）；`_nextUrl(Object?)` 5 份 | `lib/core/network/json_read.dart`（或 `entity/json_fields.dart`）一处定义，各实体/仓储 import | 9–10 | 低（纯函数；现有实体解析测试覆盖） | 简洁性 / 可维护性 |
| 9 | 拆分 `network_policy.dart`：`PixivNetworkFactory` 与图片 `CacheManager` 所有权独立成文件 | 1,198 行内含 `NetworkAccessPolicy`（L30）、`PixivNetworkFactory`（L1143）、`imageCacheManager`（L1148–L1190）；`flutter_cache_manager` 因此被网络策略文件 import（L5） | `network_policy.dart` 只留策略；`pixiv_network_factory.dart` + `image_cache.dart` 各自持有；`compat_network.dart` barrel 相应更新 | 3–4（+ `pixiv_image.dart`、`download_providers.dart` import） | 中（不改行为，但该文件受 08-29/09-01-network-perf-ab 契约约束，需等其稳定；`restricted_compat_network_test.dart` 1,133 行回归） | 可迭代性 / 可维护性 |
| 10 | 消除 `core → features` 反向依赖 | `core/widget/widget_feed_loader.dart:21` import `features/home/recommended/recommended_repository.dart`；`app/app.dart:14` ↔ `features/onboarding/*` 互引 | 把 `recommended_repository.dart`（含 `RecommendedIllustController`）移入 `core/`（与其它 13 个 core 仓储一致）；`startup_gate.dart` 上移到 `lib/app/` 或 onboarding 不再依赖 `lib/app` | 4–6 | 低 | 可维护性 |
| 11 | 统一 repository/controller 摆放规则并写入 spec | 18 个 repository：13 在 `core/`、5 在 `features/`；控制器定义在 `*_repository.dart` 内（`ranking_repository.dart:112`、`recommended_repository.dart:87`）；`frontend/directory-structure.md`、`backend/directory-structure.md` 仍是模板 | 规则："数据源/实体/控制器进 `core/<domain>/`，`features/` 只放 widget"；同步填写两份 `directory-structure.md` 与 `index.md` 状态列 | 5 个源文件 + 3 个 spec | 低 | 可维护性（LLM 可发现性） |
| 12 | 三个 `ChangeNotifier` 控制器归入 Riverpod 生命周期 | `ProfileEditController`（`profile_edit_page.dart:94` 手动 new）、`ReverseImageSearchController`（`reverse_image_search_page.dart:45`）、`NovelReaderController`（`novel_reader.dart:233`）；这两页也是 `dart:io`/平台通道泄漏点（B2） | `ChangeNotifierProvider.autoDispose(.family)` 或改 `Notifier`，平台适配器经 provider 注入，页面不再 `new MethodChannel*` | 6（3 控制器 + 3 页面） | 中（`profile_edit_test.dart` 512 行、`reverse_image_search_*_test.dart` 依赖当前构造注入方式） | 可维护性 / 可迭代性 |
| 13 | SQLite 栈：明确 `sqflite_common_ffi` 的角色 | `history_database.dart:27–31` 运行时按平台选栈；两者均在 `dependencies`（`pubspec.yaml:32–33`）；生产 FFI 消费者仅桌面平台；测试依赖 `databaseFactoryFfi` | 若桌面非目标：FFI 移到 `dev_dependencies` 并由测试注入 factory（构造器已支持 `factory:` 参数）；若桌面是目标：保持并在 `database-guidelines.md` 写明 | 2 + pubspec + 1 测试 | 低-中（需确认跨平台可构建性要求，PRD R5 提到"已有跨平台源码的可构建性"） | 简洁性 / 可迭代性 |
| 14 | `SharedPreferences` 访问收敛 | 6 处直接 `SharedPreferencesAsync()`；`blocked_tags.dart:18,36` 每次调用 new 且不可注入；键命名空间 `replica.*`/`pixiv.*`/`pixivfunc.*`/裸键 4 种 | 一个 `preferencesProvider`（`Provider<SharedPreferencesAsync>`）供 6 处注入；键常量集中一处并标注版本 | 6 + 19 个测试文件的 `InMemorySharedPreferencesAsync` 样板可随 #6 一起收敛 | 低（键值不变，仅注入方式） | 可迭代性 |
| 15 | 拆分 `illust_detail_page.dart` 与 `user_page.dart` | 1,388 行含自定义 RenderObject（L344–L406）+ 12 个私有 widget + 链接路由判断（L1271）；`user_page.dart` 959 行含 `MePage` 与 3 种 feed | RenderObject 与 caption 富文本移出为独立文件；`user_page` 的 3 个 feed 各自成文件 | 2 → 约 8 | 中（`hero_transition_test.dart` 以 `'_GlobalRectClip'` 类名字符串定位，重命名/搬移会破坏该测试，需同时改为公开类型或 Key） | 可维护性 |
| 16 | 异常体系整理 | 49 个互不相关的 `implements Exception` 类 + `sealed ApiError` 7 子类；错误→文案函数按页各写（4 处） | 不新增框架；按域给出少量 sealed 基类（download/ugoira/network 已各自成簇），并在 `backend/error-handling.md` 填写真实分类与传播规则 | 视范围 10–20 | 中（PRD R2 要求不改变错误分类语义；先只做文档 + 命名，不改 `catch` 行为） | 可维护性 |
| 17 | 测试去"类名字符串"断言 | `hero_transition_test.dart` 6 处 `'_GlobalRectClip'`、`illust_detail_page_test.dart:547 'PersonAvatar'` + `as dynamic`、`related_illust_repository_test.dart:121 'ApiParseError'` | 改为 `find.byType`/`isA<T>()`/`Key` | 3 | 低 | 可维护性 |
| 18 | 修正 `.trellis/spec` 索引与空模板 | 两份 `index.md` 全部标 "To fill"，而 `state-management.md`（1,259 行）等 4 份已填；`directory-structure`×2、`error-handling`、`database-guidelines`、`type-safety` 仍为模板 | 更新索引状态列；随上述候选逐项填入真实约定（目录规则、错误分类、单 DB、`dynamic`/`!` 约束） | 8 个 spec 文件 | 无（文档） | 可维护性（LLM 发现性） |

---

## Caveats / Not Found

- 后缀 `!` 计数基于近似正则（`[A-Za-z0-9_)\]]!` 后接 `.`/`)`/`]`/`;`/`,`/空格/行尾），可能少量误匹配字符串内容或漏掉换行处的断言；只用于定位热点，不作精确指标。
- 文档密度只统计"声明上方紧邻 `///`"，不统计方法级文档；`typedef`/`extension` 少量声明可能因多行写法未被正则捕获。
- import 图按相对路径与 `package:pixiv_func/` 解析，不含 `part` 文件（项目内无 `part`）。
- 未运行 `flutter test`（不在本次范围；`dart_test.yaml` 注释称约 600 个测试，静态计数 626 个 `test/testWidgets` 声明）。
- 未审计 `plugins/rhttp/`、`android/`、Rust 与生成链路（属其它研究主题）。
- 工作树含未提交改动，若在 commit 边界重复统计，数字会有小幅差异。
