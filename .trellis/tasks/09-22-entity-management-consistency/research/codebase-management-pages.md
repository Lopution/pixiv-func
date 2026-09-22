# W6 管理页现状（codebase）

基线 `main@8067b2d`。全部行号已核对。共同结构：五页都是 `Scaffold + AppBar + (PullToRefresh +) 列表`；**无任何页有限宽**（`AppBreakpoints` 仅被 shell/详情消费，management 角色限宽不存在——见 `app/layout/app_breakpoints.dart:5-28`，整库 `ConstrainedBox` 限宽仅 `welcome_page.dart:26-28` 的 520）。

## 1. `features/history/history_page.dart`（507 行）

- AppBar：title=`historySettings`；actions=`delete_forever` 图标 → `_deleteAll`（40-49, 60-81）。
- `_deleteAll`/`_delete` 共用 `_confirmDelete`：`showAppBottomSheet` + `FractionallySizedBox(0.35)` + OutlinedButton 取消 / FilledButton 确认（460-500）。
- 列表：`PullToRefresh` + `SmoothWheelScroll` + `CustomScrollView`（`PageStorageKey('history-<accountId>')`、`restorationId` 同名，166-178）+ `IllustFeedGrid(padding 10, spacing 10)`（179-188）。
- 条目：`_HistoryEntry` 外包 `GestureDetector(onLongPress: _delete)`（239-243）——**长按直接进删除确认，无选择模式**。
- 插画已知：`_KnownIllustEntry`（271-322）`ClipRRect r10` + `MediaQuery.sizeOf(context).width - 30) / 2` 固定两列估高（278-283）→ 与 `IllustFeedGrid` 实际列宽（`FeedItemExtent`）脱钩，宽屏下估算错误。
- 小说条目：`_NovelHistoryEntry`（324-371）封面 `AspectRatio(1)` 正方形（`_SnapshotCover` 437-458）——与 NovelCard 68×88 / NovelRow 56×72 三种封面比例并存。
- 每条底部 `_HistoryCardFrame` 追加 `_formatHistoryDate`（labelSmall，`yyyy-MM-dd HH:mm`，374-399, 502-507）——这就是"日期变体"的现存形态；无类型区分图标/日期分组头（"按日期/类型帮助找回"未实现）。
- 状态：loading/error(FeedError)/empty(ReplicaEmptyState)/loadMore 尾部自绘（189-207，非 FeedTail）。
- store：`historyFeedControllerProvider(accountId)`，`removeRecord` 硬删除无撤销（history_feed_controller.dart:61-72）。

## 2. `features/watchlater/watchlater_page.dart`（68 行）

- AppBar 仅 title（23）。无页级管理动作。
- `PullToRefresh(onRefresh: ref.refresh(watchLaterStoreProvider.future))`（24-26）。
- `IllustFeedGrid` + `IllustCard(heroScope: 'watchlater')`，`itemIds` 已接线 → 详情分页器可用（47-59）；`restorationId: 'watchlater'`（45）。
- 移除路径：卡片长按 → `showCardActionSheet` → `_WatchLaterAction`（illust_card_actions.dart:103-148）：已存时 label=`cardActionRemoveWatchLater`，`store.remove(entity.id)` **无 SnackBar、无撤销**；store 有 `add/remove/clear`（watch_later_store.dart:36-57），entry 持有完整 `IllustEntity`，撤销=重加可行。
- Astra 条目："稍后再看缺页面级管理"确认；§4.6 要求"提供移除和撤销，不扩成另一套收藏"。

## 3. `features/watchlist/watchlist_page.dart`（238 行）

- `DefaultTabController(2)`：manga / novel 两个 `_WatchlistFeedBody`（24-44）。
- 列表：`PullToRefresh` + `NotificationListener`（`maxScrollExtent-400` 触发 loadMore，76-84）+ `ListView.builder` + `_LoadMoreFooter`（101-135）。无 restorationId/PageStorageKey。
- `_WatchlistEntryTile`（147-238）：
  - `ListTile`，leading `ClipRRect r6` 48×48 封面（166-176）；
  - title 行内嵌"新内容"徽标：`latest_content_id > seen` 时显示 `errorContainer` `r4` 的 `watchlistNewContent`（186-202）——**更新变体的现存形态**；
  - subtitle：`userName · yyyy-MM-dd · {count} works`（205-215）；
  - `enabled: canOpen`（novel 且 latest==null 时禁用，163, 216）；
  - `onTap` = `_open`：**先 `markSeen(latest)` 再导航**——manga → `openIllustSeries(series.id)`（目录页），novel → `openNovel(latest)`（直接进最新一话）（221-237）。
- 缺口（对照 §4.6"区分查看更新、打开目录和继续阅读"）：
  - 单一 onTap 承载两种语义且 manga/novel 不一致（manga 进目录、novel 进正文）；
  - 小说系列无目录页（路由表只有 `series/:seriesId` → `IllustSeriesPage`，routes.dart:476-481；`_NovelSeriesBar` 在 NovelPage 内是 prev/next 条，novel_page.dart:770-845）；
  - 无"取消追更"入口（`WatchlistToggle` 存在于此页之外：`novel_page.dart:832`、`illust_series_page.dart:220`）；
  - 游标即已读：`watchlistReadCursorProvider.markSeen`（watchlist_store.dart:252-279）——`latest_content_id` 是**更新游标**，不是阅读进度（§6 明确不得冒充）。

## 4. `features/localnovel/local_novels_page.dart`（133 行）

- AppBar：title + `file_open_outlined` 导入（22-31）。
- `PullToRefresh` + `ListView.builder`（49-56）；无 restorationId。
- `_LocalNovelTile`（84-109）：`ListTile`，leading 纯 icon、title 1 行、subtitle `字数 · 导入日期`，onTap `openLocalNovelReader(novel.id)`（102），**trailing 是 delete `IconButton`**（103-107）——Astra"本地书库尾部主按钮仍是删除"确认。
- 删除确认：`showDialog`（**裸 `showDialog`，非 `showAppDialog`**）`AlertDialog` + TextButton 取消 / FilledButton 删除（111-132）→ `store.delete(novel)`，无 SnackBar/撤销。
- `LocalNovel.readOffset` 字段存在（local_novel_repository.dart:25,39），阅读器写入"character-offset cursor"（local_novel_reader_page.dart:28 注释）；W1 负责换算 `initialAnchor`。W6 可用 `readOffset != null` 表达"继续阅读"态（进度百分比 = offset/charCount 可选）。

## 5. `features/settings/pages/download_tasks_page.dart`（257 行）

- AppBar 仅 title=`downloaderSettings`（48）；body `ListView(padding 12)`：**hint 文本 + 各 group section + 各 task tile 平铺同级**（51-63）——"组与子任务无父子层级"确认。
- `_DownloadGroupSection`（70-172）：`Card`+`ListTile`，subtitle = 状态文本 + `LinearProgressIndicator(group.progress)` + `done/count`；trailing：
  - `queued|running` → pause + close（取消）（92-94, 115-130）
  - `retryable|failed|canceled` → play_arrow（`resumeDownload` → `resumeGroup`）+ close（95-98, 131-146）
  - 其余 → 静态 icon `check_circle_outline`/`info_outline`（147-151）
  - 组状态文案（156-171）：`retryable` 且全部 retryable 子项 `failureKind==paused` → `downloadPaused`，否则 `downloadFailed`；`finalizing→Running`、`orphaned→Failed`。
- `_DownloadTaskTile`（174-257）：`Card`+`ListTile`，title=`displayName`，subtitle=状态+进度条+error 文本（paused 时不显示 error，191-193）；trailing switch（196-236）：
  - `queued|running|canceling|retryable` → 第一键 `retryable?Icons.refresh+retryDownload→manager.retry : Icons.pause+pauseDownload→manager.pause` + `Icons.close+cancelDownload→manager.cancel`（canceling 时第一键省略，205）
  - `failed|canceled` → 单 refresh（`retryDownload` → `manager.retry`）（226-230）
  - `succeeded`/`finalizing`/`orphaned` → 静态 icon（231-235）
- 缺口（对照 Astra + §4.6）：
  - **暂停的任务显示"重试"**（retryable+paused → refresh/retryDownload → `manager.retry`），与 §5.7"'重试'是重试同一操作"冲突；应区分"继续/恢复"（resume paused）与"重试"（retry failed）。
  - 完成后无任何"查看"动作（succeeded 只有静态 ✓ icon）；`task.illustId`/`pageIndex` 就在 snapshot 里可 `openIllust`。
  - 无终态条目的移除/清空 API（`DownloadManager` 无 dismiss/removeTask；terminal 任务永久驻留 `_jobs`，download_manager.dart:108-109）。
  - 提交反馈 `downloadQueuedMessage` SnackBar 无 action（illust_card_actions.dart:90、page_image.dart:241、user_page.dart:268），不能直达任务页；任务页路由 `/settings/tasks`（routes.dart:730-734）。
  - 页面通过 `_manager.changes.listen → setState` 刷新（27-29），非 provider。
- 测试：`test/download_tasks_page_test.dart`（254 行）固化了现有行为——包括"paused tile 显示 refresh 图标并 retry 至成功"（203-254），W6 改动作映射时需同步改测试（quality-guidelines: 删除 guard 同步删固化测试）。

## 6. 排名页接入点

| 页 | 现状 |
|---|---|
| `ranking_page.dart` | `TabBar` 11 模式在 AppBar title（104-115）；action `menu_book` → `openNovelRanking`（98-102）；`RootSwipeSwitcher + TabSlideStack`（117-139）；`IllustFeedGrid` + `IllustCard(heroScope:'ranking:<mode>')`（218-231）——**无排名数字契约** |
| `novel_ranking_page.dart` | `TabBar` 9 模式（77-88）；body 直接按 `_selectedIndex` 换 key 替换（无滑动栈，90-96，W2 收敛）；`SliverList` + `StaggeredEntrance` + `NovelRow`（174-185）——**无排名数字** |

排名变体是 W6 契约的命名 variant；排行页**接入**归 W6，榜单结构（TabBar/切换）归 W2。

## 7. 管理模式/批量选择现状

- 管理页（history/watchlater/watchlist/localnovels/download）**均无 selection mode**。
- 现存唯一"模式切换"先例在 `illust_detail_page.dart`（W4 范围）：`_downloadMode`/`_blockMode` 布尔（58-59, 90），长按图片/标签切换，AppBar 不换色、无计数。
- 无 `Dismissible`、无 Checkbox 选择、无 "全选"。
- `HistoryPage` 长按=删除、`WatchLater` 长按=动作 sheet、`LocalNovels` 无长按——长按语义三页三种。

## 8. 限宽消费点

- `AppBreakpoints`（app_breakpoints.dart:5-28）：`compact=600`、`medium=600`（rail 起）、`expanded=1200`（双栏/extended rail 起）、`useNavigationRail/useExtendedRail/useTwoPaneDetail`。
- `TwoPane`（two_pane.dart:9-40）：固定比例双栏，≥1200 详情用。
- §5.5：management 角色"保持信息与动作聚合"的限宽由各叶子按文件所有权接入；新增跨 feature shell/helper 需 ≥3 个真实消费者。W6 名下管理页=5 个（history/watchlater/watchlist/localnovels/download_tasks），满足"≥3 消费者"门槛，但 owner 形态（`AppBreakpoints` 扩方法 vs `app/layout/` 新 widget）需在设计中固定。
- 现实表现：宽屏下 history/watchlater 由 `illustColumnsFor`（min 180，feed_grid.dart:19,273-276）自然加列，无限宽必要性的反而是 ListView 类页面（watchlist/localnovels/download_tasks）——列表项动作被推到屏幕远端。
