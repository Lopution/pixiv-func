# 执行计划：发现页查询上下文连续（W2）

需求见 `prd.md`，技术设计见 `design.md`，逐文件精确改动方案见
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
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-discovery-query-context
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：看似不相关的
> 测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec 判定噪声再排查。
> `InAppWebView` 页面测试须在 `setUp` 装平台 fake（quality-guidelines 既有模式，
> 参照 `test/reverse_image_search_page_test.dart`）。

**前置**：W1（`09-22-interaction-outcome-correctness`）已合入 `main` 才能
`task.py start`——本包阶段 3/4 触碰的 `search_page.dart`（热门标签）与
`reverse_image_search_page.dart`（cancel 语义）依赖 W1 修复先落地；start 前
rebase 到含 W1 的 `main` 并按父 implement.md §5 Step A 重新定位行号。

本机不可验项（桌面滚轮 re-tap 手感、Android 进程死亡恢复、弹栈-回顶时序、
1.3x 大字号观感、TalkBack/Narrator 朗读）在 PR body 标「未验证」，不得由
widget test 推断通过。

## 阶段 1：re-tap 通道 + 排行族（R10, R2, R3）

- [ ] **R10**：`lib/app/widgets/branch_slide_stack.dart` `BranchSlidePager`
      新增 `reTapEvents` 广播（`ValueNotifier<int>`/等价 stream，负载 = branch
      index）；`selectIndex`（:137-151）`index == tab.index` 分支在
      `goBranch` 后发射；`syncIndex`/`_suppressGoBranch`（:156-172）不发射；
      注释冻结通道契约（发射点/负载/post-frame 消费，供 W3 复用）。
      测试：`test/navigation_router_test.dart` 或 `func_bottom_nav_test.dart`
      补「同槽点按 → 信号一次」「syncIndex/拖拽 settle → 不发射」。
      提交：`feat(nav): 新增 branch re-tap 回顶广播通道`
- [ ] **R2**：`lib/features/ranking/ranking_page.dart` `TabBar`（:104-115）接
      `onTap`：同索引 → `_scrollControllerFor(mode)`（:43,:80-82）
      `animateTo(0)`（`MotionTokens` gated，reduced-motion → `jumpTo(0)`）；
      异索引不变；`onTap` 内不调 `TabController.animateTo`。
      测试：同索引点击 → feed offset 归 0 且不 refresh。
      提交：`feat(ranking): 重复点击当前榜单回到顶部`
- [ ] **R3**：`lib/app/navigation/routes.dart`（:549-553）`/novel-ranking` 加
      `?mode=` + `replaceNovelRankingMode` facade（`context.replace`）；
      `lib/features/ranking/novel_ranking_page.dart` body 由 `ValueKey` 替换
      （:90-95）改 `RootSwipeSwitcher + TabSlideStack` per-mode 常驻 body，
      复用 `_scrollControllers`（:33,:64-66）与 `novel-ranking-<mode>` keys
      （:168,:172）；`TabController` 由参数 `initialIndex` 播种 + guarded sync；
      `TabBar.onTap` 同索引回顶。
      测试：`test/navigation_restoration_test.dart` 补 novel-ranking mode
      恢复；swipe 切换与点击同走一套规则；同索引回顶。
      提交：`feat(ranking): 小说排行接入滑动切换与 mode 路由参数`

## 阶段 2：推荐 + 新作（R1, R4）

- [ ] **R1a**：`lib/app/navigation/routes.dart`（:963-972）`/recommended` 加
      `?type=` + `replaceRecommendedType` facade；
      `lib/features/home/recommended/recommended_home_page.dart` `_type`（:47）
      改由路由驱动：`TabController`（:54-55）`initialIndex` 播种 + guarded
      sync，`_onTabChanged`/`_selectType`（:66-80）写 facade；`context.replace`
      重建不重置 TabController。
      测试：`test/recommended_home_test.dart` 补 type 参数 round-trip 与
      replace 下手势不重置。
      提交：`feat(recommended): 内容类型接入路由参数`
- [ ] **R1b**：同文件删 `FittedBox(scaleDown)`（:158-164）改 `isScrollable: true`
      TabBar；新增 per-type `ScrollController` map 传入 `SmoothWheelScroll`
      （:302）；订阅 `reTapEvents` post-frame → 当前 type controller 回顶
      （`hasClients`+`mounted` 守卫）；刷新失败错误（:264-282）改经 #48 共享
      SnackBar 提示。
      测试：re-tap 终态 offset=0 不 refresh；刷新错误 SnackBar 可见；
      断言不经 `PrimaryScrollController`。
      提交：`feat(recommended): 标签不缩字、re-tap 回顶、刷新错误就近提示`
- [ ] **R4a**：`lib/app/navigation/routes.dart`（:987-996）`/new` 加
      `?scope=&type=` + `replaceNewFeed` facade；`lib/features/new/new_page.dart`
      `_selectedIndex`/`_type`（:37-38）改由路由驱动（同 D13 防护）。
      测试：`test/new_content_feed_test.dart` 补 scope/type 参数 round-trip。
      提交：`feat(new): 范围与类型接入路由参数`
- [ ] **R4b**：同文件删 `_selectorExpanded`（:39）与 `_onTabTap` 展开逻辑
      （:70-74,:102）；新增常驻可见 type 选择器行（AppBar 下 segmented/chips）；
      scope `TabBar` 去 `FittedBox`（:106-112）；scope tab 同索引与 type chip
      同索引点击 → 当前 `NewFeedKey` controller（:230,:235,:299）回顶，
      Offstage 隐藏 body（:149-161）`hasClients`+`mounted` 守卫。
      测试：固化「re-tap 展开」的旧用例同 commit 删改；type 控件常驻断言；
      两种同索引点击回顶。
      提交：`feat(new): 类型选择器常驻，re-tap 固定回顶`

## 阶段 3：搜索族（R7, R8, R5, R6）

- [ ] **R7**：`lib/app/navigation/routes.dart` `_searchFilters`/
      `_searchQueryParameters`（:295-347）补齐 `ai`/`bmin`/`bmax`/`ratio`/`ct`/
      `wmin`/`wmax`/`hmin`/`hmax`（字段对齐 `search_models.dart:120-160`）；
      解码保持宽松（`firstWhere orElse`）。
      测试：`test/search_catalog_test.dart` 补 13 字段 URL→`SearchQuery`→
      `cacheKey` round-trip 一致、非法值落默认。
      提交：`fix(search): 路由序列化补齐全部筛选字段`
- [ ] **R8**：`lib/features/search/search_page.dart` 热门标签（:153-155 处，
      W1 已修 itemCount）改自适应网格 `SliverGridDelegateWithMaxCrossAxisExtent`
      全渲染、保留瓷砖代表图；`CustomScrollView`（:37-40）补显式
      `ScrollController` 并订阅 `reTapEvents` 回顶；trending kind 内存态写入
      恢复矩阵注释。
      测试：`test/search_catalog_test.dart` 既有「trims to complete rows」用例
      随 W1 已改写，本包补「全部标签渲染 + 列数随宽度」与 re-tap 回顶。
      提交：`feat(search): 热门标签自适应网格并接入 re-tap`
- [ ] **R5**：`lib/features/search/search_result_page.dart` 头部（:66-79）
      关键词改可编辑入口（tap → `openSearchInput(q, type)` 预填 push）+ 筛选
      摘要 chips 行 + 清除入口；空结果 `FeedEmpty`（:138-145）加「修改搜索」
      动作走同一入口；header 在 loading/error/empty 保持可见。
      新 l10n key：`searchModifyQuery` 等（feature 前缀，四语）。
      测试：tap → 输入页预填同 q/type；摘要随筛选变化；空态动作可达。
      提交：`feat(search): 结果页查询头可编辑并补筛选摘要`
- [ ] **R6**：`lib/features/search/search_page.dart` `_selectSuggestion`
      （:419-424）拆分：行点击 = 只填 `TextEditingController` 不提交；行尾动作 =
      立即搜索；两动作可访问标签可区分。
      测试：tap → 字段更新且无提交；trailing → 提交；既有
      「inline suggestions submit」用例同步更新。
      提交：`feat(search): 建议项区分填入与立即搜索`

## 阶段 4：反向搜图任务头（R9）

- [ ] **R9**：`lib/features/search/reverse_image_search_page.dart` 新增常驻任务头
      （缩略图 + 引擎 chip + 阶段标识），在 ready/searching/failure/
      success-empty/结果/WebView 各阶段（`_body` :177-202 上方）保持可见；
      引擎 chip 仅在允许切换的阶段可交互；不延长 temp file 持有
      （webUpload 语义 :332-339）；`Session` 保持 `initState` 创建（:66-82）；
      消费 W1 已合入的 `stopSearch`/cancel 语义，本包不改状态机。
      测试：`test/reverse_image_search_page_test.dart` 补各阶段 header 渲染
      同图同引擎（InAppWebView fake 沿用既有模式）。
      提交：`feat(search): 反向搜图常驻图片与引擎任务头`

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec 判定）；
      `git diff --check` 干净；`task.py validate` 通过；新 l10n key 四语齐备。
- [ ] 父 implement.md §6「W2/W3」门禁逐条核对并记录结果；运行时缺口在 PR 标
      「未验证」。
- [ ] PR：`gh pr create --fill`（body 写明 re-tap 通道契约移交 W3、残缺 URL 过渡
      期 cacheKey 自愈注记、New 页 re-tap 行为迁移说明）；CI 绿后
      `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后提交）。

## 边界（不做）

- 排名/进度变体组件与排行页接入、`IllustCard`/`NovelRow`/`NovelCard` 收敛（W6）。
- W3 页面（作者/收藏/资料）对 re-tap 通道的消费——通道契约本包冻结移交。
- 热门标签 itemCount 修复本体、反向搜图 cancel 语义本体（W1，前置已合入）。
- 反向搜图输入图的进程死亡恢复（声明 memory-only）。
- 搜索历史、Spotlight、tag_search_page 独立改版（W9）；其他页面 QueryContext。
- `func_bottom_nav.dart`、`root_swipe_switcher.dart`、`SmoothWheelScroll` 内部
  改动；新 provider family、第二广播总线、`RestorationMixin` 铺开、网络/数据层。
