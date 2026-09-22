# W6 风险与拆分建议

## 1. 规模风险：本包为最大包之一

implement.md §2 已点名"W6、W8 为最大工作包……先按 stage 预拆分支"。本调研实测涉及面：

- 共享组件层：新建 `NovelEntry`/`EntityRow` + `IllustCard` 徽标统一 + rank 槽（3 个 `app/widgets` 文件级变更）
- 调用点：5 处小说条目 + 2 个排行页 itemBuilder + history 页内 4 类私有条目
- 管理页：5 页各自的状态映射/管理模式/限宽接入
- 下载页：父子层级重组 + 9 状态映射表 + 可能的核心层 dismiss API
- 横切：新 l10n 键（本地化是高冲突区，design §8）、触觉消费（依赖 W4）

### 建议 stage 拆分（`task/09-22-entity-management-consistency-<stage>`）

| stage | 分支后缀 | 内容 | 门禁 |
|---|---|---|---|
| 1 | `objects` | `EntityRow`/`NovelEntry` 基件+variant、IllustCard 徽标统一+rank 槽、5 个小说调用点迁移、2 个排行页接入、删除 novel_card/novel_row | variant 测试全绿；无视觉回归断言 |
| 2 | `management` | history/watchlater/watchlist/localnovels 迁移 + 选择模式 + 撤销 + 限宽 + 触觉消费 | 长按旧语义→新模式有 PRD 迁移说明（implement.md §4-7 要求） |
| 3 | `downloads` | 下载页层级/动作映射/查看动词/提交反馈直达（+可选 dismiss API） | 9 状态映射测试全覆盖 |

理由：stage 1 是后两者的组件前提（串行）；2 与 3 文件零重叠可并行，但同包内仍建议串行以控制 PR 体积。若叶子规划期想再压 PR，`management` 可再分 `management-lists`（history+watchlater）与 `management-rows`（watchlist+localnovels）。

## 2. 依赖风险

- **W4 触觉 owner 未落地**：W4/W2/W5 均 planning；`lib/` 无 haptics。W6 计划若先于 W4 启动，§5.6 消费点无源可接——stage 2/3 的触觉 checkbox 必须显式等待或回规划。
- **W2 共享文件**：`ranking_page.dart`/`novel_ranking_page.dart`/`recommended_home_page.dart`/`new_page.dart`/`search_result_page.dart` 同时是 W2 工作面。§3 "不让两个并行分支编辑同一文件"——W6 只动 itemBuilder/组件调用行，须在叶子 owning-files 清单中写行级边界并排在 W2 合入后。
- **W5 进度语义**：`NovelEntry` 的 progress 变体依赖 W5 的真实阅读进度落地形态（更新游标≠阅读进度）；stage 1 先交付槽位+字数/排名/日期变体，progress 变体随 W5 事实补。

## 3. 决策点（需 PRD/design 批准前明确）

1. **`EntityRow` 泛型基件 vs 纯 `NovelEntry`**：泛型可被 watchlist/localnovels/user rows 复用（§5.2 槽位契约的字面实现），但面更大；保守版只做 NovelEntry+各页自行组合。**建议泛型**——watchlist/localnovels/download 三页已在包内，≥3 消费者成立。
2. **下载"移除/清空"需要 `DownloadManager` 新 API**（终态任务当前无出口）——核心层最小增量，§6 允许但需 PRD 声明；不批则终态条目只能留"查看/重试"。
3. **小说系列"打开目录"**：无对应路由/页面（只有 illust series 页）；选项=NovelPage 系列条替代 / 新建目录 sheet / 本期不覆盖 novel 目录。
4. **历史页小说条目形态**：grid 内方形 cell（现状）→ 需 `NovelEntry` 第三密度或保留私有 cell；或历史页改列表布局（改动大，不推荐）。
5. **管理模式交互形态**：长按进选择（M3 标准、PixEz 无此模式）vs 条目 overflow 菜单——历史页"长按=直接删"是肌肉记忆变更，PRD 必须写旧→新迁移说明（implement.md §4-7）。
6. **`ReplicaEmptyState` vs `FeedEmpty` 双空态并存**：迁移时收敛其一或保持现状（组件层既有债务，可不在本包解决但需声明）。
7. **小说长按动作菜单**：`CardAction` 泛化（`CardAction<T>`）属额外扩张，建议不在 W6 做，记入待办。

## 4. 回归风险

- `download_tasks_page_test.dart` 固化"paused→refresh 图标+retry"（203-254）：改"继续"语义必须同步改测试。
- `IllustCard` 是全站最热组件（recommended/ranking/new/search/profile/related/series/watchlater…）：徽标容器统一是纯视觉变更，需 golden 或人工宽度矩阵回归（§7 运行时矩阵）。
- `NovelEntry` 默认 key `novel-<id>` 保留 NovelCard 的 id 追踪语义；NovelRow 调用点原本无 key——迁移后 id key 改变 element 复用行为（正向，但 entrance played 集合语义不变）。
- 历史页 `PageStorageKey`/`restorationId`、watchlater `restorationId: 'watchlater'` 保持；watchlist/localnovels/download 当前无恢复 id，迁移时可顺手补（声明式小改）。
- l10n 新增键走生成工作流（quality-guidelines: generated files 只经 owning workflow）；与 W8 同期改 arb 文件是高冲突区，按 owner 合入顺序串行。

## 5. 已验证的"不要重建"清单

- 不新建 SnackBar/弹层/刷新/断点第二来源（`showAppSnackBar`/`showAppBottomSheet`/`PullToRefresh`/`AppBreakpoints` 已就位）。
- 不新建触觉封装（W4 唯一 owner）。
- 不把统一扩大为裁切统一、列表结构统一（§4.6、astra §6）。
- 不把 watchlater 扩成收藏体系。
