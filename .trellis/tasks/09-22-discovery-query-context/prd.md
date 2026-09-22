# 发现页查询上下文连续（Roadmap W2）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图）。
本 leaf 拥有父 `design.md` §4.2 列出的范围，落地 §5.1 QueryContext 契约。
规划基线：`main@8067b2d`。全部行号断言已由本 leaf `research/` 在当前 HEAD
逐条核实（`research/codebase-*.md`），字段归属与落点见 `research/implementation-draft.md`。

## Goal

让推荐、排行、新作、搜索（首页/输入/结果）与反向搜图的「当前看什么」——范围、
内容类型、榜单模式、查询与筛选——持续可见、可修改、可恢复；并把「重复点击当前
分类/标签」统一固定为回到顶部（父 §5.1：纯滚动、不附带刷新、不承载展开入口）。

本 leaf 只收敛查询上下文与交互规则：不自建对象条目组件（W6）、不做布局/信息架构
重构、不改网络与数据层语义。

**前置**：W1（`09-22-interaction-outcome-correctness`）必须先合入 `main`——它改
`search_page.dart` 热门标签 itemCount（R4）与 `reverse_image_search_page.dart`
取消语义（R8），本包在同文件上继续工作；start 前 rebase 到含 W1 的 `main`。

## Requirements

### R1. 推荐页类型上下文路由化 + 标签可读性 + 刷新错误可见

现状：`_type` 只存于页面 `State`（`recommended_home_page.dart:47`），`/recommended`
路由无 query 参数（routes.dart:963-972），进程死亡回到 illust；TabBar 标签包
`FittedBox(scaleDown)`（:158-164，Astra 行 1 确认的无限缩字）；刷新失败错误只追加在
列表尾部 `FeedTail`（:264-282）；feed 无显式 `ScrollController`（:302-308 仅有
PageStorageKey/restorationId），桌面端 `PrimaryScrollController` 被
`SmoothWheelScroll` 私有 controller 屏蔽（smooth_wheel_scroll.dart:160-177）。

- 新增 `/recommended?type=` 路由参数（值为 `RecommendedContentType` 名）与
  `replaceRecommendedType` facade（`context.replace`），对齐排行 `?mode=` 先例
  （routes.dart:977-980, 1249-1255）。
- `TabController` 由参数 `initialIndex` 播种，切换时 guarded sync 写路由；
  `context.replace` 重建不得重置进行中的 tab 手势（D13）。
- 删除 `FittedBox(scaleDown)`；四个类型标签改用与排行一致的 scrollable TabBar，
  大字号/长翻译下不缩字（D10）。
- 刷新失败（非首屏）错误经共享 SnackBar 通道（父 §5.6，#48 已建）在当前视野内
  可辨，不再只出现在列表尾部。
- 每类型一个显式 `ScrollController`（map 按 type keyed），传入 `SmoothWheelScroll`；
  供 R10 re-tap 与页内同标签点击回顶消费。

### R2. 插画排行同模式点击回顶

现状：`?mode=` 已 durable（`_handleTabChanged` → `replaceRankingMode`，
ranking_page.dart:70-78）；per-mode `ScrollController` map 已存在（:43,:80-82）；
`TabBar` 无 `onTap`——点当前 mode 无任何反馈（:104-115）。

- `TabBar.onTap` 同索引分支 → 当前 mode 的 controller `animateTo(0)`；
  异索引行为不变（切换 + `replaceRankingMode`）。
- `onTap` 内不得对 `TabController` 再调 `animateTo`（Tab Navigation Animation
  Contract）；回顶只作用于 feed 的 `ScrollController`。
- AppBar 小说榜入口图标（:97-102）保持现状不变。

### R3. 小说排行收敛同一套切换规则 + 模式路由化

现状：`/novel-ranking` 是 pushed 路由 `const NovelRankingPage()`，无 `?mode=`
（routes.dart:549-553）；mode 仅在内存，重建/进程死亡回到 `day`；body 用
`ValueKey(mode)` 整块替换（novel_ranking_page.dart:90-95），无 `RootSwipeSwitcher`/
`TabSlideStack`，横向拖动不切模式——与插画榜规则不一致（Astra 行 2）。

- 路由新增 `?mode=`（`NovelRankingMode` 名）+ `replaceNovelRankingMode` facade；
  `TabController` 由参数播种（同 D13 防护）。
- body 改用 `RootSwipeSwitcher + TabSlideStack` + per-mode 常驻 body（复用现有
  `_scrollControllers` map :33,64-66 与 `novel-ranking-<mode>` keys :168,:172），
  与插画榜共享同一套点击/拖动规则。
- `TabBar.onTap` 同索引 → 当前 mode controller `animateTo(0)`。

### R4. 新作页范围/类型持续可见 + re-tap 改为回顶

现状：`_selectedIndex`(:37)/`_type`(:38)/`_selectorExpanded`(:39) 仅页面 `State`，
`/new` 无 query 参数（routes.dart:987-996）；`_onTabTap` 同索引切换
`_selectorExpanded` 展开类型选择器（:70-74，接在 :102）；类型选择器展开后随范围
切换/选择收起（:63-67,:79），"范围＋类型"非持续可见（Astra 行 3）；scope TabBar
同样 `FittedBox(scaleDown)`（:106-112）；per-key controller 已存在（:230,235,299），
Offstage 隐藏 body 的 controller 仍存活（:149-161）。

- 新增 `/new?scope=&type=` 路由参数 + replace facade（D11）；`_selectedIndex`/`_type`
  由路由驱动。
- 类型选择器改为常驻可见控件（AppBar 下 segmented/chips 行），删除
  `_selectorExpanded` 与 `_onTabTap` 展开逻辑。
- scope TabBar 移除 `FittedBox`，同 D10 不缩字。
- re-tap：scope TabBar 同索引点击与类型 chip 同索引点击都 → 当前 `NewFeedKey` 的
  controller `animateTo(0)`；只对可见 body 生效（`hasClients`+`mounted` 守卫，D2）。
- **旧行为 → 新行为迁移说明**：原版「重复点击当前范围标签」= 展开/收起隐藏的类型
  选择器；新版 = 回到顶部。类型入口由「隐藏、re-tap 才出现」变为「常驻可见」，
  可达性不降；展开手势的肌肉记忆由常驻控件替代，无功能丢失。

### R5. 搜索结果页可编辑查询头 + 筛选摘要 + 清除 + 空结果修改

现状：AppBar title 是只读 `Text`（search_result_page.dart:66-71），仅一个筛选
`IconButton`（:72-79）；无关键词编辑、无筛选摘要、无清除入口；空结果
`FeedEmpty` 只有重试原查询（:138-145）——Astra 行 5「部分」。

- 头部关键词成为可编辑入口：点击 → `openSearchInput(q, type)` 预填（push）；
  筛选摘要 chips 行 + 清除入口常驻；header 在 loading/error/empty 中保持可见。
- 空结果 `FeedEmpty` 新增「修改搜索」动作 → 同一 `openSearchInput` 预填入口（D7）。

### R6. 搜索建议项区分「填入」与「立即搜索」

现状：`_selectSuggestion`（search_page.dart:419-424）点击建议行 = 填入并立即
`_submit()`，无「只填入」路径（Astra 行 5 / 父 §4.2）。

- 行点击 = 只填入（写 `TextEditingController`，不提交）；行尾动作 = 立即搜索。
- 两个动作有可区分的可访问标签（icon tooltip / 语义标签）。

### R7. 搜索筛选 URL 序列化补齐 13 字段

现状：`_searchFilters`/`_searchQueryParameters`（routes.dart:295-347）只序列化
`target|sort|duration|start|end` 5 个字段；`SearchFilters` 共 13 字段
（search_models.dart:120-160），`aiFilter`/`bookmarkMin`/`bookmarkMax`/`ratio`/
`contentType`/`widthMin`/`widthMax`/`heightMin`/`heightMax` 在 URL 中静默丢失——
深链/进程死亡恢复后查询失真，且解码出的 `cacheKey` 与原值分叉（PageStorage +
provider family 身份同时漂移）。

- 补齐全部 13 字段的解析与序列化（参数名见 D5）；解码保持宽松
  （`firstWhere orElse` 现状语义），非法值落默认值。
- round-trip 契约：URL → `SearchQuery` → `cacheKey` 在恢复前后一致；
  `PageStorageKey`/`restorationId`（'search-<cacheKey>'）身份不漂移。

### R8. 搜索首页热门标签层级收敛 + trending 上下文声明

现状：`itemCount: tags.length - tags.length % 3` 丢末行标签
（search_page.dart:153-155，**W1 R4 修内容丢弃本体**）；固定三列网格；
`trendingKindProvider` 是内存态（search_trending_controller.dart:11-23）；
`CustomScrollView` 无显式 controller（:37-40）。

- 在 W1 itemCount 修复之上收敛层级：全部标签都渲染，布局改为宽度自适应
  （D9：自适应网格，保留瓷砖代表图）；不再存在"为凑整行"概念。
- `trendingKind` 声明为会话内存态（进程死亡回 illust），`trendingTagsProvider`
  非 autoDispose 的有意设计写进恢复矩阵。
- 补显式 `ScrollController`，接 R10 branch re-tap 回顶。
- **边界**：本 leaf 不动 itemCount 修复本体（W1 所有），只做其上的 Wrap/层级收敛。

### R9. 反向搜图常驻任务头

现状：`_body` 按阶段整块替换（reverse_image_search_page.dart:177-202）；失败态无
图片预览（:370-426）；空结果是裸 `Center(Text)`（:429-431）；会话
（输入图/阶段/结果/webView/engineFailures）挂在 autoDispose family 的页级
`Session` 上（reverse_image_controller.dart:128-133），纯内存。

- 新增常驻任务头（缩略图 + 引擎 chip + 阶段标识），在 ready/searching/failure/
  success-empty/结果列表/WebView 各阶段保持可见；引擎 chip 在允许切换的阶段
  保持可交互。
- 头部不得比流程本身更久地持有临时文件（webUpload 语义 :332-339 不破）。
- 消费 W1 修正后的 cancel 语义（`stopSearch` 原地停搜、AppBar back 离页）；
  `Session` 保持在 `initState` 创建、不在 `build` 重建（risks.md 生命周期项）。
- 进程死亡恢复输入图**不做**：temp file 是 session-owned，声明为 memory-only（D8）。

### R10. 共享 re-tap 回顶通道（branch 级，本 leaf 创建、W3 消费）

现状：`BranchSlidePager.selectIndex` 同索引分支只 `shell.goBranch(index)` 弹栈回根
（branch_slide_stack.dart:137-151）——同槽点按从未产生滚动语义。

- 在 pager 上新增广播信号（`ValueNotifier<int>`/`Stream<int>`，PixEz `topStore`
  同构，D1），**仅**从 `selectIndex` 的显式同索引点击路径发射；
  `syncIndex`/拖拽 settle/`_suppressGoBranch`（:156-172）不发射。
- 根页经 `BranchSlideStack.maybeOf(context)` 订阅，post-frame 消费
  （`goBranch` 先弹栈，信号须在弹栈动画提交后到达可见根页）。
- 消费 = 当前可见滚动体 `animateTo(0)`：`MotionTokens` gated 动画，
  reduced-motion → `jumpTo(0)`；纯滚动——不 refresh、不展开、不切换（D3）。
- 该通道是全 app 共享契约：W3 的作者页等页面在同通道上消费，本包冻结其形态。

## Acceptance Criteria

- [ ] R1–R10 每项 ≥1 个 focused test（新增或更新），受影响的既有测试同步修正。
- [ ] re-tap 双通道测试：branch 级（同槽点按 → 弹栈后根页可见滚动体回到 offset 0）
      与页内 TabBar/chip 同索引点击；断言终态 offset=0 且**不触发 refresh**。
- [ ] 路由 round-trip 测试：`?type=`(recommended)、`?scope=&type=`(new)、
      `?mode=`(novel-ranking) 写读一致；搜索 13 字段序列化→解码→`cacheKey` 一致。
- [ ] `context.replace` 重建下 `TabController` 不被重置（排行/推荐/新作各一例或
      共享断言）；re-tap 信号在 `syncIndex`/拖拽路径不发射。
- [ ] design.md 恢复等级矩阵逐页声明，且实现与声明一致；进程死亡项无法本机验证的
      在 PR 标「未验证」。
- [ ] New 页固化「re-tap 展开」旧行为的测试随删除同 commit 移除/改写（不留 `skip:`）。
- [ ] 父 implement.md §6「W2/W3」门禁逐条过：跳转前后范围/类型/筛选一致；
      控件在 loading/error/empty 可见；大字号/长翻译不靠缩字；排行点击+滑动回归；
      re-tap=回顶不承载展开。
- [ ] 新 l10n key 四语齐备（`app_{en,ja,ru,zh}.arb` → `gen-l10n` → `gen_l10n_lookup.py`）。
- [ ] 每 commit 对应 implement.md 一个勾选框；分支 `task/09-22-discovery-query-context`。
- [ ] `flutter analyze --no-pub`、相关 `flutter test`、`git diff --check`、
      `task.py validate` 全绿；运行时缺口在 PR 显式标「未验证」。

## Decisions（规划定案）

- D1 re-tap 通道形态：`BranchSlidePager` 持有的广播信号（`reTapEvents`，
  `ValueNotifier<int>` 或等价 broadcast stream，PixEz `topStore` 同构）；只从
  `selectIndex` 同索引显式点击发射，不在 `syncIndex`/拖拽 settle 发射。
- D2 消费方式统一：显式 per-key `ScrollController` + `maybeOf` 订阅 + post-frame
  消费；**不**用 `PrimaryScrollController.of`（桌面被 `SmoothWheelScroll` 私有
  controller 屏蔽）；Offstage 隐藏 body 用 `hasClients`+`mounted` 守卫。
- D3 re-tap 语义：纯滚动回顶，`MotionTokens`-aware 动画，reduced-motion →
  `jumpTo(0)`；不刷新、不展开收起、不切换选择。
- D4 New 页类型选择器常驻可见（AppBar 下 segmented/chips 行）；`_selectorExpanded`
  与 `_onTabTap` 展开逻辑删除；旧 re-tap 展开行为废弃（迁移说明见 R4）。
- D5 搜索筛选参数名：`ai`、`bmin`、`bmax`、`ratio`、`ct`、`wmin`、`wmax`、
  `hmin`、`hmax`（对齐 `search_models.dart` 既有 wireValue 风格）；宽松解码；
  过渡期内残缺 URL 解码出不同 `cacheKey` → provider 重复条目，可接受、自愈，
  PR 注明。
- D6 建议项：行点击=填入，行尾动作=立即搜索；不做长按等隐藏手势。
- D7 结果页关键词编辑与空态「修改搜索」统一走 `openSearchInput(q, type)` 预填
  （push 输入页）；不做 inline 编辑框、不做 pop-to-edit——tag 搜索/深链/恢复栈
  下方不一定有输入页，统一 push 入口永远有效。
- D8 恢复等级矩阵按 design.md §二表格定案：输入页光标/selection 声明
  memory-only（不引入 `RestorableTextEditingController`）；trending kind 声明
  会话内存态；反向搜图会话 memory-only（temp file 绑定），进程死亡仅引擎持久。
- D9 热门标签：W1 修 itemCount 本体；本包在其上改自适应网格
  （`SliverGridDelegateWithMaxCrossAxisExtent`），保留瓷砖代表图，不切换为纯文字
  chips；全部标签渲染、无凑整丢弃概念。
- D10 推荐/新作 TabBar 移除 `FittedBox(scaleDown)`，改 `isScrollable: true`
  （排行 :104-115 先例）；大字号/长翻译下不缩字维持单行。
- D11 路由参数与 facade 命名：`?type=`(recommended)、`?scope=&type=`(new)、
  `?mode=`(novel-ranking)；facade 走 `context.replace`
  （`replaceRecommendedType`/`replaceNewFeed`/`replaceNovelRankingMode`，
  最终命名随实现与既有 facade 风格对齐）。
- D12 刷新错误提示经 #48 共享 SnackBar 通道，不新增平行反馈通道；
  `FeedTail` 尾部错误保留与否由实现定，但错误必须在当前视野可辨。
- D13 `context.replace` 路由重建防护：`TabController` 由 `initialIndex` 播种 +
  guarded sync，不在 replace 引发的重建中重置/重播种。
- D14 l10n 新 key 复用 feature 前缀（`search*`/`newFeed*`/`ranking*`/
  `recommended*`/`reverseImage*`），不建跨 leaf 新前缀；与其他 leaf 的 key
  冲突在本 PR 评审协调解决。

## Out of scope

- 排名/进度变体对象组件及排行页接入、`IllustCard`/`NovelRow`/`NovelCard` 收敛
  （W6，父 §4.6/§4.11；本包不新建条目组件）。
- 作者页、收藏等 W3 页面对 re-tap 通道的消费（通道契约由本包冻结并移交）。
- 热门标签 itemCount 丢弃修复本体、反向搜图 cancel 语义本体（W1，前置）。
- 反向搜图输入图的进程死亡恢复（声明 memory-only，temp file session-owned）。
- 搜索历史层、Spotlight 入口层级、`tag_search_page` 独立改版（W9/复用结果页契约）。
- 网络/数据层语义、provider family 结构变更、`RestorationMixin` 全量铺开。
- 其他页面的 QueryContext（作者页 W3、阅读器 W5 等）。
