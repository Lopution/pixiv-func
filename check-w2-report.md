# Check Report: W2 发现页查询上下文连续 (PR #59, merge ee32dd0)

Scope: `git diff ee32dd0^1 ee32dd0` + 各提交。对照归档任务 prd/design/implement + 父门禁 §6 W2/W3。
Env: `/root/Pixiv-func-check2` detached HEAD @ b31846c (origin/main).

## 检查流水（增量落盘）

### 1. R10 ReTapChannel (branch_slide_stack.dart)
- PASS: `ReTapChannel extends ChangeNotifier`，`emit` 无条件 `notifyListeners()`——连续同槽点按不丢信号（满足 implement 中途修订的"事件而非状态"要求）。负载 `branch` getter + 同步 emit。
- PASS: 发射点只在 `selectIndex` 的 `index == tab.index` 分支、goBranch+popUntil 之后；`syncIndex`/拖拽/`_suppressGoBranch` 不发射（branch_slide_stack.dart:143-162）。
- PASS: popUntil 谓词 `route.isFirst || popDisposition == doNotPop` 切断 PopScope-vetoed 死循环，注释完整（:148-157）。
- PASS: `reTapScrollToTop` = MotionTokens.resolve gated animateTo(0)，zero → jumpTo(0)，hasClients 守卫（:274-281）。语义纯滚动。
- PASS: spec "Branch Re-tap Contract" 章节已写入 component-guidelines.md（发射点/消费时序/消费守卫/页内同索引规则冻结给 W3）。
- NOTE: spec 文本建议负载 `{branchIndex, sequence}` 或 Stream；实现用 ChangeNotifier 无 sequence——语义等价（notifyListeners 无条件触发），可接受。
- PASS: reTapEvents 在 dispose 中释放（:228）。

### 2. R1 推荐页 (recommended_home_page.dart)
- PASS: `initialType` + `onTypeChanged` facade 路由驱动；TabController initialIndex 播种（:71-75）。
- PASS: didUpdateWidget guarded sync：param 变化且 index 不同才 `_tabController.index = index`，`_suppressRouteEcho` 防回环（:89-108）；`context.replace` 不重建 controller。
- PASS: `_onTabChanged` 早退防动画结束二次通知触发重复 replace（:123）。
- PASS: `_onBranchReTap` 过滤 `channel.branch == BranchRootScope.branchIndex`，post-frame + mounted + isCurrent 守卫（:137-146）。
- PASS: TabBar.onTap 同索引 → `!indexIsChanging` → reTapScrollToTop（:168-175），不调 TabController.animateTo（符合 Tab Navigation Animation Contract）。
- PASS: FittedBox(scaleDown) 已删，isScrollable: true + tabAlignment.start（:223-235）。
- PASS: per-type ScrollController map → SmoothWheelScroll.controller（:61,148-150,385-389）；dispose 全释放（:116-118）。
- PASS: 刷新错误 → ref.listen refreshPhase→error 边沿 → showAppSnackBar + retry action，consumeRefreshError() 取真实错误（:267-285；paged_feed_controller.dart:484）。loading/error/empty 分支 listen 仍注册（:267 在 when 之前）。
- PASS: 旧尾部 refresh-error FeedTail 行移除（D12 允许实现定），SnackBar 在当前视野可辨。

### 3. R2/R3 排行族 (ranking_page.dart / novel_ranking_page.dart / routes.dart)
- PASS: ranking `TabBar.onTap` 同索引 → `!indexIsChanging` → reTapScrollToTop（ranking_page.dart:112-124），未调 TabController.animateTo。
- PASS: novel-ranking `?mode=` + `replaceNovelRankingMode` 走 `_currentStackRoot` 保 branch 前缀（routes.dart:1374-1380）；`initialMode` 播种 + `didUpdateWidget` guarded sync + `_suppressRouteEcho`（novel_ranking_page.dart:57-76）。
- PASS: novel body 改 `RootSwipeSwitcher + TabSlideStack` + `_loadedModes` + `onPrepareAdjacent`（:139-162），与插画榜同规则集。
- PASS: 同索引 onTap 回顶（:122-131）。
- ISSUE（可修）: `RankingPage` 未订阅 `reTapEvents`——branch 级 re-tap（底栏同槽点按）对排行分支只弹栈不回顶，而 recommended/new/search 三个 feed 根页都消费了通道（ranking_page.dart 全文无 reTapEvents）。5 个根页中 4 个 feed 根，排行是唯一缺失消费者 → 「同动作不同后果」。待修：补订阅 + 测试。
- NOTE: SettingsPage（根页 4）同样未订阅——但 leaf 范围为发现页，settings 不在 R1–R10 声明面内；记录为观察项不上修。
- ISSUE（报告级）: 宽屏 `NavigationRail.onDestinationSelected` 直接调 `navigationShell.goBranch`（home_page.dart:166），绕过 `pager.selectIndex`——宽屏同槽点按既不弹栈也不发 reTapEvents，re-tap 在 ≥600dp 完全缺失。pager 是 `widget.navigationShell` 的后代而非祖先，maybeOf 无法上行，修复需重构通道暴露方式 → 只报告。

### 4. R4 新作页 (new_page.dart)
- PASS: `initialScope`/`initialType`/`onFeedChanged` 路由驱动（:30-38,1081-1086）；`?scope=&type=` + `replaceNewFeed`（routes.dart:1390-1400）。
- PASS: `_selectorExpanded`/`_onTabTap` 展开逻辑删除；`_NewTypeSelector` 常驻（:210-214,254-291），ChoiceChip 行。
- PASS: scope tab 同索引 + type chip 同值 → reTapScrollToTop（:137-148,133-135 行内注释确认纯滚动）。
- PASS: `_onBranchReTap` 订阅 + branchIndex 过滤 + post-frame + isCurrent 守卫（:150-165）。
- PASS: scope TabBar 去 FittedBox 改 scrollable（:181-184）。
- PASS: per-NewFeedKey ScrollController map 由页持有，Offstage body 复用（:48,230-234）；dispose 全释放。
- PASS: `_NewFeedBody` ConsumerStatefulWidget→ConsumerWidget 收敛，controller 上移页级——无平行实现。
- NOTE: didUpdateWidget 内 setState 冗余但合法（:93-99）。

### 5. R5/R6/R7/R8 搜索族 + FeedEmpty
- PASS: `_searchFilters`/`_searchQueryParameters` 14 字段全量对称（target/sort/duration/start/end + ai/bmin/bmax/ratio/ct/wmin/wmax/hmin/hmax），wireValue↔wireValue、name↔name 一一对应，宽松解码落默认（routes.dart:326-411）。
- PASS: 结果页标题 InkWell+edit icon+tooltip → `openSearchInput(q, type)` 预填 push（search_result_page.dart:88-117）；`_FilterSummaryBar` 常驻 AppBar.bottom（loading/error/empty 均可见，:122-131）；清除 → `SearchFilters.defaults` + replaceSearchResults（:65-76）；三 feed 空态加「修改搜索」action（:201-208 等）。
- PASS: `FeedEmpty` 扩展 actionLabel/onAction 第二按钮，不 fork（feed_states.dart:114-172）。
- PASS: 建议项行点击=只填入 + requestFocus，行尾 IconButton=立即搜索；Semantics hint + tooltip 区分（search_page.dart:466-477,626-644）。
- PASS: 热门标签 `SliverGridDelegateWithMaxCrossAxisExtent(160)` 全渲染自适应（search_page.dart:196-201）；显式 ScrollController + reTap 订阅（:36-72）。
- PASS: trending kind 恢复等级注释声明 session memory（:23-32）。
- PASS: l10n 8 新 key 四语齐备 + lookup.dart 同步。

### 6. R9 反向搜图任务头 (reverse_image_search_page.dart)
- PASS: `_TaskHeader`（缩略图+引擎 chip+阶段标识）在 ready/searching/failure/success 全阶段常驻（:173-186,214-222）；idle/picking/preparing/canceled 不显示——无图片上下文阶段，合理。
- PASS: `_engineSwitchable` = input 非空 + ready/failure/success；searching 中 chip inert（:186-195）——「允许切换的阶段可交互」落实。
- PASS: `_lastInput` 仅作缩略图源，idle/canceled 清空，不延长 temp file 持有（:64-67,97-103）；`Image.file(cacheWidth:128)` + errorBuilder 兜底。
- PASS: `Session` initState 创建不变；消费 W1 stopSearch 语义未改状态机。

### 7. 测试覆盖
- PASS: re-tap 通道组（emit-once、连点 [0,0]、弹栈、异槽/sync/drag 不发射）root_swipe_switcher_test.dart:362-476。
- PASS: 搜索底栏 re-tap 端到端 offset→0 且无 refresh（search_catalog_test.dart:1303-1373）。
- PASS: 新作 scope/type 同索引回顶 offset=0 + 无 refetch + 常驻断言（new_content_feed_test.dart:183-234）；旧「re-tap 展开」用例同 commit 改写（:133）。
- PASS: 推荐 type round-trip + replace 不重置 controller（recommended_home_test.dart:219-289）。
- PASS: 推荐底栏 re-tap 端到端 offset→0 且无 refetch（recommended_home_test.dart，`branch re-tap scrolls the active feed to top without refetch`）；此前推荐是 4 个 feed 根页中唯一缺 re-tap 回归者。
- PASS: 推荐非首次刷新失败 → SnackBar + 重试 action 在视野内可辨（`a failed refresh surfaces a snackbar, not only a tail row`）——此前该 snackbar 路径无直接测试。
- PASS: novel-ranking mode 恢复测试（navigation_restoration_test.dart:223-272）。
- PASS: 反向搜图任务头各阶段同图同引擎断言（reverse_image_search_page_test.dart:464+）。
- PASS: `flutter analyze --no-pub` 全绿（3.4s）。

### 8. 门禁逐条（父 §6 W2/W3）
- 跳转前后范围/类型/筛选一致：PASS —— 全部路由化（?type/?scope&?type/?mode/13+1 筛选字段），replace 回写 + guarded sync，round-trip 测试齐备。
- 控件在 loading/error/empty 可见：PASS —— 选择器/摘要行均挂 AppBar/常驻 chrome（recommended TabBar、new _NewTypeSelector、result bottom PreferredSize、reverse _TaskHeader）。
- 不缩字：PASS —— FittedBox(scaleDown) 全删，isScrollable+TabAlignment.start。
- 排行 header 点击/滑动回归：PASS —— illust/novel 同规则集（RootSwipeSwitcher+TabSlideStack+onTap），测试在 ranking_feed/root_swipe_switcher。
- re-tap=回顶不承载其他入口：PASS —— reTapScrollToTop 纯滚动，所有消费点无 refresh/展开/切换；测试断言 offset=0 且无新请求。
- 恢复等级矩阵：PASS —— design §二表逐项落实；trending kind 与反图会话 memory-only 注释声明在代码（search_page.dart:23-32 / reverse_image_search_page.dart:62-67）。

### 9. 复核发现并处置

**已修（fix/09-22-w2-check，PR #78，commit a3f34c5 + 3940193）：**
1. `ranking_page.dart` — 排行根页未订阅 reTapEvents：底栏同槽点按只弹栈不回顶，与推荐/新作/搜索根页不一致。按冻结契约补订阅 + 测试（branch re-tap → offset 0 且无 re-request）。
2. `feed_states.dart` — FeedEmpty 缺 `onAction→actionLabel` 对称断言（`Text(actionLabel!)` 潜在 crash），按既有 retryLabel 断言惯例补齐。

**报告级（未修）：**
1. 宽屏 NavigationRail 同槽点按完全无 re-tap（既不弹栈也不滚动）——`onDestinationSelected` 直连 `goBranch`（home_page.dart:166）绕过 `pager.selectIndex`。pager 在 `widget.navigationShell` 下游，HomePage 无法 maybeOf 上行，修复需重构通道暴露（如 pager 上移或注册式访问），建议另立任务。
2. SettingsPage 根页未订阅 re-tap——leaf 范围为发现页，未越界；如需统一，属父任务级决策。
3. 边缘时序：`context.replace` 连击场景旧参数晚到会让 TabStrip 短暂回跳后自愈（最终态一致，属可接受竞态）。

### 10. 验证结论
- `flutter analyze --no-pub`：干净（main 与 fix 分支各一次）。
- `flutter test --no-pub`（ranking_feed + root_swipe_switcher + navigation_restoration）：30/30 通过（含新增 re-tap 用例）。
- `flutter test --no-pub test/recommended_home_test.dart`：7/7 通过（含复核新增 re-tap 回顶与刷新失败 SnackBar 两例）。
- `dart format` / `git diff --check`：干净。
- W1 边界：itemCount 修复本体、反图 cancel 语义均未触碰。
- 无 TODO/吞错/print；无平行组件/复制实现；l10n 四语齐备且 gen 文件同步。
