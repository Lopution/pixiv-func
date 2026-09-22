# 技术设计：发现页查询上下文连续（W2）

设计基线：`main@8067b2d`（前置 W1 合入后需 rebase 到当时的 `main`）。
逐页字段归属与落点见 `research/implementation-draft.md`（§1 每页 ownership 表、
§2 re-tap 落点、§3 恢复等级、§4 父 §4.2 条目映射）；逐项现状核实见
`research/codebase-*.md`；决策与协调风险见 `research/risks.md`；框架分层依据见
`research/external-flutter.md`，参照实现见 `research/external-pixez.md`（re-tap
广播）与 `research/external-shaft.md`（共享可编辑查询头）。本文件只固定结构、
契约与边界。

## 一、阶段划分与依赖

默认单分支 `task/09-22-discovery-query-context`，四个阶段串行合入（同分支顺序
提交，每阶段收尾跑验证命令）。若 PR 体量不可审查，按 implement.md 阶段切
`…-s1`/`…-s2` 分支。

| 阶段 | 条目 | 文件面 | 风险 | 前置 |
|---|---|---|---|---|
| 1 | R10 re-tap 通道 + R2 插画排行 + R3 小说排行 | `branch_slide_stack.dart`、`ranking_page.dart`、`novel_ranking_page.dart`、`routes.dart` | 中（新通道语义、TabSlideStack 收敛） | W1 合入 |
| 2 | R1 推荐 + R4 新作 | `recommended_home_page.dart`、`new_page.dart`、`routes.dart` | 中（branch 根路由参数化、TabController 防护） | 阶段 1（通道） |
| 3 | R7 序列化 + R5 结果头 + R6 建议 + R8 搜索首页 | `routes.dart`、`search_result_page.dart`、`search_page.dart`、`search_models.dart`（只读参照） | 中（URL 契约变更、cacheKey 身份） | 阶段 1；W1 R4 已改同文件 |
| 4 | R9 反向搜图任务头 | `reverse_image_search_page.dart` | 低-中（纯 UI 层叠加） | W1 R8 cancel 语义已合入 |

依赖说明：

- R10 通道必须先于一切消费者落地（阶段 1）；W3 后续在同通道消费，不得另建。
- `routes.dart` 在阶段 1/2/3 都被触碰——同 leaf 内串行无冲突；与其他并行 leaf
  （W3/W4…）共享该文件时按合入顺序串行（risks.md §Ownership）。
- 阶段 3/4 触碰 `search_page.dart`、`reverse_image_search_page.dart`，与 W1 的
  编辑区有交集（:153-155、:117-120,:297），必须基于含 W1 的 `main`。

## 二、关键契约

### ReTapChannel（共享通道，本 leaf 创建并冻结形态）

- **发射**：`BranchSlidePager` 持有 `reTapEvents`——**必须是事件而非选中状态**：
  `ValueNotifier<int>` 装裸 branch index 时同值重赋不通知（`_value == newValue`
  直接 return），连续同槽点按第二次起丢信号。负载用递增序号
  `({int branchIndex, int sequence})` 或等价 `Stream` 广播；仍由 pager 持有，
  不新增全局事件总线（PixEz `topStore` 同构）。只在
  `selectIndex` 的 `index == tab.index` 显式点击分支、`goBranch` 之后发射；
  `syncIndex`、拖拽 settle、`_suppressGoBranch` 路径绝不发射
  （`branch_slide_stack.dart:137-172`）。
- **时序**：`goBranch` 先把分支栈弹回根，信号须 post-frame 消费，使滚动落在
  已可见的根页上（risks.md 手势排序项）。
- **消费**：根页在 `initState` 后经 `BranchSlideStack.maybeOf(context)` 订阅
  （pager 已是 `ChangeNotifier`/`InheritedWidget` 可达，:236）；回调里取
  「当前可见」滚动体的显式 controller `animateTo(0)`。页内
  TabBar/chip 同索引点击走同一 helper，不经广播。
- **语义**：纯滚动回顶——`MotionTokens.enabled/resolve` gated 动画，
  reduced-motion → `jumpTo(0)`；不 refresh、不展开收起、不改选中（父 §5.1）。
- **controller 所有权**：凡被 re-tap 消费的 feed 必须有显式 per-key
  `ScrollController` 传入 `SmoothWheelScroll`——桌面端它的私有 `_controller`
  屏蔽 `PrimaryScrollController`（`smooth_wheel_scroll.dart:160-177`），
  环境依赖的 `PrimaryScrollController.of` 会静默 no-op。Offstage/`TabSlideStack`
  隐藏 body 的 controller 存活但不可见：消费端 `hasClients`+`mounted` 守卫。
- **移交**：通道形态（发射点、负载类型、消费时机）由本 PR 冻结进
  `branch_slide_stack.dart` 注释与本节；W3 作者页 re-tap 消费它，不得另起
  通道或改负载语义。

### QueryContext 字段 → owner → 恢复等级矩阵

恢复等级三级口径与 spec「Route Restoration Contract」
（component-guidelines.md）及 state-management.md「Route/restoration」行对齐：
内存（PageStorage/provider family）、路由 durable（typed query param）、
进程死亡（Flutter restoration + 路由 durable 值 + durable store）。

| 表面 | 内存 | Flutter restoration | 路由 durable | 进程死亡后 |
|---|---|---|---|---|
| 推荐 type | — | — | `?type=`（新） | type + 滚动 |
| 插画排行 mode | — | — | `?mode=`（已有） | mode + 滚动 |
| 小说排行 mode | — | — | `?mode=`（新） | mode + 滚动 |
| 新作 scope/type | — | — | `?scope=&type=`（新） | scope/type + 滚动 |
| 搜索输入草稿 | 光标/selection（声明 memory-only，D8） | — | `q`/`type` | q/type |
| 搜索结果查询 | — | — | 全 13 字段（R7 补齐） | 查询 + 滚动 |
| 搜索 trending kind | 会话 | — | — | 回 illust（声明） |
| 反向搜图会话 | 输入/阶段/结果/webView | — | — | 仅引擎；流程复位（声明，temp-file 绑定） |
| 筛选默认值、引擎 | — | — | — | `SettingsRepository`（已有） |

滚动 offset 的覆盖不变：各 feed 已有 `PageStorageKey`+`restorationId`（内存
切换 + Flutter 恢复两层），本包只补显式 controller 供 re-tap 寻址，不改 key。

### 路由参数契约

- 新参数：`/recommended?type=`、`/new?scope=&type=`、`/novel-ranking?mode=`；
  值 = 枚举 `name`/wire 名，宽松解码落默认。
- facade：`replaceRecommendedType`/`replaceNewFeed`/`replaceNovelRankingMode`
  全部 `context.replace`（不产生新栈项，对齐 `replaceRankingMode` 先例）。
- `context.replace` 重建不得重置 `TabController`：`initialIndex` 播种 + guarded
  sync（D13）；route 值与 provider family key 必须 1:1，不允许「路由说 A、
  widget state 说 B」（risks.md 陈旧 family 项）。
- 搜索筛选补齐 13 字段（`ai/bmin/bmax/ratio/ct/wmin/wmax/hmin/hmax`）；
  `SearchQuery.cacheKey` 已含全字段（search_models.dart:168-181），URL 完整化后
  恢复前后 cacheKey 一致，PageStorage/provider 身份不漂移。
- restoration bucket 只写标量/短串：`q` 写入时裁剪限长（risks.md 体积项）。

### 选择器可见性契约（QueryContext 最小 UI，父 §5.1）

- 主范围/类型、细分模式、修改/清除入口在 loading/error/empty 中始终可见
  （现状已有好先例：各页选择器挂 AppBar 而非 body）。
- 新作 type 选择器常驻（删 `_selectorExpanded`）；搜索结果头可编辑
  （tap → `openSearchInput` 预填）+ 筛选摘要 + 清除；空结果有「修改搜索」。
- 标签文字不缩字：移除 `FittedBox(scaleDown)`，改 scrollable TabBar（D10）。

## 三、复用与禁止

- 复用：`RootSwipeSwitcher`/`TabSlideStack`（排行先例）、`SmoothWheelScroll`
  （扩展传入 controller，不改它内部）、`feed_states`（`FeedEmpty`/`FeedError`
  加动作参数，不 fork）、`PullToRefresh`、`MotionTokens`、`BranchSlideStack.maybeOf`、
  `openSearchInput`/`replaceSearchResults`/`replaceRankingMode` 既有 facade、
  #48 共享 SnackBar 入口、现有 l10n 工具链。
- 禁止：新 provider family/第二个全局广播总线（通道挂 pager）、
  `PrimaryScrollController.of` 做 re-tap、对象条目组件（W6）、
  `RestorationMixin` 铺开（本包声明等级即可，不为草稿光标引入 restorable
  controller）、`func_bottom_nav.dart`/`root_swipe_switcher.dart` 改动
  （同索引事件已在 `selectIndex` 内可辨，无需调用侧配合）、网络/数据层语义。

## 四、测试策略

- 通道：fake pager/`selectIndex` 同索引 → 信号一次；`syncIndex`/拖拽路径 → 不发射；
  消费端断言终态 `offset == 0` 且无 refresh 调用（钉「终态而非轨迹」）。
- 页内 re-tap：排行/小说排行/新作同索引点击 → 当前 key controller 回顶；
  隐藏 Offstage body 不滚。
- 路由：新参数 round-trip（含进程死亡等价的 restoration 用例，参照
  `test/navigation_restoration_test.dart` 既有 ranking/search 模式）；搜索 13
  字段 URL↔`SearchQuery`↔`cacheKey` 一致；非法值宽松解码。
- 交互：建议项 tap=填入不提交、trailing=提交；结果头 tap→预填输入页；
  空态「修改搜索」可达；新作品 type 控件常驻且同索引点击回顶；`TabController`
  在 `context.replace` 下不重置。
- 反向搜图：任务头在 ready/failure/empty/webView 各阶段渲染同一图/引擎；
  `InAppWebView` 平台 fake 按 quality-guidelines 既有模式。
- 删旧：固化「re-tap 展开」「tag 凑整丢弃」的测试随修复同 commit 删改。
- 新/改回归测试须先对未修代码验证可失败（quality-guidelines 回归测试规则）。

## 五、运行时证据缺口（PR 标「未验证」）

桌面滚轮下 re-tap 真实链路（`SmoothWheelScroll` 桌面私有 controller 已静态
确认，但滚动手感需桌面验证）；Android 进程死亡恢复真机链路（`restorationId` +
路由参数联合恢复）；`goBranch` 弹栈动画与 re-tap 到达的相对时序真机确认；
1.3x 大字号/长翻译下推荐/新作 scrollable TabBar 与新作常驻 type 行的观感；
屏幕阅读器对可编辑查询头/任务头的朗读路径（TalkBack/Narrator）。
