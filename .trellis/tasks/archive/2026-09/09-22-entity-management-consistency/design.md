# 技术设计：对象组件与管理页一致（W6）

设计基线：`main@8067b2d`。逐文件精确改动方案见
`research/implementation-draft.md`；现状核实（行号）见
`research/codebase-object-components.md`、`codebase-management-pages.md`、
`codebase-download-tasks.md`；外部对标见 `research/external-pixez-m3.md`；
风险与决策点见 `research/risks.md`。本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

按 `risks.md` §1 预拆三个 stage，分支命名
`task/09-22-entity-management-consistency-<stage>`（连字符，与 leaf 分支
不冲突为 ref）：

| stage | 分支后缀 | 内容 | 门禁 |
|---|---|---|---|
| 1 | `objects` | `EntityRow`/`NovelEntry` 基件+variant、`IllustCard` 徽标统一+rank+meta 槽、5 个小说调用点迁移、2 个排行页接入、删 novel_card/novel_row | variant 测试全绿；裁切/列表结构零回归 |
| 2 | `management` | history 选择模式+条目迁移、watchlater 撤销、watchlist 三动作、localnovels 菜单化、限宽、触觉消费 | 长按旧语义→新模式迁移说明已随 PRD 固定 |
| 3 | `downloads` | 下载页父子层级、九态动作映射、查看动词、提交反馈直达、`DownloadManager` dismiss/clear API | 九态映射测试全覆盖；旧断言同 commit 改写 |

串行执行：stage 1 是后两者的组件前提；2 与 3 文件零重叠但同包串行控制
PR 体积。若 stage 2 仍过大，可按 risks.md 再分 `management-lists`
（history+watchlater）与 `management-rows`（watchlist+localnovels）。

### 启动门禁（rebaseline）

三个前置全部合入 `main` 后才开始 stage 1：

- W2（发现页结构）：排行/推荐/新作/搜索页文件归 W2，本 leaf 只动
  itemBuilder/组件调用行（行级边界见 §四）。
- W4（触觉 owner）：`lib/app/` 薄封装由 W4 建立；W6 只消费。
- W5（进度语义）：`NovelEntry` progress 槽位的 null↔0 边界随 W5 事实定；
  stage 1 先交付槽位+字数/排名/日期 variant，progress 变体在 W5 合入后补。

rebaseline 动作：记录新 `main` SHA → 复核本 leaf owning files 行号 →
跑聚焦测试记录真实基线 → 事实使范围实质变化时回规划审阅。

## 二、关键契约

### EntityRow / NovelEntry（父 §5.2 落点）

```dart
// lib/app/widgets/entity_row.dart —— 槽位基件：只拥有识别与基础动作
class EntityRow extends StatelessWidget {
  const EntityRow({
    required this.leading,      // 封面槽
    required this.title,        // 已本地化文本
    this.subtitle,              // 作者行
    this.meta,                  // 语义元信息行（字数/日期/进度文案）
    this.badge,                 // 覆盖徽标槽（排名/更新）
    this.progress,              // double? 0..1；null=无进度（不渲染）
    this.trailing,              // 尾部动作槽（chevron/more_vert/选择指示）
    this.onTap, this.onLongPress,
    this.selected = false,      // 管理模式选中态：整行变色+check
    this.semanticLabel,
  });
}

// lib/app/widgets/novel_entry.dart —— NovelEntity 类型适配
class NovelEntry extends StatelessWidget {
  NovelEntry({required this.entity, ...})
    : super(key: key ?? ValueKey('novel-${entity.id}'));
  const NovelEntry.compact(...)   // 封面 56×72 r4（=现 NovelRow 实测）
  const NovelEntry.regular(...)   // 封面 68×88 r6（=现 NovelCard 实测）
  const NovelEntry.ranking(..., required int rank)  // 名次徽标
}
```

> **密度决策（评审补充，防止"改名式合并"）**：compact/regular 的差异
> 必须有产品理由而非历史惯性——定案：density 由**列表角色**决定。
> `compact` = 次级内联 feed（推荐页混排流，小说是辅内容、一屏条目多）；
> `regular` = 主内容列表（搜索结果/新作/作者页，小说是页面主体）。
> 「同一本小说在推荐流是次要内容、在搜索结果是主体」是成立的理由。
> 排名/进度/更新时间是**附加信息槽**，不隐含第二套基础条目设计。
> 若实现中发现某调用点的角色归属说不通，回写到本节再落码。

- 统一 `PressScale` + `Semantics(label: 'title, author')`；占位图走
  `ClipRRect`（修 `novel_row.dart:75-83` 直角瑕疵）。
- `trailing` 默认 null（feed 场景不挂动作）；可选
  `BookmarkSwitchButton(isNovel: true)`（`bookmark_switch_button.dart:29,36`
  已支持小说）。
- `progress` 槽：`null` 与 `0.0` 语义互异——null=无进度记录不渲染，
  0.0=从头开始渲染空条（W5 null↔0 边界）。
- `selected` 即管理模式呈现：整行是选中单元、不挂次级动作（M3）。
- 字数/日期/进度/更新状态一律槽位或命名 variant，页面不自绘。

### IllustCard 徽标契约

- 抽共享徽标容器：统一 `r5-ish` 圆角、`Color(0x99343838)` 底、内边距
  一档；四角定位不变；语义色参数注入（R-18=primary、AI=error、
  ugoira/页数=中性黑纱）。
- 新增 `rank`（int?）与 `meta`（Widget?，历史页日期行）可选槽。
- **不动** `BoxFit.fitWidth` 裁切契约（illust_card.dart:157-159）与
  Hero/预载/屏蔽呈现。

### 选择/管理模式契约（M3 contextual action bar）

- 进入：长按条目（明确震动档）；退出：AppBar 关闭/系统返回/逐个取消。
- AppBar 切换：title→"已选 N"，actions→全选+删除+退出，容器色
  `primaryContainer`。
- 模式下条目整行=选中单元（表面变色+check），**不挂次级动作**（M3：
  "can't have secondary nested actions"）。
- 适用范围（D5）：仅 history 与 download_tasks；watchlater 保持长按
  动作 sheet；localnovels 用条目 more_vert 菜单。
- 选择状态是页面本地 `Set<id>`，不进 provider（无跨页消费者）。

### 下载九态 → 动作映射表

| status（+failureKind） | 文案 | 主动作 | 次动作 |
|---|---|---|---|
| queued | 排队中 | 取消 | — |
| running | 下载中 | 暂停 | 取消 |
| finalizing | 处理中 | — | — |
| canceling | 取消中 | — | — |
| retryable + paused | 已暂停 | **继续**（`manager.retry` 续传 resumeAnchor） | 取消 |
| retryable + 其它 | 失败 | 重试 | 取消 |
| failed | 失败 | 重试 | 移除 |
| canceled | 已取消 | 重新下载（`retry`） | 移除 |
| succeeded | 已完成 | **查看**（`openIllust(task.illustId)`） | 移除 |
| orphaned | 失败 | —（说明文案） | 移除 |

组级映射：queued/running→暂停+取消；retryable/failed/canceled→
继续/重试全部+取消；succeeded→查看（第一个成功子项）+移除。

父子层级：子任务按 `snapshot.groupId`（`download_task.dart:86`）聚合
缩进于组卡下；未入组任务平铺。

### DownloadManager 核心层增量（D2，PRD 已声明）

```dart
// lib/core/download/download_manager.dart —— 终态记录出口
bool dismiss(String taskId);   // 仅 isTerminal 状态可移除；非终态 no-op→false
int clearTerminal();           // 清空全部终态记录，返回移除数
```

- 只删终态快照记录（`_jobs` 出队 + 恢复记录清理 + `changes` 通知）；
  不改 pause/cancel/retry/recover 语义、不动终态事件恰好一次的
  `_complete` 契约（manager:1101-1106）。
- 组级出口：`dismissGroup(groupId)` 或借 `clearTerminal` 覆盖——实现时
  取最小面。

### 提交反馈直达

- 三处 `downloadQueuedMessage` SnackBar 加 `action: 查看` →
  `openDownloadTasks(context)`（新门面，routes.dart，
  `context.push('/settings/tasks')`，路由已在 routes.dart:730-734）。
- 动词统一"查看"（§5.7），不与"打开页面"混用。

### 触觉消费点（§5.6，消费 W4 owner）

| 时机 | 级别 | 位置 |
|---|---|---|
| 进入选择/管理模式 | 明确震动 | history、download_tasks |
| 模式内勾选切换 | 轻触 | 同上 |
| 危险确认弹出 | 明确震动 | history 删除、localnovels 删除、下载取消 |
| 移除+撤销成功 | 轻触或不加 | watchlater |
| 普通条目点击 | 不加 | 全局 |

全部经 W4 的 `lib/app/` 薄封装；`lib/` 内不得出现 `HapticFeedback` 直调。

### 术语与 l10n 新键（§5.7）

- 动词核对：删除（history 删除/清空、localnovels 删除、下载移除记录）/
  移除（watchlater 条目，可撤销）/取消（仅终止当前操作）/
  重试 vs 继续（retryable+paused→继续）/查看（下载完成、反馈直达）。
- 新键草案：`undo`、`watchLaterRemoved`、`manage`、`selectAll`、
  `selectedCount(n)`、`downloadResume`（区分 retryDownload）、
  `downloadViewResult`、`downloadRemoveRecord`、`downloadProcessing`（或
  复用 Running）、`watchlistOpenContents`/`watchlistContinueReading`/
  `watchlistUnwatch`、`localNovelContinue`。
- 流程：`app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
  `python3 tool/gen_l10n_lookup.py`；arb 是高冲突区，按 owner 合入顺序
  串行。

### 限宽 owner（§5.5）

- 读 `ContentWidths.management`（840，`lib/app/layout/content_widths.dart`，
  父 §5.5 冻结角色表；**内容宽度常量不进 `AppBreakpoints`**——断点管
  换布局，角色常量管布局内宽度）。文件未建时本包首个消费创建全表；
  禁止页面自造常量。
- 接入面：watchlist/localnovels/download_tasks 三个 ListView 型管理页，
  body 包 `Center + ConstrainedBox(maxWidth:)`；grid 型
  （history/watchlater）由 `illustColumnsFor` 自然加列，不限宽。

## 三、复用与禁止

- 复用：`PressScale`、`PixivImage.feed`、`Semantics`、`IllustFeedGrid`/
  `FeedItemExtent`、`FeedEmpty/FeedError/FeedTail`、`PullToRefresh`、
  `showAppSnackBar`（含 `SnackBarAction` 槽）、`showAppBottomSheet`/
  `showAppDialog`、`BookmarkSwitchButton`、`WatchlistToggle`/
  `watchlistActionsProvider`、`AuthorSummary`、`StaggeredEntrance`、
  `IllustCard`/`showCardActionSheet`、l10n 工具链。
- 禁止：新建 SnackBar/弹层/刷新/断点/触觉第二来源；统一扩大为裁切统一
  或列表结构统一；`CardAction<T>` 泛化（D7）；novel 系列目录页（D3）；
  watchlater 扩成收藏；`ReplicaEmptyState` 组件删除（D6）；页面自造
  宽度常量或平行选择模式框架。

## 四、跨包协调与文件边界

| 文件 | 归属 | 本 leaf 改动面 |
|---|---|---|
| ranking_page.dart / novel_ranking_page.dart | W2 | 仅 itemBuilder 组件调用行（228-231 / 178-184） |
| recommended_home_page.dart / new_page.dart / search_result_page.dart | W2 | 仅小说条目调用行（362 / 382 / 247） |
| profile_novel_feed.dart | W3 | 仅 `NovelCard`→`NovelEntry.regular` 一行（96） |
| novel_card.dart / novel_row.dart | 本 leaf | 删除（全调用点迁移后） |
| illust_card.dart | 本 leaf | 徽标容器+rank+meta 槽 |
| entity_row.dart / novel_entry.dart | 本 leaf | 新建 |
| history/watchlater/watchlist/localnovels/download_tasks 页 | 本 leaf | 全量迁移 |
| download_manager.dart | 本 leaf | 仅新增 dismiss/clearTerminal（D2） |
| illust_card_actions.dart / page_image.dart / user_page.dart | 本 leaf / W4 / W3 | SnackBar action 一行（90 / 241 / 268）；W4/W3 文件的该行级改动排在其合入后 |
| routes.dart | 本 leaf | 仅追加 `openDownloadTasks` 门面函数（高冲突区，最小改动） |
| app_breakpoints.dart | 本 leaf | 仅追加 `managementContentMaxWidth` |
| app_*.arb | 本 leaf | 新键追加（与 W8 串行） |

W2/W3/W4 文件均为行级接触、排在其合入后；若 rebaseline 时行号已漂移，
以当时 HEAD 重定位。

## 五、测试策略

- `NovelEntry` variant widget 测试：compact/regular/ranking 渲染、
  `novel-<id>` key、主动作 openNovel 不丢；`EntityRow` 槽位/selected 态。
- `IllustCard`：徽标容器统一断言 + rank 徽标 + fitWidth 不回归。
- 管理模式：进入/勾选/全选/取消选择/删除确认/返回退出。
- watchlater：remove→snackbar action→store.add 复原。
- watchlist：三动作+manga/novel 分支+取消追更。
- localnovels：tap=openLocalNovelReader、删除经菜单+showAppDialog。
- 下载：九态+failureKind=paused 全映射；`dismiss`/`clearTerminal` 单测；
  父子聚合；`download_tasks_page_test.dart` 旧断言同 commit 改写
  （`find.byIcon(Icons.refresh)` 位置断言更新）。
- 提交反馈：三处 SnackBar action → `/settings/tasks` 导航断言。
- 限宽：840/1200dp ConstrainedBox 断言；320dp 无不可达动作。
- 注意 spec 陷阱：material_ui 遮蔽、autoDispose `.future` 监听、
  loopback 噪声 retry、HistoryDatabase 注入 temp path。

## 六、运行时证据缺口（PR 标"未验证"）

IllustCard 徽标视觉回归需宽度矩阵人工核对（320/390/600/840/1200dp）；
选择模式长按手感与触觉分级真机确认；840dp 限宽观感；TalkBack/Narrator
朗读选中态与徽标标签；1.3x 大字体下 EntityRow meta 行溢出；
下载页父子缩进的横屏表现。
