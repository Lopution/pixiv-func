# Research: Dart/Flutter 代码中的现代化缺口与旧版残留

- **Query**: 盘点 Dart/Flutter 代码中"现代化不彻底"的地方：analyzer 信号、过时 API、从 beta56 原版移植的痕迹、死代码、风格一致性，并给出现代化候选表
- **Scope**: internal（`lib/`、`test/`、`pubspec.yaml`、`analysis_options.yaml`、`.trellis/` 归档笔记）
- **Date**: 2026-09-07
- **工具链**: Flutter 3.47.0（`/opt/flutter-3.47.0/bin`），Dart SDK `^3.11.0`（pubspec）
- **测量基线**: 工作树快照（含未提交改动）。`lib/` 200 个 Dart 文件、51 772 行；`test/` 72 个文件、21 903 行。其中 15 个 `lib/` 文件、2 个 `test/` 文件为**未跟踪的进行中文件**（`git ls-files --others`，属于其他 09-01 task），另有 30+ 个已跟踪文件处于修改状态。下文所有计数都包含这些工作树状态。

---

## 0. 结论速览

| 维度 | 观测 |
|---|---|
| `flutter analyze` | **0 issues**（`flutter_lints` 6.x 默认规则集）|
| `dart format --set-exit-if-changed lib test` | **46 个文件**需要重排（26 lib / 20 test；其中 9 个为未跟踪文件）|
| 已废弃 Flutter API（WillPopScope / withOpacity / 旧 textTheme 名 / accentColor / ButtonBar / RaisedButton / Scaffold.of().showSnackBar / MediaQuery.of().size / textScaleFactor / MaterialStateProperty / colorScheme.background 等） | **全部 0 命中**，已完成迁移 |
| 主题 | `useMaterial3: false`（`lib/app/theme/replica_theme.dart:19`）—— 有意保持 Material 2 视觉以复刻 beta56 |
| 路由 | 100% 命令式 `Navigator.of(context).push(ReplicaPageRoute(...))`；`go_router` 已声明但 **0 引用** |
| JSON | 全部手写 `fromJson`，无 json_serializable / freezed / build_runner |
| i18n | 自研 `ReplicaStrings` 字典（1 924 行 / 113 KB / 434 key × 4 语言），运行时字符串 key 查表；27 个逐页 `_xxxText()` 包装函数 |
| 死代码 | 4 个零引用文件（含 198 行的 `login_intercept_controller.dart`）、13 个零引用公共声明、2 个零引用依赖、图标字体 14 个 glyph 仅用 6 个 |
| 旧版命名残留 | "replica" 241 处 / 50 文件；8 个 `replica_*.dart`；两套等价的 Replica 路由构造器 |

---

## A. Analyzer 信号

### A.1 当前结果

```
$ flutter analyze --no-pub
Analyzing Pixiv-func...
No issues found! (ran in 5.2s)
```

- `analysis_options.yaml`（42 行）：仅 `include: package:flutter_lints/flutter.yaml`；`analyzer.exclude` 排除 `build/**`、各平台目录和 `plugins/rhttp/rhttp/test/**`；`linter.rules` 为空（只有注释模板）。没有 `language:` 段（`strict-casts` / `strict-inference` / `strict-raw-types` 均未开启）。
- CI（`.github/workflows/ci.yml:18-19`）只跑 `flutter analyze` 和 `flutter test`，**没有** `dart format --set-exit-if-changed`。
- `// ignore:` 全仓库仅 2 处：
  - `lib/core/network/compat/network_contracts.dart:259` — `avoid_positional_boolean_parameters`（该规则并不在 flutter_lints 中，属于预防性 ignore）
  - `test/zz_diag_tabbar_geometry_test.dart:72` — `avoid_print`
- `// TODO` / `// FIXME` / `// HACK` / `// XXX`：**0**。项目内 `@deprecated` / `@Deprecated`：**0**。

### A.2 格式漂移（`dart format`）

`dart format --output=none --set-exit-if-changed lib test` → `Formatted 272 files (46 changed)`，exit 1。需要重排的 46 个文件：

- `lib/`（26）：`app/icons/app_icons.dart`、`app/person_avatar.dart`、`app/replica_page_route.dart`、`app/theme/replica_theme.dart`、`app/widgets/replica_scaffold.dart`、`app/widgets/replica_switch_tile.dart`、`core/auth/account_repository.dart`、`core/auth/account_transfer.dart`、`core/auth/credential_store.dart`、`core/download/download_manager.dart`、`core/download/naming_rule.dart`、`core/entity/illust_caption.dart`、`core/entity/illust_entity.dart`、`core/network/api_error.dart`、`core/network/compat/network_probe.dart`、`core/network/compat/policy_download_transport.dart`、`core/network/pixiv_client_identity.dart`、`core/novel/novel_repository.dart`、`core/platform/saf_tree.dart`、`core/reverse_image/sauce_nao_navigation_policy.dart`、`core/settings/blocked_tags.dart`、`core/user/user_repository.dart`、`features/illust/detail/related_illust_repository.dart`、`features/illust/detail/related_illusts_section.dart`、`features/onboarding/welcome_page.dart`、`features/search/reverse_image_search_page.dart`
- `test/`（20）：`account_store_test.dart`、`account_transfer_clipboard_test.dart`、`account_transfer_test.dart`、`dns_message_test.dart`、`doh_resolver_test.dart`、`helpers/illust_fixtures.dart`、`illust_caption_test.dart`、`illust_detail_controller_test.dart`、`illust_store_test.dart`、`mutation_ownership_test.dart`、`profile_edit_test.dart`、`recommended_home_test.dart`、`related_illust_repository_test.dart`、`restricted_compat_network_test.dart`、`sauce_nao_navigation_policy_test.dart`、`tls_sni_behaviour_test.dart`、`token_refresh_gate_test.dart`、`updater_about_test.dart`、`web_profile_repository_test.dart`、`zz_diag_tabbar_geometry_test.dart`

其中 9 个是未跟踪的进行中文件。抽样 `lib/app/icons/app_icons.dart`：格式化 diff 85 行（14 个 `IconData(...)` 单行超过 80 列）。

### A.3 可选更严格规则的现状估算（grep 估算，非 analyzer 精确值）

| 规则 / 选项 | 估算方法 | 当前估算违规数 |
|---|---|---|
| `prefer_final_locals` | `^\s+var \w+` 局部声明 | **92 处 / 31 文件**（上界；部分确有重新赋值，如 `paged_feed_controller.dart:181-203` 的 `page/visible/nextCursor/refill`）|
| `strict-raw-types` | `is/as (Map\|List\|Set\|Iterable)` 后无类型参数（`rg -P`）| **12 处**（`download_recovery.dart:294,448`、`illust_entity.dart:320,357,358`、`comment_translation.dart:140,143`、`web_profile_repository.dart:436,457`、`account_transfer.dart:125`、`novel_entity.dart:631`、`login_intercept_controller.dart:60`）|
| `strict-casts` | `json['…']` 索引站点 158 处；显式 `as String?/int?/bool?/…` 143 处 / 29 文件；`as Map<String, dynamic>` 6 处 / 3 文件 | 隐式 downcast 无法 grep 精确；上界 ≈ 158 个索引读取中未显式 `as`/`is` 的部分 |
| `avoid_dynamic_calls` | `final dynamic decoded` 5 处（`bookmark_repository.dart:59`、`comment_repository.dart:230`、`comment_translation.dart:125,249,484`）+ `dynamic response` 参数 2 处（`bookmark_repository.dart:52`、`comment_repository.dart:224`）；后续均先 `is Map<…>/is List` 收窄 | **≤ 7**，很可能接近 0 |
| `lines_longer_than_80_chars` | `^.{81,}$` | **133 行 / 34 文件**（`replica_strings.dart` 63、`app_icons.dart` 14、`replica_theme.dart` 9）|
| `prefer_single_quotes` | 非注释行双引号字面量 | **0**（已满足）|
| `public_member_api_docs`（仅顶层类型口径） | 643 个公共顶层类型，其中前一行无 `///` | **275 个**（约 43%）|
| `unreachable_from_main` | 见 D 节 | ≥ 13 个零引用声明 + 4 个零引用文件的全部内容 |
| `prefer_relative_imports` / `always_use_package_imports` | lib 相对导入 622 / package 导入 0；test package 导入 420 / 相对 0 | lib 内已一致（相对）；test 内已一致（package）；两树之间风格不同 |
| `sort_pub_dependencies` | pubspec `dependencies` 顺序 `meta, flutter, flutter_localizations, cupertino_icons, …` | **1**（未排序）|
| `unawaited_futures` | 不可 grep 精确；项目已使用 `unawaited(` **62 处 / 25 文件** | 无法估算 |
| `avoid_print` | 已在 flutter_lints 中 | 1（test，已 ignore）|
| `flutter_style_todos` | 无 TODO | 0 |

---

## B. 已废弃 / 旧式 Flutter & Dart API 使用

### B.1 逐项计数（`lib/` + `test/`）

| 模式 | lib 命中 / 文件 | test 命中 / 文件 | 备注 |
|---|---|---|---|
| `WillPopScope` | 0 / 0 | 0 / 0 | 已用 `PopScope` ×2（`profile_edit_page.dart:150`、`home_page.dart:294`），且为当前 API `onPopInvokedWithResult`（`:152`、`:296`）|
| `withOpacity(` | 0 / 0 | 0 / 0 | 已用 `withValues(alpha:)` ×3（`person_avatar.dart:39`、`profile_header_delegate.dart:367`、`illust_detail_page.dart:921`）|
| `Color.value`（`.value` 与 color 同现） | 0 / 0 | 0 / 0 | `.value` 共 183 处均为 `AsyncValue`/`TextEditingController`/enum 等 |
| `textTheme.headline*/bodyText*/subtitle*/caption/button/overline` | 0 / 0 | 0 / 0 | |
| `accentColor`、`primaryColorBrightness`、`ButtonBar`、`RaisedButton/FlatButton/OutlineButton`、`Scaffold.of(context).showSnackBar` | 0 | 0 | |
| `MaterialStateProperty` / `MaterialState` | 0 | 0 | `WidgetStateProperty` 也为 0（未用到状态属性）|
| `colorScheme.background/onBackground/surfaceVariant` | 0 | 0 | 已用 `surfaceContainerHighest/surfaceContainer` 15 处 / 11 文件 |
| `RawKeyboardListener/RawKeyEvent` | 0 | 0 | |
| `MediaQuery.of(context).size` | 0 / 0 | 0 / 0 | 使用细粒度 `MediaQuery.sizeOf/heightOf/paddingOf/viewPaddingOf/maybeOf` **12 处 / 9 文件**（如 `login_page.dart:176,184`、`history_page.dart:348`、`bookmark_switch_button.dart:64-65`、`user_page.dart:316`）|
| `textScaleFactor` | 0 | 0 | `TextScaler.noScaling` ×1（`novel_layout.dart:710`）|
| `useMaterial3` | 1 / 1 | 0 | `replica_theme.dart:19` 显式 **`useMaterial3: false`** —— 主题是 **Material 2**；同时使用 `ColorScheme.fromSeed`（`:31`）与 Flutter 3.47 的 `AppBarThemeData`/`BottomNavigationBarThemeData`/`BottomAppBarThemeData`/`TabBarThemeData`（`:54,68,73,78`）|
| `Navigator.*` | 66 / 25 | 0 | `Navigator.of` 55、`Navigator.pop` 11、`Navigator.maybeOf` 1；`Navigator.push(` 直呼 0、`pushNamed` 0、`MaterialPageRoute` 0。`MaterialApp` 仅 `home:`（`app.dart:102-118`），无 `routes`/`onGenerateRoute`/`MaterialApp.router` |
| `showDialog` | 6 / 4 | 0 | 全部 `AlertDialog` 确认框：`settings_page.dart:502,2030`、`comments_page.dart:116,241`、`user_page.dart:938`、`profile_edit_page.dart:124` |
| `showModalBottomSheet` | 4 / 4 | 0 | `history_page.dart:562`、`bookmark_switch_button.dart:49`、`search_filter_sheet.dart:10`、`follow_switch_button.dart:35` |
| `Timer.periodic` | 1 / 1 | 0 | `ugoira_scheduler.dart:89`（帧驱动）|
| `Timer(` | 2 / 2 | 0 | `root_back_coordinator.dart:39`、`search_autocomplete_controller.dart:84`（debounce）|
| `print(` | 0 | 1 / 1 | `test/zz_diag_tabbar_geometry_test.dart:73` |
| `debugPrint(` | 23 / 5 | 0 | `widget_feed_loader.dart` 12、`widget_background.dart` 6、`widget_coordinator.dart` 2、`widget_channel.dart` 1、`features/settings/network_probe_page.dart` 2；无 `dart:developer`/logger 抽象 |
| `late` | 52 / 26 | 44 / 20 | 最多：`settings_page.dart` 7、`network_settings_page.dart` 4、`profile_edit_page.dart` 4、`home_page.dart` 4 |
| `!` 空断言（`x!.`/`x!;`/`x!,`/`x!)` 口径） | **309 / 85** | — | Top 10：`widget_snapshot.dart` 12、`reverse_image_search_page.dart` 11、`update_platform.dart` 11、`novel_entity.dart` 11、`ugoira_export.dart` 10、`intent_router.dart` 10、`comment_translation.dart` 9、`reverse_image_platform.dart` 8、`web_profile_repository.dart` 8、`network_policy.dart` 8（并列 `illust_entity.dart` 8）|
| `dynamic`（整词） | **126 / 32** | 41 / 21 | Top 10：`illust_entity.dart` 12、`search_repository.dart` 9、`novel_entity.dart` 9、`web_profile_repository.dart` 8、`comment_translation.dart` 8、`user_entity.dart` 7、`novel_repository.dart` 6、`pixiv_http_client.dart` 6、`comment_repository.dart` 6、`app_settings.dart` 5（并列 `download_recovery.dart` 5、`oauth_service.dart` 5）。其中 `Map<String, dynamic>` 105 处 / 30 文件，非 Map 用法 21 处 |
| `Map<String, Object?>` | 24 / 15 | — | 与 `Map<String, dynamic>` **在 5 个文件中并用**：`download_recovery.dart`、`settings_repository.dart`、`update_download.dart`、`update_platform.dart`、`widget_snapshot.dart` |
| `Cupertino*` | 5 / 4 | 4 / 2 | `CupertinoActivityIndicator` ×3（`bookmark_switch_button.dart:211`、`follow_switch_button.dart:137`、`illust_detail_page.dart:478`）、`CupertinoSwitch` ×1（`app/widgets/replica_switch_tile.dart:27`）—— 复刻 beta56 视觉 |
| `Radio`/`RadioListTile` | 1 / 1 | — | `groupValue`/`RadioGroup` 均 0 |
| `Switch(`/`SwitchListTile` | 6 / 2 | — | `activeColor` 0（无 3.31 弃用项）|

### B.2 API 模型解析方式

- **无代码生成**：pubspec 无 `json_serializable`/`freezed`/`build_runner`；`lib/` 无 `part '*.g.dart'`、`@JsonSerializable`、`@freezed`。
- 手写解析入口：`factory X.fromJson` **8 处 / 7 文件**（`app_settings.dart:253`、`novel_entity.dart:620`、`comment_entity.dart:58`、`illust_entity.dart:286`、`ugoira_metadata.dart:31`、`download_recovery.dart:140,220`、`account.dart:50`）+ `static fromJson` 1 处（`download_destination.dart:78`）；`toJson` 9 处。
- `jsonDecode(` **19 处 / 15 文件**（`comment_translation.dart` 3、`pixiv_http_client.dart` 2、`oauth_service.dart` 2，其余各 1）。
- 模型文件（`*_entity.dart` / `*_models.dart`）共 **14 个、4 321 行**：`novel_entity.dart` 1 559 行（**32 个类**，含 markup 解析器）、`profile_edit_models.dart` 445、`illust_entity.dart` 430、`user_entity.dart` 289、`search_models.dart` 275、`history_models.dart` 257、`mutation_models.dart` 224、`comment_entity.dart` 214、`update_models.dart` 176、`comment_models.dart` 154、`bookmark_models.dart` 129、`follow_models.dart` 88、`profile_models.dart` 44、`new_feed_models.dart` 37。
- 手写样板：`copyWith` 23 处 / 17 文件、`operator ==` 21、`hashCode` 21（`Object.hash*` 19）、`@immutable` 81。
- **重复的私有 JSON 读取 helper**（同名同职责、各文件各写一份）：`_map(Object?)` 在 `novel_entity.dart:1538`、`user_entity.dart:249`、`novel_repository.dart:327`、`reverse_image_platform.dart:163`、`reverse_image_provider.dart:218`；`_firstString(Map, List<String>)` 在 `novel_entity.dart:1541`、`user_entity.dart:282`、`search_repository.dart:355`；另有 `app_settings.dart:496 _bool`、`update_manifest.dart:256 _string`。

---

## C. 从 beta56 原版移植的痕迹

### C.1 仓库内关于原版的参考记录（无原版源码副本）

- 仓库中**没有**原版源码副本（`find` 未发现 `pixiv_func_mobile`/`beta56` 目录，无 `package:pixiv_func_mobile` 引用）。
- 原版锚点由 Trellis 记录：`.trellis/tasks/archive/2026-09/08-26-pixiv-func-replica-v1/research/beta56-source-map.md`（固定 commit `c62b18cc…`，task → 原版路径映射表）；`.trellis/tasks/archive/2026-08/08-26-restore-icon-font/research/beta56-icon-font.md`（icon.ttf 来源、SHA-256、14 个 codepoint 校验）；`.trellis/tasks/09-01-func-1-0-hardening/research/audit-original-2026-09-01.md`（2026-09-01 的问题审计，含 C12/C13/C14/C16/C20 等与本节相关条目）。
- `NOTICE` 明确"行为参考为 beta56 源码 `svenfuss/pixiv_func_mobile@c62b18cc`"。

### C.2 命名与结构

| 信号 | 观测 |
|---|---|
| "replica" 一词 | **241 处 / 50 文件**（注释 + 标识符）。8 个文件以 `replica_` 命名：`lib/app/theme/replica_theme.dart`、`lib/app/widgets/replica_button.dart`、`replica_empty_state.dart`、`replica_scaffold.dart`、`replica_switch_tile.dart`、`lib/app/navigation/replica_route.dart`、`lib/app/replica_page_route.dart`、`lib/core/i18n/replica_strings.dart`。运行期标识符：`replicaTheme()`、`ReplicaPageRoute`、`replicaRoute()`、`replicaRouteObserver`、`ReplicaStrings`、`ReplicaLanguage`、`ReplicaScaffold`、`ReplicaButton`、`ReplicaSwitchTile`、`ReplicaEmptyState`。设置持久化 key 也带前缀：`settings_repository.dart:34-36` 的 `replica.guide_completed` / `replica.language` / `replica.theme`（作为 legacy key 迁移读取）|
| "beta56" 注释 | 约 40 处 doc 注释引用 beta56 行为（如 `comment_repository.dart:37`、`bookmark_switch_button.dart:20,62,247`、`ranking_page.dart:14`、`search_router.dart:13`、`recommended_illust_page.dart:233,252,390`、`illust_detail_page.dart:46,543`、`new_page.dart:16`、`widget_feed_loader.dart:26`、`intent_router.dart:35`）|
| **两套等价路由构造器** | `class ReplicaPageRoute<T> extends PageRouteBuilder<T>`（`lib/app/replica_page_route.dart`，27 行）—— 28 个调用点 / 20 个 lib 文件 + `test/hero_transition_test.dart` 5 处；`Route<T> replicaRoute<T>(WidgetBuilder)`（`lib/app/navigation/replica_route.dart`，16 行）—— **仅 3 个调用点**，全在 onboarding（`welcome_page.dart:67`、`language_page.dart:89`、`theme_page.dart:111`）。两者都是"从右滑入、300 ms、`easeInOutCubic`"，函数版缺 `settings` 参数。导航相关文件散落在 3 个目录：`lib/app/`、`lib/app/navigation/`、`lib/core/navigation/`（后者只有 `route_observer.dart` 7 行 + `home_shell_metrics.dart`）|
| "compat" | `lib/core/network/compat/`（9 文件）是产品功能"自动兼容网络"（restricted network policy）的实现，**不是**向后兼容层；`lib/core/network/compat_network.dart` 是其 barrel，**0 引用**（见 D.1）|
| "legacy" 一词 | 约 45 处，集中在 `settings_repository.dart:33-87`（beta56 旧设置 key 迁移）、`app_settings.dart:61-96,250-389`（`previewQuality`/`scaleQuality` 布尔 → 三档枚举迁移；`fromLegacyBool`）、`download_recovery.dart:46,479`、`download_manager.dart:23`、`network_contracts.dart:364`。审计 C14 提到的 `_runLegacyLadder` **已不存在**（0 命中）|

### C.3 三套视觉/图标系统

- **自定义图标字体** `assets/icon.ttf`（5 964 字节，icomoon；family `iconFont`，pubspec `fonts:`）。`lib/app/icons/app_icons.dart`（20 行）定义 **14 个** `IconData`（`0xe900`–`0xe90d`）。实际引用 **8 处 / 3 文件**，覆盖 **6 个** glyph：`home`、`n`、`ranking`、`search`（`home_page.dart`）、`follow`、`friend`（`user_page.dart`、`profile_header_delegate.dart`）。**未引用 8 个**：`addFollow`、`filter`、`me`、`toggle`、`pawoo`、`twitter`、`web`、`blocked`。但 `test/icon_font_test.dart:47-70` 断言全部 14 个 codepoint。
- **Material Icons**：`Icons.*` **194 处 / 32 文件，84 个不同图标**（`Icons.cloud_off` 14、`chevron_right` 12、`check` 11、`refresh` 8……）。
- **Cupertino**：`CupertinoActivityIndicator` ×3、`CupertinoSwitch` ×1（来自 Flutter SDK，`cupertino_icons` 包 0 引用）。
- 颜色：`FuncTokens`（`lib/app/theme/func_tokens.dart`，15 行、9 个 `Color(0x…)` 常量）被 6 个文件引用 20 次；`replica_theme.dart` 用 `ColorScheme.fromSeed` 派生；此外 `Colors.*` 直接使用 **37 处 / 15 文件**（`network_probe_page.dart` 9、`login_page.dart` 3、`recommended_illust_page.dart` 4、`image_viewer_page.dart` 4、`ugoira_viewer.dart` 3、`replica_theme.dart` 5 等），裸 `Color(0x…)` 字面量 16 处 / 6 文件（`func_tokens.dart` 9 之外：`pixiv_image.dart:21`、`ugoira_viewer.dart:149,580`、`recommended_illust_page.dart:334,350`、`search_page.dart:206`、`replica_theme.dart:71`）。

### C.4 评论表情/贴纸资源

- `assets/emojis/` **38 个 png**，`assets/stamps/` **40 个 jpg**，pubspec 以目录声明。
- 消费者唯一：`lib/core/comments/comment_assets.dart`（89 行）—— `commentEmojiNames` 38 项（`:3-43`）、`commentStampIds` 40 项（`:44-86`），`diff` 与目录列表**完全一致**（无多余、无缺失）；路径拼接函数 `commentEmojiAsset()`/`commentStampAsset()`（`:87,89`）。渲染点 4 处 `Image.asset(...)`：`comment_text.dart:34`、`comment_input.dart:171,178`、`comment_item.dart:207`。无运行时目录枚举（`AssetManifest`/`rootBundle` 0 命中）。

### C.5 硬编码中文 vs 本地化

- `Text('…中文…')` 形式在 `lib/features/`：**0**。
- 任意含汉字的单引号字面量（排除 `lib/core/i18n/`）：**8 行 / 3 文件**：
  - `settings_page.dart:575,577`、`language_page.dart:19,21` —— 语言自称（`'简体中文'`、`'日本語'`，属正常）
  - `settings_page.dart:1309-1310` —— 命名规则预览样例 `artist: '作者名', title: '作品标题'`（未走 i18n）
  - `sauce_nao_provider.dart:263-264` —— 匹配 SauceNAO 中文响应文案 `'没有匹配'`/`'没有结果'`
  - 双引号字面量含汉字：0（3 处 `"関連作品"` 均在注释）
- 本地化实现：`lib/core/i18n/replica_strings.dart` **1 924 行 / 113 689 字节**，`ReplicaStrings._values` 为 `Map<ReplicaLanguage, Map<String, String>>`，zh-CN/en-US/ja-JP/ru-RU 各 **434 个 key**（ru 少 1 个：`detailQuality`，运行时回退 zh）。查表 API `ReplicaStrings.text(language, key, [args])`（`:1900-1915`）与 `fromTag(tag, key)`（`:1917`），**缺失 key 时执行 `_values[zhCN]![key]!`** —— 抛 null-check 异常。
- 调用形态：每个页面各写一个包装函数 —— **27 个** `_xxxText(...)`（`_historyText`、`_settingsText`、`_networkText`、`_probeText`、`_profileText` ×2、`_profileEditText`、`_detailText`、`_ugoiraText`、`_bookmarkText`、`_newText`、`_novelText`、`_recommendedText`、`_loginText`、`_text` ×4 …），主体都是 `ReplicaStrings.fromTag(Localizations.localeOf(context).toLanguageTag(), key)`（例：`history_page.dart:25-30`、`settings_page.dart:33-38`、`login_page.dart:125-128`）。
- `flutter_localizations` 仅用于 `localizationsDelegates: GlobalMaterialLocalizations.delegates`（`app.dart:4,112`）；无 ARB / `gen-l10n` / `intl` / `AppLocalizations`。
- **观测到一个未定义 key**：`lib/features/settings/network_settings_page.dart:205` 使用 `'networkDohEndpointsInvalid'`，而 `replica_strings.dart` 四种语言只定义了 `networkDohEndpoints`（`:80,542,1019,1484`）；`test/i18n_network_keys_test.dart` 的 key 列表也未包含它。两文件当前均处于工作树已修改状态（`git status` 为 ` M`）。
- 代码中以字面 key 调用的数量约 303 个不同 key；132 个 zh key 未被本 grep 口径命中（可能通过变量/enum 拼接使用，也可能未使用；未逐一核实）。

### C.6 "1:1 移植"形态的文件

- **超过 150 行的方法/函数（5 个）**：`update_service.dart:194 UpdateService._checkOnce` 228 行；`download_manager.dart:528 DownloadManager._run` 187 行；`login_page.dart:149 _LoginPageState.build` 186 行；`recommended_illust_page.dart:243 IllustCard.build` 180 行；`profile_edit_controller.dart:223 ProfileEditController.submit` 167 行。
- **`build()` 方法 185 个**：>150 行 2 个、>100 行 11 个、>60 行 37 个。>100 行的另 9 个：`profile_edit_page.dart:290` 138、`illust_detail_page.dart:984 _InfoBlock` 132、`settings_page.dart:1178` 125、`settings_page.dart:993` 125、`related_illusts_section.dart:24` 124、`illust_detail_page.dart:773 _PageImageState` 118、`settings_page.dart:1388` 107、`recommended_illust_page.dart:28` 106、`comment_input.dart:48` 102、`search_filter_sheet.dart:54` 100。
- **最大文件**：`settings_page.dart` 2 088 行（**27 个类，含 10 个公开页面类** `SettingsPage/MePage/AccountSettingsPage/ThemeSettingsPage/LanguageSettingsPage/TranslateSettingsPage/TranslationCredentialsPage/BrowseSettingsPage/DownloadSettingsPage/DownloadDestinationPage/HistorySettingsPage/BlockedTagsPage/DownloadTasksPage/AboutSettingsPage`，`setState` 25 处、`late` 7 处、`showSnackBar` 9 处）；`illust_detail_page.dart` 1 388 行（15 类）；`user_page.dart` 959 行（16 类）；`novel_layout.dart` 926 行；`profile_header_delegate.dart` 713 行。
- **`setState(` 149 处 / 27 文件**，其中 25 处是两个 `ChangeNotifier` 控制器内部的 `_setState(`（`profile_edit_controller.dart` 14、`reverse_image_controller.dart` 11）；Widget 内真正的 `setState` **124 处 / 25 文件**：`settings_page.dart` 25、`ugoira_viewer.dart` 12、`comment_input.dart` 9、`search_filter_sheet.dart` 8、`history_page.dart` 7、`comment_item.dart` 6、`network_settings_page.dart`/`login_webview_page.dart`/`comments_page.dart` 各 5。
- Widget 类构成：`StatelessWidget` 95、`ConsumerWidget` 41、`StatefulWidget` 14、`ConsumerStatefulWidget` 23（共 738 个类，187 个私有）。
- **手写 `ScrollController` + 分页**：`history_page.dart:113,124,142-145`（`_scrollController.position.extentAfter < 400 → _loadMore()`，并自持 `_records/_hasMore/_loading/_generation` 状态，`setState` 7 处），**没有使用** `lib/core/paging/PagedFeedController`（该控制器被 18 个 lib 文件导入，是其余所有 feed 的分页实现）。另两处 `ScrollController`（`ranking_page.dart:27-56` 每模式一个、`new_page.dart:178`）用于 tab 保持滚动位置。
- **裸 HTTP 客户端**：`package:http` 在 13 个 lib 文件导入；`client ?? http.Client()` 默认参数 5 处（`oauth_service.dart:98`、`pixiv_http_client.dart:60`、`sauce_nao_provider.dart:33`、`comment_translation.dart:535`、`pixiv_download_transport.dart:55`）；`dart:io HttpClient()` 3 处（`network_probe.dart:474`、`secure_resolver.dart:790`、`update_service.dart:433`）。生产 wiring：Pixiv 目的地由 `pixivNetworkFactoryProvider` 注入 policy client（`pixiv_http_client.dart:418-432`、`account_store.dart:324`）；`commentTranslationServiceProvider`（`comment_translation.dart:532-536`）在生产中**直接 `http.Client()`**（第三方翻译服务，不在 Pixiv 策略范围）。`login_intercept_controller.dart:90` 与 `network_probe_page.dart:106` 构造 `http.Request` 手动发送。
- **三个 `ChangeNotifier` 控制器**（`ProfileEditController` `profile_edit_controller.dart:65`、`ReverseImageSearchController` `reverse_image_controller.dart:62`、`NovelReaderController` `novel_reader.dart:135`）不经 Riverpod（`ChangeNotifierProvider` 0 命中），由 StatefulWidget 手动 `addListener` + `setState`/`AnimatedBuilder` 消费（`profile_edit_page.dart:94,168,236`、`reverse_image_search_page.dart:45-50`、`novel_reader.dart:233-234`）。

### C.7 同一关切的重复实现

| 关切 | 观测 |
|---|---|
| 页面路由过渡 | `ReplicaPageRoute`（28 处）vs `replicaRoute()`（3 处），见 C.2 |
| SnackBar | **44 处 / 17 文件**直接 `ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(...)))`，无共享 helper；`.of` 与 `.maybeOf` 混用（`maybeOf`：`bookmark_switch_button.dart:192`、`profile_edit_page.dart:276`、`user_page.dart:95`、`follow_switch_button.dart:125`、`home_page.dart:222`）|
| i18n 查表 | 27 个逐页 `_xxxText` 包装（C.5）|
| 错误占位 widget | 14 个逐页私有 `_XxxError` 类：`_CommentError`、`_HistoryError`、`_NewError`、`_NewRefreshError`、`_NovelError`、`_ProfileFeedError`、`_RankingInitialError`、`_RecommendedError`、`_TailError`、`_SearchInlineError`、`_SearchResultError`、`_StartupError`、`_InitializationFailure`；公共的 `ReplicaEmptyState`/`SettingsLoadError` 在 `lib/app/widgets/` |
| 偏好存储访问 | `SharedPreferencesAsync()` 在 **7 个 store 中各自默认构造**：`blocked_tags.dart:18,36`、`download_recovery.dart:372`、`settings_repository.dart:30`、`update_download.dart:163`、`account_repository.dart:38`、`network_fast_route_store.dart:40`；密钥走 `FlutterSecureStorage`（`credential_store.dart`、`translation_credentials.dart`）|
| SQLite | `sqflite`（`history_repository.dart:4`、`history_database.dart:4`）+ `sqflite_common_ffi`（`history_database.dart:5,29-30`）：`_platformDatabaseFactory()` 在 Android/iOS 用 `sqflite.databaseFactory`，其他平台 `sqfliteFfiInit(); databaseFactoryFfi`（服务桌面/测试）。是有意的平台分支，非两套栈 |
| JSON helper | `_map`/`_firstString` 等 6 处重复（B.2）|
| JSON 容器类型 | `Map<String, dynamic>` 105 vs `Map<String, Object?>` 24；5 文件并用 |
| 图片 URL | 无重复构造：URL 直接来自实体字段（`imageUrls.medium/large/squareMedium`、`previewUrl(quality)`），Referer 常量集中在 `pixiv_client_identity.dart:52-55`；`ugoira_export.dart:685` 有一处手拼 `https://i.pximg.net/img-ugoira-export/…gif`（导出元数据用）|
| barrel/export | 6 个文件含 `export`：两个专用 barrel（`compat_network.dart` 7 行、`android_platform.dart` 16 行）**均 0 引用**；4 个"顺带 export"（`update_service.dart:11-12`、`paged_feed_controller.dart:16`、`download_task.dart:5`、`related_illusts_section.dart:12`）|

---

## D. 死代码与未使用面

### D.1 `lib/` 中无任何 lib 文件导入的 Dart 文件（脚本：解析 `import/export/part` 相对路径与 `package:pixiv_func/`）

| 文件 | 行数 | 说明 |
|---|---|---|
| `lib/core/network/compat_network.dart` | 7 | barrel，lib/test 均 0 导入 |
| `lib/core/platform/android_platform.dart` | 16 | barrel，lib/test 均 0 导入 |
| `lib/features/login/login_intercept_controller.dart` | 198 | `LoginWebViewInterceptController`；lib/test 均 0 导入；`android/` 中 `login_webview_intercept`/`fetchWithPolicy`/`intercept` **0 命中**（原生侧也已不存在）；对应审计 C16"已 descoped 的 native login intercept" |
| `lib/features/search/tag_search_repository.dart` | 27 | `TagSearchController`/`tagSearchControllerProvider`；仅 `test/illust_detail_controller_test.dart:16` 导入 |

### D.2 公共顶层声明：全仓库（lib+test）名字只出现 1 次（即声明处）—— **13 个**

| 位置 | 声明 |
|---|---|
| `lib/core/network/compat/dns_message.dart:40` | `DnsRecordType` |
| `lib/core/paging/paged_feed_controller.dart:91` | `PageFetcher` |
| `lib/core/profile/profile_edit_models.dart:10` | `ProfileFieldContract` |
| `lib/core/search/search_models.dart:6` | `SearchResultTypeWire` |
| `lib/core/settings/settings_controller.dart:161` | `imageSourceProvider`（对应审计 C13"单一图片源设置隐藏"后残留）|
| `lib/core/settings/settings_controller.dart:234` | `localBlockR18Provider` |
| `lib/core/settings/settings_controller.dart:242` | `localBlockAIProvider` |
| `lib/core/user/user_repository.dart:361` | `UserRestrictWire` |
| `lib/features/novel/novel_layout.dart:124` | `StableAnchor` |
| `lib/features/profile/profile_edit_page.dart:569` | `showProfileEditPage` |
| `lib/features/profile/profile_models.dart:6` | `ProfileTab` |
| `lib/features/profile/profile_models.dart:9` | `MeProfileTab` |
| `lib/features/profile/user_page.dart:105` | `showMePage` |

此外 **142 个**公共声明只在自己文件内使用、test 亦未引用（可私有化候选），例如 `NetworkFailure`（`network_contracts.dart:287`，本文件 28 次）、`NovelMarkupToken`（`novel_entity.dart:40`）、`DeepLinkRoute`（`intent_router.dart:4`）、`CommentStoreState`（`comment_store.dart:15`）、`UgoiraZipEntry`（`ugoira_zip.dart:23`）、`settings_page.dart` 中的 9 个页面类（`LanguageSettingsPage`、`TranslateSettingsPage`、`TranslationCredentialsPage`、`BrowseSettingsPage`、`DownloadSettingsPage`、`DownloadDestinationPage`、`HistorySettingsPage`、`BlockedTagsPage`、`DownloadTasksPage`）、`LoginWebViewInterceptController`（`login_intercept_controller.dart:26`）等。（脚本为词边界 grep，未区分同名不同实体，需逐项复核。）

### D.3 资源

| 资源 | 目录文件数 | 引用方式 | 结论 |
|---|---|---|---|
| `assets/emojis/` | 38 | `comment_assets.dart:3-43` 显式名单 38 项 → `commentEmojiAsset()` | 38/38 全部引用 |
| `assets/stamps/` | 40 | `comment_assets.dart:44-86` 显式 id 40 项 → `commentStampAsset()` | 40/40 全部引用 |
| `assets/icon.ttf` | 1（5 964 B）| `AppIcons` 14 个 codepoint | **6/14** glyph 被 UI 引用；14/14 被 `icon_font_test.dart` 断言 |

### D.4 依赖（`rg "package:<name>/"`）

| 依赖 | lib 文件 | test 文件 | 说明 |
|---|---|---|---|
| `go_router` | **0** | **0** | 未使用（路由全部命令式）|
| `cupertino_icons` | **0** | **0** | 未使用（`CupertinoIcons` 0 命中；Cupertino widget 来自 SDK）|
| `meta` | 11 | 0 | `@immutable`/`@visibleForTesting` 等 |
| `flutter_localizations` | 1 | 15 | 仅 Material delegates |
| `flutter_riverpod` | 79 | 29 | |
| `shared_preferences` | 6 | 3 | |
| `flutter_secure_storage` | 2 | 0 | |
| `cached_network_image` | 1 | 1 | `app/pixiv_image.dart` |
| `flutter_cache_manager` | 2 | 0 | |
| `webview_flutter` | 2 | 0 | |
| `crypto` | 7 | 3 | |
| `http` | 13 | 19 | |
| `http_parser` | 1 | 0 | `sauce_nao_provider.dart:6` |
| `path` | 5 | 1 | |
| `sqflite` | 2 | 1 | |
| `sqflite_common_ffi` | 1 | 1 | 非 Android/iOS 平台分支 + 测试 |
| `flutter_staggered_grid_view` | 8 | 1 | |
| `archive` | 1 | 1 | `ugoira_zip.dart:5`（仅 `Inflate, getCrc32`）|
| `image` | 1 | 1 | `ugoira_export.dart:6` |
| `path_provider` | 3 | 0 | |
| `visibility_detector` | 1 | 1 | `ugoira_viewer.dart:6` |
| `rhttp`（path） | 4 | 2 | |
| `easy_refresh` | 2 | 2 | `app/pull_to_refresh.dart`、`user_page.dart` |
| dev: `flutter_lints` | 0 | 0 | 仅 analysis_options |
| dev: `shared_preferences_platform_interface` | 0 | 21 | |
| dev: `webview_flutter_platform_interface` | 0 | 1 | |
| dev: `network_image_mock` | 0 | 8 | |

### D.5 测试目录中的杂项

- `test/` 为扁平 71 个 `*_test.dart` + `helpers/illust_fixtures.dart` + `goldens/home_bar.png`（已跟踪）+ `failures/`（4 个 golden 失败产物 png，**未跟踪**）。
- `test/zz_diag_tabbar_geometry_test.dart` **已跟踪**，但文件头注释写"One-off diagnostic (not committed)"，内含 `// ignore: avoid_print` + `print(`。

---

## E. 一致性

### E.1 语言

- 标识符：全部英文（`lib/` 非 i18n 文件中汉字仅出现在注释和 8 行字面量）。
- 注释：3 166 行 `//`/`///` 中 **43 行含中文 / 17 文件**，集中在 `network_probe.dart:547-568`（探测结论 doc）、`illust_caption.dart:3-10`（解析器说明）、`bookmark_models.dart:40,81`、`app_settings.dart:100,218`、`account_transfer_clipboard.dart:32`、`download_task.dart:64`、`rhttp_client_factory.dart:10`。其余为英文。
- 日志文案（`debugPrint`）：全部英文（含汉字 0）。
- 文档/PRD/spec：中文。

### E.2 文件与目录命名

- `lib/` 所有 Dart 文件均为 snake_case（正则校验 0 例外）。
- 分层放置不一致：`*_controller.dart` 10 个在 `lib/core/`（如 `settings_controller`、`search_feed_controller`、`profile_edit_controller`），6 个在 `lib/features/`（`recommended_feed_controller`、`illust_detail_controller`、`illust_download_controller`、`profile_feed_controller`、`user_detail_controller`、`login_intercept_controller`）；`*_repository.dart` 13 个在 `core/`，5 个在 `features/`（`recommended_repository`、`illust_detail_repository`、`related_illust_repository`、`ranking_repository`、`tag_search_repository`）。`lib/features/profile/profile_models.dart` 是唯一放在 features 的 models 文件。
- 单文件多页面：`settings_page.dart` 10 个页面类（C.6）。
- 导入风格：`lib/` 622 个相对导入、0 个 `package:pixiv_func/`；`test/` 420 个 package 导入、0 个相对导入 —— 各自一致。

### E.3 错误处理风格（并存）

| 风格 | 计数 / 例子 |
|---|---|
| 异常类 `implements Exception` | **49 个 / 36 文件**（`class *Exception` 40；无 `extends Exception/Error`）；`throw` 674 处 / 84 文件，`rethrow` 27 |
| `sealed` 结果/状态类型 | 14 个 sealed class，其中结果/状态 6 个：`ApiError`（`api_error.dart:5`，同时 `implements Exception`）、`RefreshOutcome`、`ProfileEditOutcome`、`ReverseImageSearchOutcome`、`AndroidIntentResult`、`IllustDetailState`/`UserDetailState` |
| `*Failure` 值对象 | 6 个（`NetworkFailure`、`ProfileEditFailure`、`ProfileEditSubmitFailure`、`ReverseImageFlowFailure`、`ReverseImageSearchFailure`、`CommentTranslationError`）+ 21 个 `enum *Kind/*Code/*Reason`（`NetworkFailureKind`、`DownloadFailureKind`、`AccountTransferErrorCode`……）|
| `*Result` 普通类 | 11 个（`OAuthResult`、`UpdateCheckResult`、`HistoryPageResult`、`TransferImportResult`……）|
| 可空返回 `Future<T?>` | 32 处 / 17 文件 |
| `Result<T>`/`Either` | 0 |
| `catch` 命名 | `catch (error` 199 / 209 处；`catch (_)` 5；无 `catch (e)` —— 命名一致 |
| Riverpod `AsyncValue` 消费 | `.when(` 26 处 / 17 文件；同时存在 `switch`/pattern matching（`case Xxx(` 30 处，`= switch (` 24 处）|

### E.4 异步风格

- `await` 713 处 / 104 文件；`.then(` 30 处 / 14 文件，主要用于串行尾链（`_writeTail.then` / `_flushTail.then` / `_persistenceTail.then`，`history_tracker.dart:127-130`、`history_repository.dart:250-258`、`settings_controller.dart:126-138`、`network_fast_route_store.dart:57-68`、`download_manager.dart:899-910`、`download_recovery.dart:428`）和 `cancelToken.whenCancel.then`（6 处）；`unawaited(` 62 处。风格基本统一。

### E.5 Widget 组织

- 私有 `Widget _build*()` 方法 **8 个 / 7 文件**；任意私有返回 `Widget` 的方法 **19 个 / 8 文件**（`reverse_image_search_page.dart` 8：`_body/_sauceNaoWebView/_idle/_privacyCard/_progress/_ready/_failure/_results`；`settings_page.dart` 3；`user_page.dart` 2；`illust_detail_page.dart` 2；其余各 1）；顶层 `Widget` 辅助函数 5 个（`settings_page.dart:61,1134`、`network_settings_page.dart:39,134`、`illust_detail_page.dart:68`）。主体仍以独立 widget 类为主（738 类）。
- Riverpod 声明风格：`Provider<T>` 62、`NotifierProvider.family` 12、`NotifierProvider<` 9、`AsyncNotifierProvider` 15（含 `.family` 1）、`FutureProvider` 4；`ref.read(` 177 vs `ref.watch(` 168；`ref.listen` 1；`ref.onDispose` 16。无 `StateProvider`/`StateNotifierProvider`/`ChangeNotifierProvider`（已是 Riverpod 3 风格），但 3 个 `ChangeNotifier` 控制器游离在 Riverpod 之外（C.6）。
- `WidgetsBinding.instance.addPostFrameCallback` 8 处 / 5 文件；`GlobalKey` 1；`mounted` 135 处 / 21 文件（`context.mounted` 15）。

### E.6 现代 Dart 特性采用度（正向观测）

`sealed`/`final`/`base`/`interface` 类修饰符 79 处；`super.key` 70 处（`Key? key` 旧写法 0）；enhanced enum 81 个；record 类型/typedef 约 245 处；switch 表达式 24 处、模式匹配 `case X(` 30 处。

---

## Related Specs

- `.trellis/spec/frontend/state-management.md`（1 259 行）—— 各核心契约（`PagedFeedController`、`SettingsRepository`、`NetworkAccessPolicy`、`ProfileEditController` 等）的权威描述；重构 C.6/C.7 涉及项时需对照。
- `.trellis/spec/frontend/quality-guidelines.md` —— "Forbidden/Required Patterns"（不得在框架状态机旁再建一套；guard 阻塞正确行为时删除而非再包一层）。
- `.trellis/spec/frontend/component-guidelines.md` —— 仅 Detail Transition / Tab Animation / Pull-to-Refresh 三个契约有内容；"Component Structure / Props / Styling / Common Mistakes" 仍为 `(To be filled by the team)` 占位。
- `.trellis/spec/frontend/type-safety.md`、`.trellis/spec/backend/error-handling.md`、`.trellis/spec/backend/logging-guidelines.md` —— **全部为模板占位**，没有项目约定；与 E.3（多种错误风格并存）、B.1（`debugPrint` 无 logger）相对应。
- `.trellis/tasks/09-01-func-1-0-hardening/research/audit-original-2026-09-01.md` —— C12（`UnimplementedError` 已消失，仅存注释 `paged_feed_controller.dart:116`）、C13（`imageSourceProvider` 现为零引用）、C14（`_runLegacyLadder` 已删除）、C16（`login_intercept_controller.dart` 仍在但零引用）、C20（硬编码中文已基本清除）的当前状态见上文。

---

## F. 现代化候选（按优先级）

| # | 候选 | 证据（文件:行、计数） | 目标形态 | 影响面 | 风险 | 目标 |
|---|---|---|---|---|---|---|
| 1 | 删除零引用文件 | `login_intercept_controller.dart`（198 行，lib/test/android 均 0 引用）、`compat_network.dart`（7）、`android_platform.dart`（16）；`tag_search_repository.dart`（27，仅 1 个测试引用）| 直接删除；`tag_search_repository` 视测试需要保留或并入 `search_feed_controller` | 4 文件、1 测试 | 低 | 简洁性 |
| 2 | 删除零引用公共声明 | D.2 的 13 项：`imageSourceProvider`/`localBlockR18Provider`/`localBlockAIProvider`（`settings_controller.dart:161,234,242`）、`showMePage`（`user_page.dart:105`）、`showProfileEditPage`（`profile_edit_page.dart:569`）、`StableAnchor`、`PageFetcher`、`ProfileTab`/`MeProfileTab` 等 | 删除；同步开启 `unreachable_from_main` 防回流 | 10 文件 | 低 | 简洁性 |
| 3 | 合并两套 Replica 路由构造器 | `ReplicaPageRoute`（`app/replica_page_route.dart`，28 处）vs `replicaRoute()`（`app/navigation/replica_route.dart`，3 处：`welcome_page.dart:67`、`language_page.dart:89`、`theme_page.dart:111`）；导航文件分布于 `lib/app/`、`lib/app/navigation/`、`lib/core/navigation/` | 保留一个（类版有 `settings`/reverse duration），onboarding 3 处改用；三处导航文件归到一个目录 | 4 文件 | 低 | 可维护性 / 简洁性 |
| 4 | 移除未使用依赖 | `go_router`、`cupertino_icons` 在 lib/test 均 0 导入（D.4）| 从 pubspec 删除，`flutter pub get` 后重跑 analyze/test | pubspec + lock | 低 | 升级 / 体积 |
| 5 | i18n key 校验与统一入口 | 27 个 `_xxxText` 包装函数（C.5）；`ReplicaStrings.text` 缺 key 时 `!` 抛异常（`replica_strings.dart:1906`）；`networkDohEndpointsInvalid` 未定义（`network_settings_page.dart:205`）；ru 缺 `detailQuality`；`i18n_network_keys_test.dart` 靠手写 key 列表 | 单一 `BuildContext` 扩展/顶层函数替代 27 个包装；key 由常量/enum 或生成代码承载，测试自动比对四语言 key 集合与代码引用 | 29 个引用 `ReplicaStrings` 的文件 | 中（需覆盖所有调用点）| 可维护性 / 可迭代性 |
| 6 | i18n 存储形态 | `replica_strings.dart` 1 924 行 / 113 KB 单文件 Map 字典，434 key × 4 语言；63 行超 80 列 | 候选：拆为每语言一文件的 const Map，或迁 ARB + `flutter gen-l10n`（SDK 内置，无第三方依赖）| 1 文件 + 5 的调用点 | 中（迁 ARB 会改变所有查表写法；纯拆文件风险低）| 可维护性 / 升级 |
| 7 | 拆分 `settings_page.dart` | 2 088 行、27 类、10 个页面类、`setState` 25、`late` 7、`showSnackBar` 9、>100 行 build 3 个（`:993,1178,1388`）| 每页面一文件（`features/settings/pages/…`），共享 tile/label widget 单独文件 | 1 文件 → ~10 文件；导入方 `home_page`、`user_page` 等 | 低–中（纯移动，需保留公开类名或更新导入）| 可维护性（LLM 编辑面）|
| 8 | 统一 SnackBar 反馈 | 44 处 / 17 文件手写 `ScaffoldMessenger.of/maybeOf(context).showSnackBar(SnackBar(content: Text(...)))`；`.of` vs `.maybeOf` 混用 | 一个 `showReplicaSnackBar(context, text)`（或 `BuildContext` 扩展）；统一 `maybeOf` 语义 | 17 文件 | 低 | 简洁性 / 一致性 |
| 9 | JSON 读取 helper 去重 / 容器类型统一 | `_map` ×5、`_firstString` ×3（B.2）；`Map<String, dynamic>` 105 vs `Map<String, Object?>` 24，5 文件并用；`json['…']` 158 处、`as X?` 143 处 | `lib/core/entity/json_read.dart` 一组纯函数（`readMap/readString/readInt/...`），统一 `Map<String, Object?>`（或统一 `dynamic`）；不引入 codegen 也可完成 | 约 15 个解析文件 | 中（解析路径需测试保护：`illust_entity_pages_test`、`novel_markup_hardening_test` 等已存在）| 可维护性 / 一致性 |
| 10 | `ChangeNotifier` 控制器并入 Riverpod | `ProfileEditController`（`profile_edit_controller.dart:65`，`_setState` 14）、`ReverseImageSearchController`（`reverse_image_controller.dart:62`，`_setState` 11）、`NovelReaderController`（`novel_reader.dart:135`）；消费侧手动 `addListener`+`setState`/`AnimatedBuilder`（`profile_edit_page.dart:94,168,236`、`reverse_image_search_page.dart:45-50`、`novel_reader.dart:233-234`）| `Notifier`/`AsyncNotifier` + `ref.watch`，与其余 89 个 provider 同一范式 | 3 控制器 + 3 页面 + 对应测试（`profile_edit_test`、`reverse_image_search_test`、`novel_reader_test`）| 中（生命周期/取消语义需保持，见 state-management.md "Profile Edit Contract"）| 一致性 / 可维护性 |
| 11 | `history_page` 改用 `PagedFeedController` | `history_page.dart:113,124,142-145` 手写 `ScrollController` + `_loadMore/_hasMore/_generation`，`setState` 7；其余 feed 全部经 `lib/core/paging/`（18 个导入方）| 复用 `PagedFeedController` 及共享 tail/error widget | 1 文件（608 行）+ `history_persistence_test` | 中（history 有 account 切换与 outbox 语义，见 spec "Browsing History Contract"）| 一致性 / 简洁性 |
| 12 | 超长方法拆分 | >150 行：`update_service.dart:194 _checkOnce`（228）、`download_manager.dart:528 _run`（187）、`login_page.dart:149 build`（186）、`recommended_illust_page.dart:243 IllustCard.build`（180）、`profile_edit_controller.dart:223 submit`（167）；>100 行 build 另 9 个（C.6）| 按阶段/子 widget 拆为私有类或小函数，行为不变 | 5–14 处 | 低–中（`_run`/`_checkOnce` 有恢复/签名语义，需保留测试）| 可维护性（LLM 编辑面）|
| 13 | 图标字体裁剪与图标系统收敛 | `AppIcons` 14 个 glyph 仅 6 个被引用（8 处 / 3 文件）；`icon_font_test.dart:47-70` 断言 14 个；Material Icons 194 处 / 84 种；Cupertino 4 处 | 二选一：(a) 删除 8 个未用 `IconData` 常量并放宽测试；(b) 子集化 `icon.ttf`（需 fontTools 重生成并更新 `beta56-icon-font.md` 记录）| 1–2 文件 + 资源 | 低（a）/ 中（b，触及资源来源记录）| 简洁性 / 体积 |
| 14 | 偏好存储注入统一 | `SharedPreferencesAsync()` 在 7 个 store 各自默认构造（C.7）| 一个 `sharedPreferencesProvider` 注入；测试用 override | 7 文件 | 低 | 一致性 / 可测试性 |
| 15 | 日志出口统一 | `debugPrint` 23 处 / 5 文件（4 个在 `core/widget/`）；`logging-guidelines.md` 为空模板 | 极小 `log(tag, message)` 函数（或保留 `debugPrint` 但集中到 `core/widget/widget_log.dart`）；补 spec | 5 文件 | 低 | 一致性 |
| 16 | 格式化纳入 CI | 46 文件 `dart format` 漂移；`ci.yml:18-19` 无 format 步骤 | 一次性 `dart format lib test`；CI 增加 `dart format --output=none --set-exit-if-changed lib test` | 46 文件（纯格式）| 低（与其他 09-01 未提交改动可能冲突，需在其提交后执行）| 一致性 / 可迭代性 |
| 17 | 硬编码颜色收敛 | `Colors.*` 37 处 / 15 文件、`Color(0x…)` 16 处（C.3）；`FuncTokens` 与 `ColorScheme.fromSeed` 双来源 | 把重复的 `Color(0x99343838)`（`ugoira_viewer.dart:149`、`recommended_illust_page.dart:334,350`）、`0x33343838` 等提升到 `FuncTokens`/`ThemeExtension`；`Colors.white/black` 视 replica 视觉决定保留 | ~15 文件 | 低 | 一致性 |
| 18 | 可私有化的公共声明 | 142 个只在本文件使用的公共声明（D.2），如 `settings_page.dart` 的 9 个页面类、`NetworkFailure`、`NovelMarkupToken` | 加 `_` 前缀或拆文件后按需公开；配合 `unreachable_from_main` | 分散 | 低（大量小改动）| 简洁性 |
| 19 | 测试目录整理 | 扁平 71 个测试；`zz_diag_tabbar_geometry_test.dart` 标注"not committed"却已跟踪；`test/failures/` 4 个未跟踪 png | 删除诊断测试；`.gitignore` 加 `test/failures/`；按 `core/`/`features/` 分目录（可选）| test/ | 低 | 可维护性 |
| 20 | Material 3 迁移 | `useMaterial3: false`（`replica_theme.dart:19`）；M2 组件主题已用 3.47 的 `*ThemeData` 名；无 `MaterialStateProperty` 等 M2 遗留 API | **不建议在本 task 内迁移**：README/PRD 冻结 beta56 视觉；M3 会改变 AppBar/TabBar/BottomNavigationBar 外观与 golden（`test/goldens/home_bar.png`）| 全部 UI | 高 | 升级（记录为后续独立决策）|

### 建议启用的更严格 analyzer/lint 选项（附本次测得的估算值）

| 选项 | 估算当前违规 | 说明 |
|---|---|---|
| `analyzer.language.strict-raw-types: true` | 12 处（`is/as Map|List` 无类型参数）| 集中在 JSON 解析文件，可与候选 9 一起修 |
| `analyzer.language.strict-casts: true` | 上界 ≈ 158 个 `json['…']` 读取中未显式收窄的部分；显式 `as` 已有 143 处 | 精确数需开启后看 analyzer 输出 |
| `analyzer.language.strict-inference: true` | 未测量 | 与 `dynamic` 126 处相关，需开启后测 |
| `unreachable_from_main` | ≥ 13 个声明 + 4 个文件 | 直接对应 D.1/D.2 |
| `prefer_final_locals` | ≤ 92 处 / 31 文件 | 有 quick-fix，可 `dart fix --apply` |
| `avoid_dynamic_calls` | ≤ 7 处 | 几乎已满足 |
| `unawaited_futures` | 未测量（项目已用 `unawaited` 62 处）| 建议开启后看输出 |
| `lines_longer_than_80_chars` | 133 行 / 34 文件（63 在 i18n 字典）| 与 `dart format` 一起处理 |
| `sort_pub_dependencies` | 1 | pubspec |
| `public_member_api_docs` | 275 个公共顶层类型无 `///` | 若嫌重，可只对 `lib/core/` 开启 |
| `avoid_positional_boolean_parameters` | 已有 1 处 ignore（`network_contracts.dart:259`）| 其余未测量 |
| `prefer_single_quotes` | 0 | 免费开启，锁定现状 |
| `always_declare_return_types`、`type_annotate_public_apis`、`directives_ordering`、`cascade_invocations`、`use_if_null_to_convert_nulls_to_bools` | 未测量 | 常见于严格集（如 `very_good_analysis`），需开启后测 |

---

## Caveats / Not Found

- **工作树含其他 task 的进行中改动**：15 个未跟踪 `lib/` 文件（如 `related_illusts_section.dart`、`sauce_nao_provider.dart`、`web_profile_repository.dart`、`naming_rule.dart`、`saf_tree.dart` 等）、2 个未跟踪测试、30+ 个已修改文件（含 `replica_strings.dart`、`network_settings_page.dart`）。以上计数与 `git HEAD` 会有差异；本文件不应被当作 HEAD 的基线。
- 脚本均为启发式：方法长度按花括号配对（忽略字符串/注释内的括号，可能误差 ±数行）；"零引用"按词边界 grep，同名不同实体或字符串/反射式引用会被漏判；`Widget _build` 统计只覆盖返回类型写为 `Widget`/`Widget?` 的私有方法。
- `strict-*` 与 `unawaited_futures` 的精确违规数无法用 grep 得到；表中为上界或"未测量"，需在 analysis_options 中开启后由 analyzer 给出。
- 未找到原版源码副本，因此本文件**不**对原版代码本身做任何断言；"移植痕迹"仅指本仓库内的命名、注释、结构和 Trellis 记录。
- `flutter analyze` 0 issues 说明 flutter_lints 默认集全部通过；上文所有"缺口"都在默认规则集之外。
- `login_intercept_controller.dart` 是否可删由 `09-01-settings-productization`（D3 "删除 native login intercept 生产接线"）最终决定；本文件只记录其当前零引用状态。
- 未验证运行时行为（未运行 `flutter test` / 构建）；`networkDohEndpointsInvalid` 未定义 key 的后果是基于 `ReplicaStrings.text` 的源码推断（`_values[zhCN]![key]!`）。
