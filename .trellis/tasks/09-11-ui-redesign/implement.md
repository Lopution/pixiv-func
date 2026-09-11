# 实现计划：全项目 UI 视觉与交互重构

本计划按可独立验证的阶段组织。每个阶段结束后都能运行聚焦检查并可单独回滚；阶段依赖写在这里，不能只依赖任务树顺序。全局视觉原则和样板链路已获用户确认；完整设计评审通过后才进入实现阶段。

## 阶段 0：基线与约束冻结

- [x] 前置修正已随首个 PR 合入 main（nav 栈本地化、底栏间距、Ugoira 拖拽、搜索内联建议），记为 merged prerequisites，不计入任何阶段完成度。
- [ ] 记录当前 `flutter analyze --no-pub`、`flutter test`、golden 清单、关键页面截图和现有路由/恢复行为，以任务 HEAD 落盘为准（`baseline-2026-09-07` tag 仅历史参照）。
- [ ] 盘点 `lib/app/theme`、shell、feed/detail/profile/settings 共享组件和重复变体（清单见 design.md §0），标注高风险业务消费者。
- [ ] 建立视觉矩阵目录与命名规则；确认本阶段不触碰 API、OAuth、网络策略、下载恢复和阅读位置。
- [ ] 提交：`docs(ui): capture redesign baseline and constraints`。

依赖：无。退出条件：基线命令与未执行项均有记录，用户数据/业务契约清单已确认。

## 阶段 1：设计 tokens 与基础组件

- [ ] `FuncTokens` 保留为基础常量 owner；新增语义化 `ThemeExtension`（引用常量、不复制色值）与亮/暗映射，完成排版/间距/形状/动效语义；删除扁平 `surfaceContainer*` 的直接使用。
- [ ] 在现有 `feed/*`、`replica_*`、`settings/*` 家族上扩展或显式重命名来统一公共 API，补缺 `FeedLoading`/`PageHeader`/按钮/标签/反馈（**不新建同义平行组件**）；补 semantics、tooltip 和触控尺寸测试。
- [ ] 为 `IllustCard`、`MediaPreview`、`AuthorSummary`、`ActionBar` 定义最小变体矩阵（落到现有家族或显式重命名），禁止新增页面私有近似组件。
- [ ] 提交边界：tokens、基础组件、组件测试分开提交；每次提交都跑 `dart format`、`flutter analyze --no-pub` 和聚焦 widget 测试。

依赖：阶段 0。退出条件：组件在独立样例页覆盖亮/暗、长文本和异步三态，主题对比度问题有记录或通过。

## 阶段 2：应用壳层与信息架构

- [ ] `/me` 升为第 5 个 branch：从 `_commonBranchRoutes` 移除 `me`、`openMe` 改 branch 切换、`_stackRoots` 加 `/me`、底栏第 5 项换"我的"图标与 l10n key（4 个 arb 同步）。
- [ ] `/settings` 整树改挂 root-level route（URL 空间、`settingsNavigatorKey`/`includeHistory:false` 语义、深链与恢复不变）；`routeExternalIntent` 落点回归。
- [ ] 窄屏在五个 tab 页 AppBar actions 注入同一个共享 settings action（不加 shell chrome）；宽屏 `NavigationRail`/Drawer 同级设置入口；`home_shell_metrics` 无底栏时如实上报。
- [ ] 路由门面只传 id/query 等稳定参数；为分支栈、深链接、恢复和预测性返回补测试；`navigation_router_test`/`navigation_restoration_test`/`home_bar` golden 同步更新。
- [ ] 提交：`nav(ui): promote my and separate app settings`。

依赖：阶段 1 的 tokens、壳层基础组件。退出条件：手机/宽屏导航与路由聚焦测试通过，设置不再藏在“我的”账户菜单内。

## 阶段 3：样板链路（推荐 → 详情 → 作者 → 我的/收藏）

- [ ] 用共享图库/卡片/媒体区/作者摘要/操作栏迁移推荐页和作品详情页，保留分页、Ugoira、Hero、收藏/下载真实状态。
- [ ] 迁移作者页与“我的”页的头部、tab、收藏/关注入口；关键 action 提供可见按钮和进行中/成功/失败反馈。
- [ ] 覆盖内容、加载、空、错误、受限、长标题、多语言、亮/暗和代表宽度截图/测试。
- [ ] 提交按“feed、detail、profile”三个可回滚单元拆分；每单元运行对应行为测试与 golden。

依赖：阶段 1、2。退出条件：样板链路端到端视觉一致，返回/恢复/异步状态无回归；这是后续页面迁移的组件契约来源。

## 阶段 4：页面族迁移

- [ ] Feed 族：排行、最新、搜索、历史统一列数、筛选、分页尾部和错误重试。
- [ ] 内容族：小说列表/阅读、评论、标签、受限内容和动图查看器使用命名变体，不强行套用插画卡片。
- [ ] 个人与任务族：收藏、关注、粉丝、下载任务、账户管理、登录/引导统一页面头部、状态和反馈。
- [ ] 设置族：拆分设置子页和共享 tile/control 原语；应用设置与“我的”账户内容保持视觉区分但共享 tokens。
- [ ] 每个页面族提交前运行页面族聚焦测试、响应式断点测试和 `git diff --check`；保留未迁移清单。

依赖：阶段 3；详情/Ugoira 和设置页面的高风险行为测试必须先绿。退出条件：主要页面迁移清单完成，无未记录的局部魔法样式。

## 阶段 5：响应式、可访问性与视觉证据

- [ ] 固化基于可用宽度的列数/最大内容宽度/导航形态纯函数和测试（`_illustColumnsFor` 导出为吃 `crossAxisExtent` 约束而非 `MediaQuery` 整窗宽）；覆盖 320/390/600/840/1200dp 与横屏。
- [ ] 完成大字号、键盘/遥控焦点、语义标签、最小触控目标和 reduced-motion 验证；修复溢出、遮挡和不可达操作。
- [ ] golden 分级：样板链路（推荐→详情→作者→我的/收藏）走亮/暗×四语言×关键状态完整矩阵，其余页面族只做代表状态矩阵；先补 golden harness（泵装、字体/环境稳定化、CI 一致性），记录设备与人工复核结果。
- [ ] 提交：`test(ui): add responsive accessibility and visual matrix`。

依赖：阶段 4。退出条件：自动测试、人工视觉审阅、真实设备验证三类结果分栏记录，失败项有明确回滚或后续任务。

## 阶段 6：全量质量门与交付评审

- [ ] 运行 `dart format --set-exit-if-changed .`、`flutter analyze --no-pub`、聚焦测试、全量 `flutter test`，必要时运行 Android/Kotlin 测试。
- [ ] 检查 `git diff --check`、未涉及文件和既有用户修改；确认没有 mock、空操作处理器、吞错或未记录降级。
- [ ] 根据 PRD Acceptance Criteria 逐项附证据，区分 Implemented、Compiled、Unit-tested、Device-tested。
- [ ] 实现完成并通过质量门后按仓库流程归档（任务已 `in_progress`，首个 PR 已合入前置修正）。

依赖：阶段 0–5。退出条件：PRD 验收项全部有证据，用户明确接受剩余限制和未执行的设备检查。

## 回滚策略

每个阶段使用独立提交；发现路由、状态、Hero、分页、Ugoira 或下载语义变化时只回滚当前阶段提交，不使用 `git reset --hard`、`git clean` 或覆盖其他用户改动。主题 token 与组件迁移先保留兼容适配层，待所有调用点迁移并验证后再删除旧样式。
