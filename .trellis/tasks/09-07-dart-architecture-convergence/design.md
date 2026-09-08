# 技术设计：Dart 架构收敛（child C）

本文件从 parent `design.md` §2–§3、§13–§14 派生，叠加 `research/architecture-recount.md`（HEAD `1186ff2`）
的重算结果。parent 未变部分不重复，只列 C 特化细节与差异矩阵。行为、协议、持久化格式不变；
交互/视觉变化属于 child F。

## 1. 范围与边界

- 目标：`lib/` 分层规则测试化、单一 owner、零引用删除、现代化（i18n/状态范式/分页/文件规模）、
  组件层 enabler、静态性能约束。
- 不做（parent §15 / PRD R6）：M3、material_ui、go_router、json 代码生成、Result、错误分类重构、
  路由阶梯/ECH/SNI 改动、持久化 key/格式改动。
- 已确认变化（recount 意外发现）：`login_intercept_controller.dart` 与原生 `login_webview_intercept`
  已从 HEAD 消失 → C1/C5 不再涉及；`imageSourceProvider` 已删（once 声明 13→12）；测试基线 **686**。

## 2. 分层规则（R1，写入 `.trellis/spec/frontend/directory-structure.md`）

规则文本（layering_test 与 spec 一致）：

```text
lib/app/            应用壳：MaterialApp、主题、共享 UI 基件、路由门面     可 import: core, app
lib/features/<f>/   仅 widget/页面/页面级 controller 的 UI 胶水          可 import: core, app；
                    features 之间只能 import lib/app/navigation/routes.dart 门面
lib/core/<domain>/  entity / repository / controller(Notifier) / store /
                    platform 适配器；不得 import features 或 app
```

- 唯一被批准环：`lib/app/navigation/routes.dart`（门面 import 各 feature 页面入口，页面 import 门面），
  layering_test 白名单化。
- 附加名称检查：`features/` 不得定义 `_*Tail/_*Error/_*Empty/_*Card/_*Status`（组件职责归 `lib/app/widgets/`）。
- 白名单初值（HEAD 1186ff2，recount §B）：
  - `core→features`：1 条（`widget_feed_loader.dart:21`）
  - `features→features`（a≠b）：44 条（recount 表格，抄 HEAD 行号）
  - `app↔onboarding`：app→onboarding 1 条（`app.dart:14`）+ onboarding→app 16 条（`replica_route.dart`、
    `func_tokens.dart`、`replica_button.dart`、`replica_scaffold.dart`、`replica_switch_tile.dart`、
    `settings_load_error.dart`）
  - 每消除一项即删白名单条目；C 结束时白名单为空。

## 3. 路由与导航（R2，C2）

- 合并三处为 `lib/app/navigation/`：
  - `replica_page_route.dart`（`ReplicaPageRoute`，唯一转场原语；删 `replicaRoute()` 3 处调用）
  - `route_observer.dart`（从 `lib/core/navigation/` 迁入）
  - `routes.dart`（门面，形参为 id/参数；F 以 go_router 重实现，调用点不变）
- `lib/core/navigation/home_shell_metrics.dart`：两个可写 static（`bottomNavTop`/`bottomNavHeight`）
  改 `InheritedWidget`/provider；`home_page.dart:196-197` 写入、`illust_detail_page.dart:301,306` 读取。
- 门面 API 初稿：

```dart
// lib/app/navigation/routes.dart
void openIllust(BuildContext context, int illustId);
void openUser(BuildContext context, int userId);
void openNovel(BuildContext context, int novelId);
void openSearch(BuildContext context, String query);
void openRanking(BuildContext context);
void openSettings(BuildContext context);
void openHistory(BuildContext context);
void openReverseImageSearch(BuildContext context);
void openComments(BuildContext context, {required int illustId});
void openProfileEdit(BuildContext context, {required int userId});
// … 以「调用点现有 push 目标」为准补全，不以 widget 实例为形参
```

## 4. 组件层（R7，C3/C3b）

### 4.1 Feed 共享组件

- `lib/app/widgets/feed/illust_card.dart`：`IllustCard` 从 `recommended_illust_page.dart:236` 移出
  （6 模块 import）。
- `lib/app/widgets/feed/feed_grid.dart`：`IllustFeedGrid`（sliver）+ `illustColumnsFor(double crossAxisExtent)`；
  8 处 `SliverMasonryGrid.count` 统一，手机宽度恒 2 列（现状不变）。
- `lib/app/widgets/feed/feed_states.dart`：`FeedTail`/`FeedEmpty`/`FeedError` 消费 `PagedFeedState`。
  差异矩阵（32 项，recount §D）归并为三族，差异以参数表达：

| 参数 | 取值（来源） |
|---|---|
| `tail` spinner / 完结 / 重试 | `_FeedTail`×4、`_NewFeedTail`、`_ProfileFeedTail`、`_RankingFeedTail`、`_SearchFeedTail`、`_CommentFeedTail`、`_LoadMoreFooter` |
| `empty` icon+title / 刷新按钮 | `_NewEmpty`、`_SearchEmpty`、`_RecommendedEmptyFeed`（已用 `ReplicaEmptyState`）、`_ProfileEmpty` |
| `error` 可滚动/行内/overlay | `_*Error`×12、`_SearchInlineError`、`_ErrorOverlay`（ugoira 保持 overlay 样式）、`_InitializationFailure` |
| `status` icon+title+detail | `_NewStatus`、`_NovelStatus`、`_ProfileStatusPage`、`_PagePlaceholder`、`_SearchStatus` |
| 特殊 | `_StartupError`（无 onRetry，局部文案）、`_TailError`/`_NewRefreshError`（横条）、`_RankingInitialError`（带 mode） |

- 保留不动：`_ErrorView`（详情页错误，独立契约）、`_InitialErrorView`（recommended 首屏）。
- 无障碍基线（R7）：共享组件带 `Semantics`/tooltip、≥48dp、可聚焦；14 处 `GestureDetector` 逐个评估。

### 4.2 PixivImage 变体（C3b）

- `PixivImageSize.feed/detail/viewer/avatar` + 命名构造器（含 `.hero(tag:)`）；13 个调用点切换
  （recount §D 列表）；`PersonAvatar` 复用 avatar 变体。
- `memCacheWidth`：feed/avatar = 布局宽 × dpr（上限 1.5× 逻辑像素）；detail = 屏宽；viewer 不限。
- 测试：widget 测试断言 feed/avatar 变体的 `ImageProvider` 带 `memCacheWidth`。
- Hero 转场契约（`component-guidelines.md`）不变；`_GlobalRectClip` → `motion/hero_rect_clip.dart` 公开为
  `HeroRectClip`（C8 时做，`hero_transition_test` 改 `find.byType`）。

### 4.3 Motion

- `lib/app/motion/motion_tokens.dart`：全部时长/曲线单一来源（16 处 `Duration(milliseconds:`、3 处 `Curves.`）。

## 5. 单一 owner（R3，C5）

按 parent §2.2 契约表逐项落地；recount 确认：主机再声明少 login intercept 一处（已删），
其余不变。HTTP 客户端三套栈 provider 化，理由写 spec（`backend/directory-structure.md`）。

## 6. i18n 迁移（R4，C6，方案 (a) gen-l10n）

- `l10n.yaml`：`arb-dir: lib/l10n`，`template-arb-file: app_zh.arb`，`output-class: AppStrings`，
  `nullable-getter: false`。
- 迁移脚本 `tool/migrate_i18n.py`：从 `ReplicaStrings._values` 生成四语言 ARB；`{0}` 参数 → ARB 占位符；
  ru 补 `detailQuality`（沿用 zh 文案 + TODO-翻译）；修 `networkDohEndpointsInvalid`（当前未定义仍被调用，
  `network_settings_page.dart:205`）。
- 调用点：`AppStrings.of(context).<key>`；删 27 个 `_xxxText` 包装与 `ReplicaStrings`；
  `ReplicaLanguage` 保留为设置枚举映射 `Locale`。
- 校验：`gen-l10n` 保证四 ARB key 集合一致；删除 `i18n_network_keys_test.dart` 手写 key 列表。
- 键数：432（zh/en/ja），ru 431→补齐 432。

## 7. 状态范式（R4，C7）

- 3 个 `ChangeNotifier`（`ProfileEditController`、`ReverseImageSearchController`、`NovelReaderController`）
  → Riverpod `Notifier`/`AsyncNotifier` + `.autoDispose(.family)`；页面 `ref.watch`；
  平台适配器（`MethodChannelReverseImageInputPlatform` 等 3 页 4 处）经 provider 注入。
- `history_page.dart:113-145` 手写分页 → `HistoryFeedController extends PagedFeedController`；
  保持 account 切换与 outbox 语义（`frontend/state-management.md` 契约）。
- `startup_gate.dart` 上移 `lib/app/`；`HomeShellMetrics` 静态量 provider 化。

## 8. 删除与拆分（R5，C1/C8）

- 零引用：`compat_network.dart`、`android_platform.dart`（2 个 barrel，0 导入）；`tag_search_repository.dart`
  视测试处理（仅 `illust_detail_controller_test.dart:16` import）；12 个 once 声明（recount §F2 表）；
  8 个未用 `AppIcons`（`addFollow/filter/me/toggle/pawoo/twitter/web/blocked`，放宽 `icon_font_test`）；
  141 个仅本文件公共声明按需私有化（随 C8 顺带）。
- 拆分（每文件 ≤600 行）：`settings_page.dart`（14 页面类 → `features/settings/pages/<name>_page.dart`，
  `MePage` 去重改名 `AccountCard`）；`illust_detail_page.dart`（1,388 行）；`user_page.dart`（959 行）；
  `network_policy.dart`（1,198 行 → 策略/`PixivNetworkFactory`/`image_cache.dart` 三份，受 network-perf-ab
  契约约束，等其归档——已满足）；5 个 >150 行方法（recount 提示 HEAD 实有 8 个，按 PRD 5 个为主，
  其余作为候选）。
- 拆分后 `hero_transition_test` 类名字符串断言改 `find.byType`。

## 9. 性能约束（R8，C3b/C7b）

- feed/avatar 变体 `memCacheWidth` + 测试断言（见 4.2）。
- 共享组件只 `watch` 所需切片（`select`）；`IllustCard` 不 `watch` 整个 feed state（C3c）。
- 首帧等待项固定 settings + 账号；`startup_gate_test` 断言。
- `DownloadRecoveryStore` 设记录上限（如 200，淘汰已完成项）或改增量写入（code review 决定）；
  history 三条查询 `EXPLAIN QUERY PLAN` 单测断言无 `SCAN TABLE`。
- 不做数值化性能结论（D-9）。

## 10. 停止条件

- 任何拆分改变 `component-guidelines.md` 的 Detail Transition / Tab / Pull-to-Refresh 三个契约 →
  回滚该提交。
- `PagedFeedController` 三相语义（loading/loaded/error + tail）变化 → 回滚。
- 每个删除旧路径（旧 repository 位置、`ReplicaStrings`、`replicaRoute()`）为显式回滚点，先有新路径测试。
