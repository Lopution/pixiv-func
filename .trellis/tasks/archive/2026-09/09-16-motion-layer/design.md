# 设计：动效层补齐（motion-layer)

## 目标与边界

把动效从"只有 hero 图片转场"补齐为统一语法：**页面转场分级、列表首屏进场、按压反馈、弹层时长、减少动态效果**。不改动 hero 转场、viewer 手势、predictive-back 的已验证行为；不追逐逐帧效果，只立可复用的语法与降级闸门。

## 现状证据

- `MotionTokens` 已有 pageTransition/fast/medium/imageFade 系 + `resolve(context)` 只合并 `MediaQuery.disableAnimations`，无应用内开关。
- 路由全部走 `_page`：右滑入 + `_RoutePopSnapshot` + 转场期 `TickerMode` 冻结（已验证，不动）。
- 分支切换 = `StatefulShellRoute.indexedStack` + `NoTransitionPage`，瞬时硬切；底栏指示器已有弹性拉伸（自研 landing-ink 重放）。
- 散落 UI 字面量：`func_bottom_nav.dart` 300/200/130ms、`smooth_wheel_scroll.dart` 240ms；`showModalBottomSheet`×4、`showDialog`×6 全部走框架默认时长/曲线。
- 卡片无任何按压反馈（`IllustCard` GestureDetector / `NovelRow` InkWell 只有 ink)；列表项进场无 stagger。

## 契约

### 1. MotionTokens 扩展（单一来源）

新增 token，组件禁字面量：

| Token | 值 | 用途 |
|---|---|---|
| `press` / `pressCurve` / `pressScale` | 120ms / easeOut / 0.97 | 卡片按压回弹 |
| `listEntrance` / `listEntranceCurve` / `listEntranceOffset` / `listStaggerStep` / `listEntranceMaxItems` | 220ms / easeOutCubic / 12px / 30ms / 24 | 首屏 stagger fade+slide |
| `sheet` / `sheetCurve` | 250ms / easeOutCubic | bottom sheet 弹出 |
| `dialog` | 220ms | 对话框呈现 |
| `navIndicator` / `inkHold` / `inkFade` | 300 / 130 / 200ms | 底栏指示器与 landing ink |
| `wheelScroll` | 240ms | SmoothWheelScroll 动画 |

`resolve(context, base)` 扩展为合并两个来源：`MediaQuery.disableAnimations` **或** 应用内 reduce-motion 开关；`enabled(context)` 暴露 bool 供非 duration 语义（stagger、scale）直接跳过包装。

### 2. Reduce-motion 设置（单 owner)

- `AppSettings.reduceMotion`(bool，默认 false,json key `reduceMotion`,schema 版本不变——新增字段向前兼容）+ `setReduceMotion`。
- `MotionScope`(InheritedWidget）挂在 `app.dart` 的 `MaterialApp.builder` 输出上，`MotionTokens.resolve`/`enabled` 读取它。测试可用 `MediaQuery(disableAnimations:true)` 或直接包 `MotionScope(reduce:true)`。
- 设置入口放「浏览设置」页尾部 `SettingsControl`;l10n 键 `reduceMotion` + `reduceMotionHint`，四语言。
- 降级语义：**去掉位移/缩放/淡入飞行，不丢状态**——路由瞬时切换、列表项直接终态、sheet 用 `AnimationStyle.noAnimation`。

### 3. 页面转场分级

- 下钻推栈：保持现有右滑 + 冻结/快照（不动）。
- 模态页（搜索输入 `_modalPage`)：下缘滑入 + 淡入（`Offset(0,0.06)→0` + fade)，同样套 ticker 冻结 + pop snapshot。viewer 保持原状（hero 飞行 + drag-dismiss 已验证）。
- 分支切换：`HomePage` 给 `navigationShell` 套 keyed `FadeTransition`(AnimatedSwitcher 语义的手写版——`BranchSwitchFade` 记录上一 branch index,crossfade 250ms)。IndexedStack 语义保留：分支状态不丢。宽屏 NavigationRail 切换同规则。
- 转场构建收敛到 `lib/app/motion/page_transitions.dart`:`FuncRouteTransition`（推栈滑入）+ `FuncModalTransition`（模态上浮）+ `FuncBranchFade`（分支淡入）,`routes.dart` 只引用。

### 4. 列表首屏进场

- `lib/app/motion/feed_entrance.dart`:
  - `FeedEntranceScope`（有状态，随页面/feed 挂载创建）：单调计数器 `nextDelay()` → `index * staggerStep` 直至 `listEntranceMaxItems`，此后返回 null。
  - `StaggeredEntrance`：子树 fade+slide 入场；scope 缺席或额度用尽或降级 → 原样渲染。
- 挂载点：`IllustFeedGrid` 的 itemBuilder 包装（所有插画 feed 自动获得）+ 推荐页/小说排行的 `SliverList` 小说段显式包 `FeedEntranceScope`+`StaggeredEntrance`。
- 滚动期新建项不播（计数器额度天然限在首屏构建批次）；分支切换重建时重播 = 与分支淡入一致的轻反馈。

### 5. 按压反馈

- `lib/app/motion/press_scale.dart` `PressScale`:Listener onPointerDown→scale 0.97(120ms easeOut),up/cancel→回弹；`enabled=false` 或降级 → 透传。用 `AnimatedScale`+状态位，不自管 controller。
- 应用：`IllustCard`（整卡）、`NovelRow`(InkWell 外层）。hero 源框由 Transform 祖先仅影响绘制不改动 hero 测量误差≈2px，接受。

### 6. 弹层与字面量收口

- `lib/app/motion/app_sheet.dart` `showAppBottomSheet<T>`：包装 `showModalBottomSheet`，注入 `sheetAnimationStyle`（降级时 `AnimationStyle.noAnimation`)。替换 4 处调用点（filter sheet、history、follow/bookmark restrict)。
- `showDialog` 6 处统一 `transitionDuration: MotionTokens.resolve(context, MotionTokens.dialog)`。
- `func_bottom_nav.dart` 300/200/130ms、`smooth_wheel_scroll.dart` 240ms 全部改 token。

## 测试

- `test/motion_tokens_test.dart`:`resolve`/`enabled` 在 disableAnimations、MotionScope(reduce:true)、两者皆无下的矩阵。
- `test/motion_entrance_test.dart`:StaggeredEntrance 额度内动画、超额直出、降级直出；PressScale 按下 scale 值断言（非逐帧）。
- `test/settings`：新增 reduceMotion 序列化/迁移断言（在现有 settings 测试文件内追加）。

## 风险

- AnimatedSwitcher 类 crossfade 会让两分支短暂共存 → 用手写 `BranchSwitchFade` 控制时长与 dispose 时机，禁复杂 switcher。
- 卡片按压与 hero 起点的亚像素差：接受（0.97 上限、120ms 内完成）。
- `sheetAnimationStyle` 需 Flutter ≥3.22，本仓 3.47 满足。
