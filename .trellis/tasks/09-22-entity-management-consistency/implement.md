# 执行计划：对象组件与管理页一致（W6）

需求见 `prd.md`，技术设计见 `design.md`，逐项精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实）。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
`python3 tool/gen_l10n_lookup.py`。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-entity-management-consistency
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：
> 看似不相关的测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec
> 判定噪声再排查。widget test 触碰 SnackBar/SwitchListTile 须用
> `material_ui` 的 `MaterialApp` 与类型；HistoryDatabase 注入 temp path；
> autoDispose provider 先持监听再 read `.future`。

本机不可验项（真机触觉分级、TalkBack/Narrator、宽度矩阵观感、1.3x 大字体
溢出、横屏父子缩进）在 PR body 标"未验证"，不得由 widget test 推断通过。

## Stage 0：启动门禁（rebaseline，先于所有分支）

- [x] 确认 W2/W4/W5 均已合入 `main`；记录新 `main` SHA；复核
      design.md §四 文件边界的行号漂移；跑聚焦测试记录真实基线。
      若 W4 未合入：本 leaf 不启动（触觉无源可接，禁止临时封装）。
      若 W5 未合入但其余已合入：可启动，progress variant 条目推后到
      W5 落地后补（design.md §一）。
      **W3 交接条件**：`profile_novel_feed.dart`、`user_page.dart:268`
      属 W3 文件——O4 的 profile 调用点与 R8 的 SnackBar action 行
      仅在 W3 合入后落笔；W3 未合入则这两处行级改动推迟，
      不阻塞其余 stage（与 W5 的降级条款同构）。

## Stage 1：objects — 对象组件与调用点迁移

分支：`task/09-22-entity-management-consistency-objects`

- [x] **O1**：新增 `lib/app/widgets/entity_row.dart`（`EntityRow` 槽位
      基件：leading/title/subtitle/meta/badge/progress/trailing/
      onTap/onLongPress/selected/semanticLabel；`PressScale`+`Semantics`
      固定；契约见 design.md §二）。
      测试：槽位渲染、selected 整行变色+check、semanticLabel。
      提交：`feat(app): 新增 EntityRow 对象行槽位基件`
- [x] **O2**：新增 `lib/app/widgets/novel_entry.dart`（`NovelEntry`
      compact=56×72 r4 / regular=68×88 r6 / ranking(rank:) 三 variant；
      默认 key `novel-<id>`、默认动作 `openNovel`、占位图走 ClipRRect；
      progress 槽预留，null↔0 边界见 design.md §二）。
      测试：三 variant 渲染、key、tap→openNovel、占位圆角。
      提交：`feat(app): 新增 NovelEntry 单契约条目及命名 variant`
- [x] **O3**：`lib/app/widgets/feed/illust_card.dart` 徽标容器统一
      （共享圆角/底色/内边距，语义色注入；四角定位不变）+ `rank` 与
      `meta` 可选槽；**不动** fitWidth 裁切与 Hero/屏蔽路径。
      测试：徽标容器一致性断言、rank 徽标渲染、fitWidth 回归。
      提交：`feat(app): 统一 IllustCard 徽标容器并新增排名与 meta 槽`
- [x] **O4**：迁移 5 个小说调用点（行级边界，W2/W3 已合入前提）：
      `recommended_home_page.dart:362`→compact；
      `novel_ranking_page.dart:183`→ranking(rank: index+1)；
      `search_result_page.dart:247`、`new_page.dart:382`、
      `profile_novel_feed.dart:96`→regular。
      测试：各调用点既有测试更新；主动作不丢断言。
      提交：`refactor(features): 小说条目调用点迁移至 NovelEntry`
- [x] **O5**：排行接入：`ranking_page.dart` itemBuilder（228-231）传
      `IllustCard(rank: index + 1)`（O4 已含 novel 榜）。
      测试：rank 徽标出现在榜单条目。
      提交：`feat(ranking): 排行条目接入排名变体`
- [x] **O6**：删除 `lib/app/widgets/novel_card.dart` 与
      `lib/app/widgets/novel_row.dart`（owning leaf 明确删除，父 §8）。
      测试：全量 analyze + 受影响测试回归。
      提交：`refactor(app): 删除 NovelCard 与 NovelRow 双实现`

## Stage 2：management — 管理页迁移（历史/稍后再看/追更/本地书库）

分支：`task/09-22-entity-management-consistency-management`（自 stage 1
合入后的 main 切出）

- [x] **M1**：`history_page.dart` 选择模式——长按进入（W4 触觉明确震动
      档），AppBar 换"已选 N"+全选+删除+退出（`primaryContainer`），
      选中条目整行变色+check 不挂次级动作；删除走 `showAppBottomSheet`
      （复用 `_confirmDelete` 460-500 形态）；系统返回退出模式。
      新 l10n：`manage`/`selectAll`/`selectedCount(n)`（四语）。
      测试：进入/勾选/全选/删除确认/退出全链路；长按旧语义用例改写。
      提交：`feat(history): 长按进入选择模式，删除收进管理动作`
- [x] **M2**：`history_page.dart` 条目迁移——illust 条目改
      `IllustCard(meta: 日期)`；`_KnownIllustEntry` 的 `MediaQuery` 两列
      估高（278-283）改读 `FeedItemExtent.maybeOf`；novel 方形 cell 按
      对象契约对齐（圆角/PressScale/Semantics/类型角标，D4）；
      `_HistoryCardFrame` 日期行升级为共享 meta 呈现；空态
      `ReplicaEmptyState` →
      `FeedEmpty`（D6）。
      测试：FeedItemExtent 消费断言、日期 meta 渲染、类型可辨。
      提交：`refactor(history): 历史条目迁移至共享对象组件契约`
- [x] **M3**：`watchlater_page.dart` + `illust_card_actions.dart:103-148`
      移除后 `showAppSnackBar` 带 `action: 撤销`（`store.add` 幂等）；
      不扩成收藏。
      新 l10n：`undo`、`watchLaterRemoved`。
      测试：移除→撤销→store 复原。
      提交：`feat(watchlater): 移除后支持撤销`
- [x] **M4**：`watchlist_page.dart` `_WatchlistEntryTile`（147-238）改
      `EntityRow`（48×48 封面、"New"→badge、`user·date·count`→meta）；
      tap=查看更新（manga→`openIllust(latest)`、novel→`openNovel(latest)`）；
      trailing `more_vert`→`showAppBottomSheet`：打开目录（仅 manga，
      `openIllustSeries`，D3 novel 降级不显示）/继续阅读（seenId）/
      取消追更（`watchlistActionsProvider.toggle`）；markSeen 仅作
      更新游标。
      新 l10n：`watchlistOpenContents`/`watchlistContinueReading`/
      `watchlistUnwatch`。
      测试：manga/novel 分支三动作+取消追更；novel 菜单无目录项。
      提交：`feat(watchlist): 区分查看更新、打开目录、继续阅读与取消追更`
- [x] **M5**：`local_novels_page.dart`——条目向 `EntityRow` 靠拢；
      tap=`openLocalNovelReader`；`readOffset != null` 时 meta 显示进度
      文案（null↔0 边界）；trailing delete 图标改 `more_vert`→
      `showAppBottomSheet` 菜单；确认弹层裸 `showDialog`（111-128）改
      `showAppDialog`。
      新 l10n：`localNovelContinue`。
      测试：tap=继续阅读、删除经菜单+统一确认弹层。
      提交：`feat(localnovel): 主操作改为继续阅读，删除下沉更多菜单`
- [x] **M6**：`app_breakpoints.dart` 增 `managementContentMaxWidth=840`；
      watchlist/localnovels 两页 body 包 `Center+ConstrainedBox`
      （download_tasks 随 stage 3 接入）。
      测试：840/1200dp 限宽断言；320dp 无不可达动作。
      提交：`feat(app): 管理列表接入限宽断点`
- [x] **M7**：触觉消费接入（W4 owner 薄封装）：进入选择模式=明确震动、
      勾选=轻触、删除确认弹出=明确震动（history/localnovels）、
      watchlater 撤销=轻触或不加。`lib/` 零 `HapticFeedback` 直调。
      测试：封装调用点断言（可 mock 薄封装）；真机手感标"未验证"。
      提交：`feat(app): 管理页消费共享触觉分级`

## Stage 3：downloads — 下载任务页与提交反馈

分支：`task/09-22-entity-management-consistency-downloads`（自 stage 2
合入后的 main 切出）

- [x] **T1**：`download_manager.dart` 新增 `dismiss(taskId)`/
      `clearTerminal()`（仅终态可移除；`_jobs` 出队+恢复记录清理+
      `changes` 通知；不改 pause/cancel/retry/recover 与 `_complete`
      恰好一次契约——D2 核心层增量）。
      测试：core 单测（终态可删/非终态 no-op/清空计数/记录清理）。
      提交：`feat(download): 新增终态任务移除与清空 API`
- [x] **T2**：`download_tasks_page.dart` 父子层级——子任务按
      `snapshot.groupId` 聚合缩进于组卡下；未入组平铺；组卡头=组状态+
      聚合进度+组级动作。
      测试：聚合渲染断言（组内子项不再出现于顶层）。
      提交：`feat(settings): 下载任务按组聚合父子层级`
- [x] **T3**：九态动作映射重写（表见 design.md §二）：
      retryable+paused→**继续**（非 retry 图标）；succeeded→**查看**
      （`openIllust(task.illustId)`）；failed/canceled→重试+移除；
      orphaned→说明+移除；组级映射同步。
      **同 commit 改写 `test/download_tasks_page_test.dart`** 固化旧行为
      的用例（203-254：`paused tile shows paused status with retry and
      cancel` 的 `find.byIcon(Icons.refresh)` 断言 → 继续语义断言）。
      新 l10n：`downloadResume`/`downloadViewResult`/`downloadRemoveRecord`/
      `downloadProcessing`（或复用 Running）。
      测试：九态+failureKind=paused 全映射覆盖。
      提交：`feat(settings): 下载九态动作映射对齐术语契约`
- [ ] **T4**：提交反馈直达——`routes.dart` 追加 `openDownloadTasks`
      门面（`context.push('/settings/tasks')`）；三处
      `downloadQueuedMessage` SnackBar 加 `action: 查看`
      （`illust_card_actions.dart:90`、`page_image.dart:241`、
      `user_page.dart:268`；W3/W4 文件行级接触）。
      测试：action 点击 → `/settings/tasks` 导航断言。
      提交：`feat(download): 提交反馈可直达任务页`
- [ ] **T5**：download_tasks 选择模式（批量移除终态/批量取消）+
      限宽接入（M6 的 `managementContentMaxWidth`）+ 补
      `restorationId`（watchlist/localnovels 亦顺手补，声明式小改）+
      触觉（进入选择/勾选/取消确认）。
      测试：批量移除/取消链路；限宽断言。
      提交：`feat(settings): 下载任务选择模式与限宽接入`

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec
      判定）；`git diff --check` 干净；`task.py validate` 通过。
- [ ] 三 stage 各开 PR（`gh pr create --fill`），CI 绿后
      `gh pr merge --merge`，串行合入。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随末个 PR 的最后
      提交）。

## 边界（不做）

- 榜单 TabBar/切换结构与发现页层级（W2）；`_UserRow` 迁移与否由 W2 定
  （D9）；profile_novel_feed 仅一行组件替换，其余归 W3。
- `CardAction<T>` 泛化与小说长按动作集（D7）；novel 系列目录页（D3）；
  watchlater 收藏化；`ReplicaEmptyState` 组件删除（D6）；裁切统一；
  下载过滤 chips/排序（可选增强，按 PR 体积裁剪）。
- 真实阅读进度填充 progress 槽——槽位先行，数据随 W5 事实接入；
  更新游标不得冒充阅读进度（父 §6）。
