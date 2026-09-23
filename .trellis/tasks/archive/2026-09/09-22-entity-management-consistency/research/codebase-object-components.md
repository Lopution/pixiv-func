# W6 对象组件现状清单（codebase）

基线：`main@8067b2d`（PR #54 合入后），当前分支 `docs/09-22-ui-leaf-planning`。行号已对照 HEAD 核对。

## 1. 小说条目：两份平行实现

### `lib/app/widgets/novel_card.dart` — `NovelCard`（81 行）

| 维度 | 现状 | 行号 |
|---|---|---|
| 容器 | `Card`，margin `h12/v6` | 20-21 |
| 按压反馈 | 无 `PressScale`，仅 `InkWell` 水波 | 22 |
| 圆角 | InkWell `borderRadius: 4`；封面 `ClipRRect r6` | 24, 68 |
| 封面 | `68×88`（3:4 近似），`PixivImage.feed(layoutWidth: 68)`；null 时 `ColoredBox + menu_book_outlined` | 67-78 |
| 标题 | `maxLines: 2`，`fontWeight.w600` | 36-41 |
| 作者 | `FuncSemanticTokens.caption` | 43-48 |
| 元信息 | **无字数、无 chevron、无收藏、无排名** | — |
| 语义 | 无 `Semantics` 包装 | — |
| 长按 | 无 | — |
| key | `ValueKey('novel-${entity.id}')` | 13-14 |

调用点（3 处）：`search_result_page.dart:247`、`new_page.dart:382`、`profile_novel_feed.dart:96`。

### `lib/app/widgets/novel_row.dart` — `NovelRow`（96 行）

| 维度 | 现状 | 行号 |
|---|---|---|
| 容器 | `PressScale` 包 `Card`，margin `h12/v6` | 18-20 |
| 按压反馈 | `PressScale` + `InkWell` | 18, 21 |
| 圆角 | InkWell `borderRadius: 4`；封面 `ClipRRect r4`；**placeholder 分支无 ClipRRect（直角）** | 23, 75-83 |
| 封面 | `56×72`（7:9），`PixivImage.feed(layoutWidth: 56, fit: cover, placeholderColor: surfaceContainer)` | 82-93 |
| 标题 | `maxLines: 2`，`fontWeight.w600` | 35-40 |
| 作者 | `Theme.bodySmall`（与 Card 版的 `tokens.caption` 不同来源） | 42-47 |
| 字数 | `'${entity.textLength} ${context.l10n.novelWords}'`，bodySmall | 48-52 |
| 尾标 | `Icon(Icons.chevron_right)` | 56 |
| 语义/长按/收藏 | 均无 | — |
| key | 无默认 id key（`super.key`） | 12 |

调用点（2 处）：`recommended_home_page.dart:362`（StaggeredEntrance 包裹）、`novel_ranking_page.dart:183`（StaggeredEntrance 包裹）。

### 差异摘要

| | NovelCard | NovelRow |
|---|---|---|
| 封面 | 68×88 r6 | 56×72 r4（占位图无圆角） |
| 按压 | 无 PressScale | PressScale |
| 作者样式 | tokens.caption | theme.bodySmall |
| 字数 | 无 | 有 |
| 尾部 | 无 | chevron_right |
| key | `novel-<id>` | 无默认 |

两份都不支持：长按菜单、屏蔽呈现（`NovelEntity.isMuted` 字段存在但列表不消费）、排名槽位、收藏快捷动作（`BookmarkSwitchButton` 已支持 `isNovel`，见 bookmark_switch_button.dart:29,36）。

## 2. 插画卡片：`lib/app/widgets/feed/illust_card.dart` — `IllustCard`（408 行）

- 签名 `IllustCard({required entity, heroScope = 'feed'})`，默认 key `illust-<heroScope>-<id>`（34-35）。
- 预览按原图宽高比 `BoxFit.fitWidth`，瀑布流不裁切（157-159）——§4.6 明确"不统一图片裁切"。
- 交互：`PressScale`（164）+ `Semantics(container/button/image, label=title,author 或 muted 文案)`（165-173）+ `GestureDetector(onTapDown 预载, onTap openDetail, onLongPress showCardActionSheet)`（174-186）。
- Hero：`ClipRRect r12` 在 Hero 内部（237-255）。
- 徽标四角（258-325），**容器不统一**：
  - R-18 左上：`Card(color: primary)`，padding `5h/1v`（260-274）
  - ugoira 左下：`Container` `r5` `Color(0x99343838)`，`gif_box_outlined size 30`（275-290）
  - 页数 右上：`Container` `r7.5` 同色，padding `10h/5v`（291-308）
  - AI 右下：`Card(color: error)`，padding `5h/1v`（309-323）
- 标题行：10px 左缩进、title `label.bold`、作者 `caption`、尾部 `BookmarkSwitchButton`（327-357）。
- 屏蔽：`MutedCover` 模糊+原因标签，点击就地 reveal（会话级 `revealedMuteIdsProvider`）（129-135, 194-203；muted_cover.dart:25-60+）。

## 3. 其它"对象行"平行实现

| 组件 | 位置 | 形态 |
|---|---|---|
| `_UserRow` | `recommended_home_page.dart:399-429` | `Card` + `InkWell` + `AuthorSummary(avatarRadius 24)` + `chevron_right`，margin 同 NovelCard |
| `_WatchlistEntryTile` | `watchlist_page.dart:147-238` | `ListTile`，封面 `48×48 r6`，标题行内嵌 "New" 徽标（`errorContainer` `r4`），subtitle `user · date · count` |
| `_LocalNovelTile` | `local_novels_page.dart:84-109` | `ListTile`，leading 纯 icon `menu_book_outlined`（无封面），trailing delete `IconButton` |
| `_DownloadTaskTile` / `_DownloadGroupSection` | `download_tasks_page.dart:70-172, 174-257` | `Card`+`ListTile`，subtitle 内嵌 `LinearProgressIndicator` |
| `_KnownIllustEntry` / `_NovelHistoryEntry` / `_SnapshotEntry` | `history_page.dart:271-458` | 页面私有卡片：ClipRRect r10 封面 + title 2 行 + author；小说封面 `AspectRatio(1)` 正方形（437-458）；已知插画用 `MediaQuery` 固定两列估高（278-283），**不读 `FeedItemExtent`** |
| `_UserSeriesCardView` | `user_series_feed.dart:113-140` | `Card`+`InkWell`，封面 `AspectRatio(1.4)`，`FeedItemExtent.maybeOf(context) ?? 180` |

注：`test/architecture/layering_test.dart:84-86` 的 R5 规则禁止 `features/` 定义 `_*(Tail|Error|Empty|Card|Status|Placeholder)` 命名的类，allow-list 已清空（31-37）。`_HistoryCardFrame`/`_UserSeriesCardView` 因后缀为 Frame/View 恰好不命中，但语义上就是页面私造卡片——W6 迁移后应删除。

## 4. 可复用槽件（已存在的共享基件）

| 基件 | 位置 | 能力 |
|---|---|---|
| `AuthorSummary` | `app/widgets/author_summary.dart:14` | avatar+name(+account)，`compact` 单行变体，`avatarRadius`/`padding`/`onTap` |
| `TagChip`/`TagChips` | `app/widgets/tag_chips.dart:12,85` | label+translated，`onTap/onLongPress`，`blockMode` 徽标（62-79） |
| `WatchlistToggle` | `app/widgets/watchlist_toggle.dart:12` | `seriesKey`+`detailAdded`+`iconOnly`；store 观察覆盖远端值 |
| `BookmarkSwitchButton` | `app/widgets/bookmark_switch_button.dart:24` | `illustId`+`title`+`isNovel`+`isButton`；读 BookmarkStore pending |
| `PressScale` | `app/motion/press_scale.dart:25` | 被动缩放包装，不消费手势 |
| `FeedItemExtent` | `app/widgets/feed/feed_grid.dart:235` | 列宽 inherited（history 的已知插画条目未消费它） |
| `showCardActionSheet`+`CardAction` 注册表 | `app/widgets/card_actions/` | 插画长按动作注册表（bookmark/download/watchlater/mute×2/share）；**仅支持 IllustEntity，无小说版** |
| `showAppSnackBar` | `app/widgets/app_snack_bar.dart:49` | 支持 `SnackBarAction action`（撤销动作可用此槽） |
| `showAppBottomSheet`/`showAppDialog` | `app/motion/app_overlays.dart:9,36` | 统一弹层入口 |
| `FeedEmpty`/`FeedError`/`FeedLoading`/`FeedTail` | `app/widgets/feed/feed_states.dart` | 共享列表态；另有平行 `ReplicaEmptyState`（replica_empty_state.dart:9）——两个空态组件并存，history 用后者、其余管理页用前者 |

## 5. 触觉现状

`grep -r "HapticFeedback\|haptic" lib/` = **0 命中**。§5.6 的唯一 owner 由 W4 建立（当前 W4 仍为 planning），W6 只消费。
