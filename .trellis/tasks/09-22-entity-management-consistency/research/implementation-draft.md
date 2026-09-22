# W6 实现草案（implementation-draft）

逐条对应父任务 design.md §4.6。定位：调研结论 + 待设计确认的提案，不替代叶子的 design.md。

## 0. 依赖检查

- W6 必须等 W2（发现页结构）、W4（触觉 owner `lib/app/` 薄封装）、W5（阅读进度语义）合入 main 后 rebaseline。当前三者均 `planning`，`lib/` 内 `HapticFeedback` 零引用。
- §5.7 术语种子阶段 0 已冻结，直接消费。
- 本包内 5 个管理页 + 2 个排行接入点 + 5 处小说条目调用点 + 共享组件层，是最大包之一 → 建议 stage 拆分（见 risks.md）。

## 1. "一套小说条目" —— `NovelEntry` 单契约（§4.6-1，§5.2）

### 1.1 形态

新文件 `lib/app/widgets/novel_entry.dart`（或 `feed/` 子目录），一个泛型表达层 + 类型适配：

```dart
/// 基础对象行：识别槽位 + 基础动作。封面、标题、作者、语义元信息、
/// 按压反馈、可访问标签固定；排名/日期/进度/更新是命名 variant。
class EntityRow extends StatelessWidget {   // 槽位基件（仅识别与呈现）
  const EntityRow({
    required this.leading,      // 封面槽
    required this.title,        // 已本地化文本
    this.subtitle,              // 作者行
    this.meta,                  // 语义元信息行（字数/日期/进度文案）
    this.badge,                 // 覆盖徽标槽（排名/更新）
    this.progress,              // 进度条槽（0..1）
    this.trailing,              // 尾部动作槽（chevron/收藏/选择指示）
    this.onTap, this.onLongPress,
    this.selected = false,      // 管理模式选中态（表面变色+check）
    this.semanticLabel,
  });
}

class NovelEntry extends StatelessWidget {  // NovelEntity 类型适配
  NovelEntry({required this.entity, this.density, this.metaVariant, ...})
    : super(key: key ?? ValueKey('novel-${entity.id}'));
  const NovelEntry.compact(...)   // 封面 56×72 r4
  const NovelEntry.regular(...)   // 封面 68×88 r6
  const NovelEntry.ranking(..., required int rank)  // 名次列/徽标
}
```

要点：
- 封面两档密度固定为现有两种实测尺寸（compact=NovelRow 56×72 r4、regular=NovelCard 68×88 r6），占位图也走 ClipRRect（修 NovelRow 占位直角瑕疵，novel_row.dart:75-80）。
- 默认动作 `openNovel`；`trailing` 默认 null（feed 场景不挂动作，与管理模式契约一致）；可选 `BookmarkSwitchButton(isNovel: true)`——基件已支持小说（bookmark_switch_button.dart:29,36）。
- `selected` 槽位同时是管理模式呈现：M3 要求多选条目整行即选中单元、不挂次级动作。
- 所有行统一 `PressScale`（修 NovelCard 缺失）+ `Semantics(label: 'title, author')`（两者今皆无）。
- 字数、日期、进度、更新状态一律是 `meta`/`badge`/`progress` 槽或命名构造，页面不自绘（§5.2）。

### 1.2 NovelRow/Card 合并方案

| 步骤 | 内容 |
|---|---|
| 1 | 新增 `novel_entry.dart`（`EntityRow` + `NovelEntry`），带 widget 测试覆盖 variant |
| 2 | 迁移 5 个调用点：recommended(:362)→compact；novel_ranking(:183)→ranking；search_result(:247)、new(:382)、profile_novel_feed(:96)→regular |
| 3 | 删除 `novel_card.dart`/`novel_row.dart`（owning leaf 明确删除，§8） |
| 4 | 小说长按动作：可选项——`CardAction` 注册表目前只支持 `IllustEntity`（card_action.dart:24 签名绑定）；若 W6 要给小说行补长按菜单，需把注册表泛化（`CardAction<T>`）或新增 novel 动作集——**决策点**，W2 页面不依赖它 |

`_UserRow`（recommended_home_page.dart:399-429）可同构消费 `EntityRow`（AuthorSummary 进 leading/subtitle），但属 W2 页面范围——建议 W6 只提供基件，迁移与否由 W2/W6 协商（低风险顺手项）。

## 2. 插画卡片：只统一徽标角色（§4.6-2）

- 不碰裁切（`BoxFit.fitWidth` 瀑布流契约保留，illust_card.dart:157-159）。
- 抽 `_CardBadge` 私有（或 `feed/` 内共享）：统一容器 `r5-ish` 圆角、`Color(0x99343838)` 底、内边距一档；四角定位不变；语义色由参数注入（R-18=primary、AI=error、ugoira/页数=中性黑纱）。
- 加 `rank` 可选参数：排名徽标（左上，R-18 存在时垂直错开），供排行接入。

## 3. 排名变体接入（§4.6-1 后半 + W2 边界）

- `ranking_page.dart` `itemBuilder`（228-231）传 `rank: index + 1`；`novel_ranking_page.dart`（178-184）换 `NovelEntry.ranking(rank: index+1)`。
- 只改 itemBuilder/徽标槽；TabBar、RootSwipeSwitcher、novel 榜换 body 的结构收敛归 W2（astra #2）。

## 4. 历史页（§4.6-4）

- 找回：`IllustFeedGrid` 保留；`_HistoryCardFrame` 的日期行升级为对象组件的 `meta` 槽；illust 条目改 `IllustCard(entity, meta: date)`（含书签尾键保留）或共享 grid cell；novel 条目改 `NovelEntry` 的 grid-cell 密度（**新增第三密度需设计批准**，否则历史页小说保持方图 cell 但复用 `EntityRow` 的竖向变体——待定）。类型区分：封面角标（illust=novel icon 区分）或 meta 行前缀。
- 修复 `_KnownIllustEntry` 的 `MediaQuery` 两列估高（history_page.dart:278-283）→ 读 `FeedItemExtent.maybeOf`。
- 删除改为**明确管理动作**：长按进入选择模式（M3：AppBar→"已选 N"+删除+全选，条目整行选中态），删除走 `showAppBottomSheet` 确认（现有 `_confirmDelete` 460-500 可复用）；删除全部保留在 AppBar。
- 备选增强（PixEz 先例）：类型过滤 chips（全部/插画/小说）或日期分组头——工作量与 PRD 边界决策点。

## 5. 稍后再看（§4.6-5）

- 移除保持卡片长按 → `showCardActionSheet` → `_WatchLaterAction`（illust_card_actions.dart:103-148）。
- 补**撤销**：移除成功后 `showAppSnackBar(..., action: SnackBarAction(label: undo, onPressed: store.add(entry.entity)))`——`buildAppSnackBar` 已有 action 槽（app_snack_bar.dart:21-34）；store.add 幂等（watch_later_store.dart:36-43）。
- 不扩成收藏：不加标签/筛选/多选，管理动作止步于"移除+撤销"。新增 `undo`/`watchLaterRemoved` l10n 键。

## 6. 追更（§4.6-6）

`_WatchlistEntryTile`（watchlist_page.dart:147-238）改三动作语义：

| 动作 | manga | novel |
|---|---|---|
| 查看更新（tap 主动作） | `openIllustSeries(id)` 或 `openIllust(latestContentId)`——建议后者=直接看新话 | `openNovel(latest)`（现状 233） |
| 打开目录 | `openIllustSeries(id)` | **无小说系列目录页**——决策点：a) NovelPage 内系列条（770-845）只是 prev/next；b) 新建小说系列列表（fetchSeries 已有数据）；c) 本轮不为 novel 提供目录，设计写明 |
| 继续阅读 | `openIllust(seenId)`（seen 游标=最近打开内容，watchlist_store.dart:252-279） | `openNovel(seenId)` |
| 取消追更 | `watchlistActionsProvider.toggle(key)`（watchlist_toggle.dart:54） | 同左 |

- 呈现：`EntityRow` 承载（cover 48×48、title+New 徽标→`badge` 槽、subtitle meta）；trailing `more_vert` 菜单（`showAppBottomSheet` 动作清单）承载目录/继续/取消追更。
- 保留"打开即 markSeen"（797-814 novel_page 同规）但**只算更新游标**，UI 文案不得称其为阅读进度（§6）。

## 7. 本地书库（§4.6-7）

- 主操作=继续阅读：tile tap → `openLocalNovelReader`（local_novels_page.dart:102 现状）；`readOffset != null` 时 meta 行显示进度（offset/charCount，local_novel_repository.dart:25,39）。
- 删除下沉：trailing 改 `more_vert` → `showAppBottomSheet` 菜单（含删除）或选择模式；确认弹层统一走 `showAppDialog`（现状裸 `showDialog`，111-128）。
- 条目视觉向 `EntityRow` 靠拢（leading 书封/icon 槽）。

## 8. 下载任务页（§4.6-8）

### 8.1 父子层级

- 组卡内嵌子任务列表（按 `snapshot.groupId` 聚合，download_task.dart:86），子项缩进/分组于组卡之下；未入组任务平铺。组卡头=组状态+聚合进度+组级动作。

### 8.2 动作映射表（修正现状）

| 状态 | 文案 | 主动作 | 次动作 |
|---|---|---|---|
| queued | 排队中 | 取消 | — |
| running | 下载中 | 暂停 | 取消 |
| finalizing | 处理中（新增键或复用 Running） | — | — |
| canceling | 取消中 | — | — |
| retryable+paused | 已暂停 | **继续**（resume，`manager.retry` 续传 resumeAnchor） | 取消 |
| retryable+其它 | 失败 | 重试 | 取消 |
| failed | 失败 | 重试 | 移除* |
| canceled | 已取消 | 重新下载（`retry`） | 移除* |
| succeeded | 已完成 | **查看**（`openIllust(task.illustId)`） | 移除* |
| orphaned | 失败 | —（说明文案） | 移除* |

\* "移除"需要 `DownloadManager` 新增 dismiss/clear API（当前终态任务无出口，manager:108-109）——**核心层增量，需在 PRD 声明**（§6 允许的最小必要边界）。
组级映射：queued/running→暂停+取消；retryable/failed/canceled→继续/重试全部+取消；succeeded→查看（第一个成功子项作品页）+移除。

### 8.3 提交反馈直达

三处提交点 SnackBar 加 `action`："查看"→ `openSettingsPage(context, '/settings/tasks')`（路由 routes.dart:730-734）：`illust_card_actions.dart:90`、`page_image.dart:241`、`user_page.dart:268`。§5.7："查看"动词统一，不与"打开页面"混用——下载查看一律 `查看`。

### 8.4 可选增强（PixEz 对标）

状态过滤 chips、清空已完成（依赖 8.2 的移除 API）、排序——按 PRD 裁剪。

## 9. 管理列表限宽（§4.6-9，§5.5）

- 5 个管理页满足 ≥3 消费者门槛。建议扩展现有 owner：`AppBreakpoints` 增 `managementContentMaxWidth`（如 840dp）静态量或 `contentMaxWidth(role)`；页面 body 包 `Align/Center + ConstrainedBox`。grid 类（history/watchlater）是否限宽待定——瀑布流多列本身自适应，限宽反而降信息密度；**建议限宽只覆盖 ListView 型管理页**（watchlist/localnovels/download_tasks + history 若改列表形态）。
- 不得各自发明宽度常量（§5.5）。

## 10. 触觉消费点（§5.6，依赖 W4 owner）

| 时机 | 级别 | 位置 |
|---|---|---|
| 进入选择/管理模式（长按触发） | 明确震动（medium/heavy 档） | history、download tasks、（若启用）watchlater/localnovels |
| 管理模式内勾选切换 | 轻触（selectionClick 档） | 同上 |
| 危险确认弹出（删除确认 sheet 打开） | 明确震动 | history 删除、本地书库删除、下载取消 |
| 移除+撤销成功 | 轻触或不加 | watchlater |
| 普通条目点击 | 不加 | 全局 |

实现=调 W4 建立的 `lib/app/` 薄封装；若 W6 启动时 W4 未合入，触觉接入拆为最后 checkbox 等待。

## 11. 术语核对（§5.7）

- 删除（不可恢复，有确认）：历史删除/清空、本地书库删除、下载移除。
- 移除（列表条目，可撤销）：稍后再看移除。
- 取消：仅终止当前操作（下载取消=终止传输并弃部分输出——文案 `cancelDownload` 已合规）。
- 重试 vs 继续：retryable+paused→继续；failed/canceled→重试/重新下载。
- 查看：下载完成看结果、提交反馈直达任务页。
- 新增 l10n 键清单（草案）：`undo`、`watchLaterRemoved`、`manage`/`selectAll`/`selectedCount(n)`、`downloadResume`（区分 retryDownload）、`downloadViewResult`、`downloadRemoveRecord`、`historyTypeIllust/Novel`（若做过滤）。

## 12. 测试计划锚点

- variant widget tests：`NovelEntry` 各密度/variant 渲染与主动作不丢（W6 门禁：shared object variant 在所有调用点不丢主动作）。
- 下载动作映射：全 9 状态 + failureKind=paused 分支；更新 `download_tasks_page_test.dart` 中固化"paused→refresh 图标"的用例（203-254）。
- 管理模式：进入/勾选/全选/取消/删除确认/返回退出模式。
- 撤销：watchlater 移除→snackbar action→store 复原。
- 宽屏：840/1200dp 限宽截图或 layout 断言；320dp 无不可达动作。
