# 对象组件与管理页一致（Roadmap W6）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图，
§4.6 工作包范围、§4.11 依赖冻结、§5 跨包契约、§7 验收矩阵）。
规划基线：`main@8067b2d`（PR #54 合入后）。全部代码断言由本 leaf
`research/codebase-*.md` 在当前 HEAD 逐条复核；行号以 research 为准。

## Goal

以一套小说条目组件（`EntityRow` 槽位基件 + `NovelEntry` 命名 variant）替换
`NovelRow`/`NovelCard` 双实现，统一 `IllustCard` 徽标容器并接入排名变体；
把历史、稍后再看、追更、本地书库、下载任务五个管理页迁到统一的对象呈现、
明确管理动作与一致动作映射上。本 leaf 不收敛页面信息架构（W2/W8）、
不统一图片裁切、不新建触觉/SnackBar/弹层/断点第二来源。

## 依赖与启动门禁

本 leaf 必须等以下前置合入 `main` 后 rebaseline（记录新 SHA、复核 owning
files 行号、跑聚焦测试记录真实基线），再启动第一个 stage：

- **W2 discovery-query-context**：`ranking_page.dart`、`novel_ranking_page.dart`、
  `recommended_home_page.dart`、`new_page.dart`、`search_result_page.dart`
  是 W2 工作面；本 leaf 只动 itemBuilder/组件调用行，排在 W2 合入后。
- **W4 artwork-viewer-series-flow**：§5.6 唯一触觉 owner（`lib/app/` 薄封装）
  由 W4 随下载/保存建立；本 leaf 只消费，不另建来源。
- **W5 novel-reader-parity**：`NovelEntry` 的 progress 槽位消费 W5 的真实
  阅读进度语义——`null`（无进度记录，不渲染进度）与 `0.0`（从头开始，渲染
  空进度条）必须可区分；更新游标（watchlist `latest_content_id`、本地书库
  `readOffset`）不是阅读进度，UI 文案不得混称（父 §6）。

规模：本包是路线图最大工作包之一，按 `research/risks.md` §1 预拆三个 stage
分支（objects → management → downloads），见 `implement.md`。

## Requirements

### R1. 小说条目单契约：`EntityRow` + `NovelEntry`

现状：`NovelCard`（`lib/app/widgets/novel_card.dart`，68×88 r6 封面、无
PressScale、无字数、无尾标、无 Semantics）与 `NovelRow`
（`lib/app/widgets/novel_row.dart`，56×72 r4 封面、有 PressScale/字数/
chevron、占位图直角瑕疵 75-83 行）两份平行实现并存，共 5 个调用点
（`recommended_home_page.dart:362`、`novel_ranking_page.dart:183`、
`search_result_page.dart:247`、`new_page.dart:382`、
`profile_novel_feed.dart:96`）。

- 新增 `EntityRow` 槽位基件（`lib/app/widgets/`）：leading 封面、title、
  subtitle 作者、meta 语义元信息、badge 覆盖徽标、progress 进度条、
  trailing 尾部动作、`selected` 选中态、onTap/onLongPress、semanticLabel。
  识别与呈现固定在基件；排名/日期/进度/更新状态一律是槽位或命名 variant，
  页面不得自绘（父 §5.2）。
- 新增 `NovelEntry` 类型适配，命名 variant：`compact`（56×72 r4）、
  `regular`（68×88 r6）、`ranking(rank:)`（名次徽标）；默认 key
  `ValueKey('novel-<id>')`；默认动作 `openNovel`；统一 `PressScale` +
  `Semantics(label)`；占位图走 `ClipRRect`（修 NovelRow 直角瑕疵）。
- 5 个调用点按 variant 迁移后，`NovelCard`/`NovelRow` 由本 leaf 明确删除
  （父 §8：删除由 owning leaf 覆盖）。
- Given 任一调用点渲染小说条目，When 用户点击，Then 打开对应
  `openNovel` 详情——迁移前后主动作不丢（门禁项）。

### R2. 插画卡片：只统一徽标容器

现状：`IllustCard`（`lib/app/widgets/feed/illust_card.dart`）四角徽标容器
不统一——R-18 左上 `Card+primary`（260-274）、ugoira 左下 `Container r5`
（275-290）、页数右上 `Container r7.5`（291-308）、AI 右下 `Card+error`
（309-323）。

- 抽取统一徽标容器（共享圆角/底色/内边距一档），四角定位不变，语义色由
  参数注入（R-18=primary、AI=error、ugoira/页数=中性黑纱）。
- 新增可选 `rank` 参数：左上排名徽标（与 R-18 垂直错开）。
- **不统一裁切**：`BoxFit.fitWidth` 瀑布流契约保留（157-159 行不动）。
- Given 任意徽标组合，When 卡片渲染，Then 所有徽标共享同一容器样式、
  语义色可辨、可访问标签保留 `title, author` 语义。

### R3. 排名变体接入排行页

现状：`ranking_page.dart` itemBuilder（228-231）与
`novel_ranking_page.dart`（178-184）均无排名数字契约。

- `ranking_page.dart` 传 `IllustCard(rank: index + 1)`；
  `novel_ranking_page.dart` 换 `NovelEntry.ranking(rank: index + 1)`。
- 只改 itemBuilder/徽标槽；TabBar、RootSwipeSwitcher、novel 榜 body 切换
  结构归 W2，本 leaf 不碰。

### R4. 历史页：按日期/类型找回 + 删除进入明确管理动作

现状（`lib/features/history/history_page.dart`）：长按=直接进删除确认
（239-243），无选择模式；`_KnownIllustEntry` 用 `MediaQuery` 固定两列估高
（278-283）与 `FeedItemExtent` 脱钩；小说条目是第三种封面比例
（`AspectRatio(1)`，437-458）；`_HistoryCardFrame` 自绘日期行（374-399）；
空态用 `ReplicaEmptyState` 而非共享 `FeedEmpty`。

- 找回能力：illust 条目改 `IllustCard` + 日期 meta 槽；novel 条目保持
  grid 方形 cell（决策 D4），按对象契约对齐圆角/PressScale/Semantics/
  类型角标；日期行升级为共享 meta 呈现。`_KnownIllustEntry` 改读
  `FeedItemExtent.maybeOf`（修宽屏估高错误）。
- 管理动作：长按进入选择模式（M3 contextual action bar：AppBar 换
  "已选 N"+删除+全选+退出；选中条目整行变色+check，模式下不挂次级动作）；
  删除走 `showAppBottomSheet` 确认（复用 `_confirmDelete` 460-500 形态）；
  "删除全部"保留在 AppBar。
- 空态收敛为 `FeedEmpty`（决策 D6）。
- Given 历史列表，When 长按条目，Then 进入选择模式而非弹删除确认——
  旧行为→新行为迁移说明见下节（肌肉记忆变更，父 implement.md §4-7 要求）。

### R5. 稍后再看：移除 + 撤销

现状（`lib/features/watchlater/watchlater_page.dart`）：长按 →
`showCardActionSheet` → `_WatchLaterAction`（`illust_card_actions.dart:103-148`）
`store.remove` 后无 SnackBar、无撤销。

- 移除保持"卡片长按 → 动作 sheet"路径；移除成功后
  `showAppSnackBar(..., action: 撤销)` 恢复条目。
- **撤销必须回到原序位置**：列表按 `added_at DESC` 排序
  （`watch_later_repository.dart:38`），而 `store.add()` 会刷新时间戳
  （`watch_later_store.dart` 注释自承 "re-adding refreshes the timestamp"）——
  直接重加会顶到最前，不构成"撤销"。方案：repository 增
  `add(accountId, entity, {int? addedAt})` 形参（或 `restore(entry)`），
  撤销时携带被移除条目的原 `addedAt` 落库；这是与 R8 同级的最小
  core API 增量，不改 add/remove 现有语义。
- 不扩成收藏体系：不加标签/筛选/多选（父 §4.6）。
- Given 已移除条目，When 点击"撤销"，Then 条目回到列表原序位置
  （测试断言恢复后 `addedAt` 与移除前一致）。

### R6. 追更：区分查看更新 / 打开目录 / 继续阅读 / 取消追更

现状（`lib/features/watchlist/watchlist_page.dart`）：`_WatchlistEntryTile`
（147-238）单一 onTap 承载两种语义且 manga/novel 不一致（manga 进目录
`openIllustSeries`、novel 进正文 `openNovel`）；无"取消追更"入口；
无目录动作区分。

- 条目改 `EntityRow` 承载（48×48 封面、"New"徽标→badge 槽、
  `user · date · count`→meta）；trailing `more_vert` → `showAppBottomSheet`
  动作清单。
- 动作语义按**三态分设**（与 W4 对齐，禁止混用）：
  - **查看更新**（tap 主动作）：`latestContentId` 游标驱动——
    manga→`openIllust(latest)`、novel→`openNovel(latest)`；
  - **打开目录**：manga→`openIllustSeries(id)`；novel 本期降级不提供
    （决策 D3：无 novel 系列目录路由）；
  - **返回最近打开**：消费 W4 的 `series_recent_open_store` 内存记录
    （有记录才渲染，文案「返回第 n 话」/「返回上次作品」）；
  - **取消追更**→`watchlistActionsProvider.toggle(key)`
    （`watchlist_toggle.dart:54`）。
- **禁止把 `seenId` 当"继续阅读"**：`markSeen` 只前进不回退
  （`watchlist_store.dart:252-279`）——先开第 20 话再回看第 5 话，
  游标仍指 20，它不是"最后停在哪"。`seenId` 只驱动新内容标识；
  漫画无持久阅读锚点，本轮不做持久化进度——无记录时显示
  「开始阅读/查看目录」，不强行让游标兼职。novel 可消费真实
  `readOffset` 锚点时才叫「继续阅读」。
- 打开仍 `markSeen` 更新游标，但**只算更新游标**，UI 文案不得称其为
  阅读进度（父 §6；W5 进度语义边界）。

### R7. 本地书库：主操作=继续阅读，删除进入更多

现状（`lib/features/localnovel/local_novels_page.dart`）：`_LocalNovelTile`
trailing 是 delete `IconButton`（103-107）；删除确认用**裸 `showDialog`**
（111-128），未走 `showAppDialog`。

- tile tap → `openLocalNovelReader`（现状保留）；`readOffset != null` 时
  meta 行显示进度文案（`local_novel_repository.dart:25,39`；null↔0 边界
  见依赖节）。
- 删除下沉：trailing 改 `more_vert` → `showAppBottomSheet` 菜单（含删除）；
  确认弹层改 `showAppDialog`（修裸 showDialog）。
- 条目视觉向 `EntityRow` 靠拢（leading 书封/icon 槽）。

### R8. 下载任务页：父子层级 + 九态动作映射 + 查看动词 + 提交反馈直达

现状（`lib/features/settings/pages/download_tasks_page.dart`）：groups 与
tasks 平铺（51-63），属组子任务仍出现在顶层；`retryable+paused` 映射为
refresh 图标+`retryDownload` 文案（§5.7 冲突）；`succeeded` 仅静态 ✓ icon
无"查看"；无终态移除出口；三处提交点 SnackBar 无 action。

- 父子层级：子任务按 `snapshot.groupId` 聚合缩进在组卡之下
  （`download_task.dart:86`）；未入组任务平铺；组卡头=组状态+聚合进度+
  组级动作。
- 九态→动作映射表（设计定稿见 design.md §二）：
  `retryable+paused`→**继续**（resume，非 retry 图标）；`failed/canceled`→
  重试/重新下载；`succeeded`→**查看作品**（`openIllust(task.illustId)`——
  打开的是作品页而非已下载文件，文案必须如实；「打开文件/保存位置」
  在有本地文件打开能力前不提供）；`orphaned`→说明文案+移除；
  终态条目给"移除"出口。
- **核心层增量声明**：`DownloadManager` 新增终态移除 API
  （当前 `manager:108-109` 无 dismiss/clear，终态任务永久驻留 `_jobs`）。
  这是 UI 驱动的最小 core API 新增——只删终态快照记录，不改
  pause/cancel/retry/recover 语义（决策 D2，父 §6 允许的最小必要边界）。
- 提交反馈直达：三处 `downloadQueuedMessage` SnackBar 加 `action: 查看` →
  `/settings/tasks`（`illust_card_actions.dart:90`、`page_image.dart:241`、
  `user_page.dart:268`）；经 `lib/app/navigation/routes.dart` 新增
  `openDownloadTasks` 门面（分层约束：app/widgets 不能 import
  features/settings 的 helper，决策 D10）。
- `download_tasks_page_test.dart`（203-254）固化了"paused→refresh 图标+
  retry"旧行为，必须与映射表同步改（quality-guidelines：删 guard 同步删
  固化测试）。

### R9. 管理列表限宽 + 触觉消费 + 术语

- 限宽（父 §5.5，management 角色"信息与动作聚合"）：`AppBreakpoints`
  扩展 `managementContentMaxWidth`（≈840dp），ListView 型管理页
  （watchlist/localnovels/download_tasks）body 包 Center+ConstrainedBox；
  grid 型（history/watchlater）不限宽（瀑布流自适应，限宽降密度）。
  不得自造宽度常量。
- 触觉（父 §5.6，消费 W4 owner）：进入选择/管理模式=明确震动；
  模式内勾选=轻触；危险确认弹出=明确震动；移除+撤销成功=轻触或不加；
  普通条目点击不加。W4 未合入时触觉 checkbox 显式等待，不建临时封装。
- 术语（父 §5.7 种子已冻结，直接消费）：删除（不可恢复，有确认）/
  移除（列表条目，可撤销）/取消（仅终止当前操作）/重试 vs 继续/
  查看（下载完成看结果、提交反馈直达）。新增 l10n 键清单见 design.md。

## 行为迁移说明（旧 → 新）

| 页面 | 旧行为 | 新行为 | 迁移说明 |
|---|---|---|---|
| 历史 | 长按条目 = 直接弹删除确认 | 长按条目 = 进入选择模式，删除收进 AppBar 动作 | 用户旧肌肉记忆"长按删单条"改为"长按开始多选"；单条删除仍可达（选择该条→删除），删除前确认弹层不变 |
| 本地书库 | 条目尾部 delete 图标直删 | 尾部 more_vert 菜单含删除 | 主操作让给继续阅读；删除多一步菜单，确认弹层不变 |
| 下载任务 | 暂停任务显示"重试"（refresh 图标） | 暂停任务显示"继续"（play/resume 语义） | 文案与真实后果对齐：paused 续传保留 resumeAnchor，不是重试 |
| 追更 | 点条目 manga 进目录 / novel 进正文（不一致） | 点条目一律"查看更新"（进最新内容）；目录/继续阅读/取消追更收进 more_vert | 统一 tap=查看更新；manga 目录经菜单到达 |
| 稍后再看 | 移除后无反馈 | 移除后 SnackBar + 撤销 | 纯增量，无旧路径破坏 |

## Acceptance Criteria

- [ ] `NovelEntry` compact/regular/ranking variant 各有 widget 测试；
      5 个调用点迁移后点击主动作（openNovel）断言不丢；
      `novel_card.dart`/`novel_row.dart` 删除。
- [ ] `IllustCard` 徽标共享容器 + `rank` 徽标测试；`BoxFit.fitWidth`
      与四角定位回归不变。
- [ ] 排行两页 itemBuilder 接入 rank；榜单结构零改动。
- [ ] 历史页：选择模式进入/勾选/全选/删除确认/退出全链路测试；
      `_KnownIllustEntry` 读 `FeedItemExtent`；长按旧语义用例同步改写。
- [ ] 稍后再看：移除→SnackBar action→store 复原测试。
- [ ] 追更：三动作 + 取消追更语义测试；manga/novel 分支各覆盖。
- [ ] 本地书库：tap=继续阅读、删除经菜单+`showAppDialog` 确认测试。
- [ ] 下载页：九态 + `failureKind=paused` 动作映射测试全覆盖；
      `download_tasks_page_test.dart` 旧断言同 commit 改写；
      `dismiss`/`clearTerminal` core 单测；父子聚合渲染测试。
- [ ] 三处提交 SnackBar "查看" action 断言导航至 `/settings/tasks`。
- [ ] 限宽：840/1200dp 下 ListView 管理页限宽断言；320dp 无不可达动作。
- [ ] 触觉调用点全部走 W4 owner（`lib/` 无 `HapticFeedback` 直调）；
      分级符合 §5.6。
- [ ] 每个 implement.md checkbox 一个提交；三 stage 分支
      `task/09-22-entity-management-consistency-{objects,management,downloads}`
      串行合入。
- [ ] `flutter analyze --no-pub`、相关 `flutter test`、`git diff --check`、
      `task.py validate` 全绿；运行时矩阵未覆盖项在 PR 标"未验证"。

## Decisions（规划定案）

- D1 `EntityRow` 泛型槽位基件 + `NovelEntry` 类型适配（非纯 NovelEntry）：
  本包内 watchlist/localnovels/history/download 四页是 ≥3 真实消费者，
  满足 §5.5 共享门槛。
- D2 `DownloadManager` 新增 `dismiss(taskId)`/`clearTerminal()`：UI 驱动的
  core 最小增量，只移除终态快照记录，不改下载语义。这是本包唯一的
  核心层改动，在此显式声明（父 §6）。
- D3 watchlist novel "打开目录"本期降级不提供：路由表只有
  `series/:seriesId`→`IllustSeriesPage`（routes.dart:476-481），
  `_NovelSeriesBar` 是 prev/next 条非目录页；菜单项按实体类型条件呈现，
  novel 项不显示该动作。新建小说系列目录页另立任务。
- D4 历史页 novel 条目保持 grid 方形 cell，不新增 `NovelEntry` 第三密度；
  按对象契约对齐圆角/PressScale/Semantics/类型角标，日期走共享 meta
  呈现。`IllustCard` 增 `meta` 可选槽承载历史日期。
- D5 管理模式交互=长按进入选择模式（M3 标准）；仅 history 与
  download_tasks 建选择模式（多删/批量移除是真实需求）；watchlater 保持
  长按动作 sheet、localnovels 用条目 more_vert 菜单（单条管理动作，
  无多选必要）。
- D6 history 空态从 `ReplicaEmptyState` 收敛到共享 `FeedEmpty`；
  `ReplicaEmptyState` 组件本体不删（comments 等其它消费点归各自 owner）。
- D7 小说长按动作菜单（`CardAction` 泛化为 `CardAction<T>`）本期不做，
  记入 Out of scope——`card_action.dart:24` 签名绑定 `IllustEntity`，
  泛化属额外扩张。
- D8 限宽 owner 形态：`AppBreakpoints` 增 `managementContentMaxWidth`
  静态量（840dp），页面包 `Center + ConstrainedBox`；只在 ListView 型
  管理页接入。
- D9 `_UserRow`（recommended_home_page.dart:399-429）可消费 `EntityRow`
  但属 W2 页面范围：本 leaf 只提供基件，迁移与否由 W2 决定，不列入
  本包范围。
- D10 新增路由门面 `openDownloadTasks(BuildContext)`（routes.dart，
  `context.push('/settings/tasks')`）：`illust_card_actions.dart` 位于
  `app/widgets/`，分层规则禁止 import `features/settings/settings_helpers.dart`
  的 `openSettingsPage`；`page_image.dart`/`user_page.dart` 跨 feature
  同理须走门面。

## Out of scope

- 榜单 TabBar/切换结构、发现页层级（W2）；作者页改造（W3，仅让出
  profile_novel_feed 一行组件调用）；详情/查看器/系列（W4）；阅读器
  舞台与真实阅读进度（W5，本包只留 progress 槽位）；设置信息架构（W8）。
- `CardAction<T>` 泛化与小说长按动作集（D7）；novel 系列目录页新建（D3）；
  watchlater 扩成收藏体系；`ReplicaEmptyState` 组件删除（D6）；
  图片裁切统一；下载状态过滤 chips/排序（可选增强，按需裁剪）。
- 不新建 SnackBar/弹层/刷新/断点/触觉第二来源（risks.md §5 已验证清单）。
