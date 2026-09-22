# 代码调研：图片查看器状态机 — `lib/features/illust/viewer/image_viewer_page.dart`

对应设计 §4.4：chrome 显隐、双击常规缩放、适配屏幕、页码选择、保存/分享/信息、空态、键盘/鼠标等价。行号已核对。

## 1. 入参与路由接线

- 构造参数：`urls`（必需）、`initialPage`、`heroTagForPage`（L24/33）、`prefetchUrlForPage`（L28/45）等；无 `onDownload`/`onShare`/实体对象——**查看器是纯 URL 驱动**，不持有 `IllustEntity`。
- 路由 `illust/:illustId/viewer/:page`（routes.dart L418）：builder 内 `ref.watch(illustStoreProvider).get(illustId) ?? extra?.entity`（L126）取 canonical 实体，`urls = entity?.viewerUrls(quality) ?? extra?.urls ?? const []`（L127）——**store 优先、extra 仅作首帧快照**，符合 spec 的 route-facade 约定。
- `ImageViewerRouteExtra`（routes.dart L94-117）：urls/entity/heroScope/heroImageUrl 等内存快照，不可持久化。
- 页码持久化：`onPageChanged: (page) => replaceImageViewerPage(...)`（routes.dart L144）——滑动改页会 `replace` 当前路由的 `:page`，**返回键回退链不受滑动污染**，深链可直达第 N 页。

## 2. 状态机现状

### 缩放
- `minScale = 0.9`、`maxScale = 6.0`（L48-49，注释 L47 标 PRD R3）。
- 每页独立 `TransformationController`：`_transformations` map（L57）+ `_transformationFor` `putIfAbsent`（L145-146）——**缩放态按页保留，切页不丢**。
- 单指/双指由 `InteractiveViewer`（L183）默认手势处理；**无双击缩放、无缩放级别指示、无“恢复适应”显式动作**。

### 页与缩放互斥
- `_activeZoomed` 为真时 `PageView.physics = NeverScrollableScrollPhysics`（L177-178）：放大后禁止翻页，符合预期。
- L135-137 注释：只在缩放态翻转时 setState，避免逐帧重建整个 PageView（U3/R7 性能约束）。

### chrome
- `Scaffold(backgroundColor: 黑)` + 常驻 `AppBar`，title = `'${_activePage + 1} / $_pageCount'`（L163-166）。
- **chrome 不可隐藏**：无点击空白收起/展开，无沉浸式全屏；AppBar 无 actions（无保存/分享/信息按钮）。
- 对比 PixEz：有常驻底部工具栏（页数、全屏、画质、分享、长按保存）+ 系统 UI 显隐切换——见 external-pixez-viewer.md。

### 输入等价
- **无键盘快捷键**（无 CallbackShortcuts/Focus）；**无鼠标滚轮缩放/翻页**；桌面端仅靠手势与 AppBar 返回。
- 返回优先级：无 `PopScope`/`WillPopScope` 拦截——**放大状态下按返回直接退路由**，不会先复位缩放（设计 §4.4 要求的“返回先退出放大/模式，再退页面”目前不成立）。

### 空态
- `_activePage = _pageCount == 0 ? 0 : initialPage.clamp(...)`（L66-69），注释明示 “1 / 0” 占位 + R6 不崩溃。
- body `_pageCount == 0 → FeedEmpty(viewerNoImages)`（L168-174）；AppBar title 仍渲染 `1 / 0`（L166）。
- **缺口**：占位文案有了，但 `1 / 0` 标题是误导性页码；且无“无图可保存/分享”的动作禁用语义（当前根本没有动作）。

### 预取
- `_prefetchNeighbours`（L104-117）：初始页与每次翻页后预取 ±1 页（L74、L96），经 `prefetchUrlForPage` 回调到 medium 档；越界保护 L117。

### Hero
- `heroTagForPage` 非空时包 `Hero`/transitionKey（L182-210、L223），与详情页 `illustHeroTag(heroScope, illustId)`（routes.dart L131-134）配套。

## 3. 与 §4.4 验收的差距清单

| 要求 | 现状 | 差距 |
|---|---|---|
| chrome 显隐 | 常驻 AppBar，不可隐藏 | 需实现点击切换 + 系统 UI 配合 |
| 双击常规缩放 | 无 | 需包 GestureDetector 双击→fit/放大循环 |
| 适配屏幕 | InteractiveViewer 默认即 fit | 缺显式“复位适应”动作 |
| 页码选择 | 仅 `n/total` 静态文本 | 需可点出页码跳转（sheet/输入） |
| 保存/分享/信息 | 无动作 | 需工具栏/菜单，接 DownloadCoordinator + ShareService + 详情回跳 |
| 空态 | `1 / 0` + 占位文案 | 标题页码需修正为无语义文案或隐藏计数 |
| 键盘/鼠标等价 | 无 | ←/→ 翻页、+/- 或滚轮缩放、Esc/Backspace 返回、F 全屏等 |
| 返回优先级 | 直接退路由 | 需 PopScope：zoomed→复位→再退；chrome 隐藏态先恢复 |
| 读屏可达 | AppBar 返回有默认语义；图片无语义标签 | 页内容/动作需 Semantics |

## 4. 复用资产

- `MotionTokens`：sheet 250ms/dialog 220ms/fast 180ms（lib/app/motion/motion_tokens.dart L37/41/17）——chrome 淡入淡出、页码 sheet 都应取 token 而非自造时长。
- `showAppBottomSheet`/`showAppDialog`：页码选择、信息面板、确认框的统一通道。
- `PlatformCaps.system().isDesktop`（platform_caps.dart L36）：键鼠等价可按平台启用提示/帮助。
- 路由已能独立打开第 N 页——“从详情第 k 页进入查看器第 k 页”链路已通（`openImageViewer` 传 initialPage + extra）。
