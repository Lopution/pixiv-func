# 实现计划：全项目 UI 视觉与交互重构

本计划按可独立验证的阶段组织。每个阶段结束后都能运行聚焦检查并可单独回滚；阶段依赖写在这里，不能只依赖任务树顺序。全局视觉原则和样板链路已获用户确认；完整设计评审通过后才进入实现阶段。

## 阶段 0：基线与约束冻结

- [x] 前置修正已随首个 PR 合入 main（nav 栈本地化、底栏间距、Ugoira 拖拽、搜索内联建议），记为 merged prerequisites，不计入任何阶段完成度。
- [x] 记录当前 `flutter analyze --no-pub`、`flutter test`、golden 清单、关键页面截图和现有路由/恢复行为，以任务 HEAD 落盘为准（`baseline-2026-09-07` tag 仅历史参照）。
- [x] 盘点 `lib/app/theme`、shell、feed/detail/profile/settings 共享组件和重复变体（清单见 design.md §0），标注高风险业务消费者。
- [x] 建立视觉矩阵目录与命名规则；确认本阶段不触碰 API、OAuth、网络策略、下载恢复和阅读位置。
- [x] 提交：`docs(ui): capture redesign baseline and constraints`。

依赖：无。退出条件：基线命令与未执行项均有记录，用户数据/业务契约清单已确认。

## 阶段 1：设计 tokens 与基础组件

- [x] `FuncTokens` 保留为基础常量 owner；新增 `FuncSemanticTokens` `ThemeExtension`（`func_semantic_tokens.dart`，全部引用 FuncTokens 常量）+ `FuncSpacing`/`FuncShape` 常量族 + `MotionTokens.resolve` reduced-motion 收敛；`replicaTheme` 接线并把 `surfaceContainerHigh/Highest` 解开为 `surfaceRaised`，14 处占位填充改挂 `surfaceContainer`（`69e6f97`）。
- [x] 组件收敛：`FeedLoading` 统一替换 20+ 处 `Center(CircularProgressIndicator())`（`2dc31fe`）；`PageHeader`/`AppScaffold` 落到 `ReplicaScaffold`（扩展 actions/bottom/FAB，`7903e48`）；`FeedEmpty` 加 `detail` 槽位；`_ProfileStatusPage`、详情页 `_RestrictedView`/`_NotFoundView`/`_ErrorView`、推荐页 `_InitialErrorView`、下载任务空态全部收敛到共享件；按钮/反馈统一到主题化 M3 按钮 + `ReplicaButton`（登录/引导大 pill）+ `AppSnackBar`；TagChip/AuthorSummary 语义与触控测试在 `shared_widgets_test.dart`。
- [x] 变体矩阵（落到现有家族）：`IllustCard`（feed 列表卡 + heroScope 作用域）、`MediaPreview`→`PixivImage` 命名构造族（avatar/feed/detail/original 已是变体契约）、`AuthorSummary`（standard/compact，`author_summary.dart`）、`TagChips`/`TagChip`（interactive/display/blockMode，`tag_chips.dart`）、`ActionBar`→`BookmarkSwitchButton`/`FollowSwitchButton` 家族 + AppBar actions（`f4f55d4`）。
- [x] 提交边界：tokens（`69e6f97`）、基础组件（`7903e48`）、组件测试（`2f26f0a`）分开提交；每次提交都跑 `dart format`、`flutter analyze --no-pub` 和聚焦 widget 测试。

依赖：阶段 0。退出条件：组件在独立样例页覆盖亮/暗、长文本和异步三态，主题对比度问题有记录或通过。

## 阶段 2：应用壳层与信息架构

- [x] `/me` 升为第 5 个 branch：`_commonBranchRoutes` 移除 `me`、`openMe` 改 `context.go('/me')`、`_stackRoots` 加 `/me`、第 5 项换 `Icons.person_outline` + 新增 `homeMe` l10n（4 个 arb 同步并重新生成）。
- [x] `/settings` 整树改挂 root-level route（URL 空间、深链、恢复、`includeHistory:false` 语义不变；branch key 随分支移除，`settingsRouteObserver` 合并入 root observer）；`routeExternalIntent` 落点测试通过。
- [x] 窄屏四个内容 tab 页 AppBar actions 注入共享 `SettingsActionButton`（`MePage` 按 spec 契约不加 settings 入口）；宽屏 `NavigationRail`（≥600，trailing 齿轮同级入口）；`home_shell_metrics.publish` 改 nullable，无底栏时上报 `(null, 0)`，Hero 兜底到视口边。
- [x] 路由门面只传 id/query 等稳定参数；新增 settings 推入/返回、`/settings/theme` 深链、rail trailing 入口测试；bar 相关测试 pin 390×844 窄画布；`home_bar` golden 重生成。
- [x] 提交：`nav(ui): promote my and separate app settings`（`18bd872`）。

依赖：阶段 1 的 tokens、壳层基础组件。退出条件：手机/宽屏导航与路由聚焦测试通过，设置不再藏在“我的”账户菜单内。

## 阶段 3：样板链路（推荐 → 详情 → 作者 → 我的/收藏）

- [x] 推荐页/详情页迁移：`IllustFeedGrid`+`IllustCard`+共享状态件已统一；详情页 info_block 迁移到 `AuthorSummary`/`TagChips`，私有状态件删除（`f4f55d4`）；分页、Ugoira、Hero、收藏/下载语义由既有测试守护（全绿）。
- [x] 作者页/“我的”页：`_ProfileStatusPage` 收敛为 `ReplicaScaffold`+`FeedEmpty`；profile feeds 统一 `FeedLoading`/`FeedError`/`FeedEmpty`；关注/收藏按钮沿用既有共享开关件。
- [x] 状态覆盖：内容/加载/空/错误/受限态走共享件；亮/暗组件 golden（`golden_matrix_test.dart`）+ 长文本 ellipsize + 2x 大字号冒烟测试；多语言标签测试沿用 home_page_test 四语言断言。
- [x] 提交按可回滚单元拆分（tokens/组件/测试/nav/spinner 收敛/约束列数各自独立提交），每单元伴随对应行为测试与 golden。

依赖：阶段 1、2。退出条件：样板链路端到端视觉一致，返回/恢复/异步状态无回归；这是后续页面迁移的组件契约来源。

## 阶段 4：页面族迁移

- [x] Feed 族：排行、最新、搜索、历史共用 `IllustFeedGrid`+`FeedLoading`/`FeedError`/`FeedEmpty`/`FeedTail`，列数统一走 `illustColumnsFor`（`6634dab`）。
- [x] 内容族：小说列表/阅读、评论、标签、受限内容、Ugoira 查看器统一共享状态件；`TagChip`/`AuthorSummary` 命名变体迁移（`f4f55d4`+`2dc31fe`）。
- [x] 个人与任务族：收藏/关注/粉丝/下载任务/账户管理/登录/引导统一 `ReplicaScaffold` 头部与共享状态/反馈件。
- [x] 设置族：子页沿用 `settings_*` tile/control 原语与共享状态件；`/settings` 作为根级流程与“我的”内容区保持视觉区分，tokens 共享。
- [x] 每批提交运行聚焦测试 + `git diff --check`；未迁移清单：页面级多语言 golden 矩阵与设备实测归入阶段 5/6 记录。

依赖：阶段 3；详情/Ugoira 和设置页面的高风险行为测试必须先绿。退出条件：主要页面迁移清单完成，无未记录的局部魔法样式。

## 阶段 5：响应式、可访问性与视觉证据

- [x] 纯函数固化：`illustColumnsFor` 导出为吃 `crossAxisExtent` 的纯函数，`IllustFeedGrid` 改走 `SliverLayoutBuilder` 约束（rail 下不再用整窗宽）；`AppBreakpoints`（compact/medium/expanded + `useNavigationRail`）共享给壳层；`responsive_layout_test.dart` 覆盖 320/390/600/840/1200/1600 与边界（`6634dab`）。
- [x] 可访问性：`MotionTokens.resolve` 提供 reduced-motion 收敛，路由转场已接入门控（`_page` 下 zero 时长）；TagChip `Semantics(selected:)`、FeedLoading `semanticsLabel`、IconButton tooltip 约定沿用；2x 大字号 FeedEmpty 冒烟测试通过；最小触控目标在 chip/inline 控件仍 <48dp——记录为已知限制，深度键盘/焦点与真机扫描归入 windows 桌面任务。
- [x] golden harness：`test/goldens/` 组件级亮/暗矩阵（feed 三态、tag chips、author summary、home bar）；页面级 亮/暗×四语言 完整矩阵未建——组件级矩阵 + 语言标签测试构成当前证据，页面矩阵留待视觉验收阶段在真机/截图流程补齐。
- [x] 提交：`test(ui): add responsive accessibility and visual matrix`（`1e1d173`）。

依赖：阶段 4。退出条件：自动测试、人工视觉审阅、真实设备验证三类结果分栏记录，失败项有明确回滚或后续任务。

## 阶段 6：全量质量门与交付评审

- [ ] 运行 `dart format --set-exit-if-changed .`、`flutter analyze --no-pub`、聚焦测试、全量 `flutter test`，必要时运行 Android/Kotlin 测试。
- [ ] 检查 `git diff --check`、未涉及文件和既有用户修改；确认没有 mock、空操作处理器、吞错或未记录降级。
- [ ] 根据 PRD Acceptance Criteria 逐项附证据，区分 Implemented、Compiled、Unit-tested、Device-tested。
- [ ] 实现完成并通过质量门后按仓库流程归档（任务已 `in_progress`，首个 PR 已合入前置修正）。

依赖：阶段 0–5。退出条件：PRD 验收项全部有证据，用户明确接受剩余限制和未执行的设备检查。

## 回滚策略

每个阶段使用独立提交；发现路由、状态、Hero、分页、Ugoira 或下载语义变化时只回滚当前阶段提交，不使用 `git reset --hard`、`git clean` 或覆盖其他用户改动。主题 token 与组件迁移先保留兼容适配层，待所有调用点迁移并验证后再删除旧样式。
