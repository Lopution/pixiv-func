# 交互与视觉现代化：material_ui、go_router、状态恢复（09-02 child F）

## Goal

在 C 交付的组件层之上，把 Func 的交互模型与视觉带到 Flutter 当前的官方形态：应用整体迁到 `material_ui`/`cupertino_ui`
并采用 Material 3；用 `go_router` 的 `StatefulShellRoute` 给首页各 tab 独立返回栈、支持进程被杀后的状态恢复与 `pixiv://`
深链；启用 Predictive Back；viewer 拖拽关闭驱动 Hero 反向转场；采用 `NavigationBar`/`SearchBar`/`SegmentedButton`。
这是 09-02 中**唯一允许改变用户可见交互与视觉**的 child；README "第一阶段冻结用户可感知体验" 原则在此终止。
对应 parent R10；决策 D-10～D-13；设计见 parent `design.md` §13.3。

## 开工 gate

- child C 归档：组件层（`lib/app/widgets/feed/`、`PixivImage` 变体）、`lib/app/motion/motion_tokens.dart`、以 id/参数为形参的
  `lib/app/navigation/routes.dart` 门面已存在。
- child B 归档：per-ABI 体积门禁存在，才能量化 `material_ui` 迁移对快照体积的影响。
- 开工第一步：记录当时 `lib/` 中 `package:flutter/material.dart` import 数、`Navigator.of(context).push(` 数、golden 清单、
  arm64 release APK 大小；确认 `material_ui`/`cupertino_ui`/`go_router`/`cached_network_image` 的当时最新版本与 Flutter 兼容矩阵。

## Requirements

### R1. `material_ui` / `cupertino_ui` 迁移（D-11）

- `flutter add material_ui cupertino_ui`；`dart fix --apply --code=migrate_design_widgets` 改全部 import；`lib/` 中
  `package:flutter/material.dart` 与 `package:flutter/cupertino.dart` 归零。
- `localizationsDelegates` 改用 material_ui 的 `GlobalMaterialLocalizations.delegates`（与 C 的 gen-l10n 委托并列）。
- 仍 import SDK material 的第三方 widget（`easy_refresh`、`flutter_staggered_grid_view` 等，以当时 `pub deps` 为准）经
  `MaterialApp.builder` 的 `MaterialUiCompatibilityBridge` 包住；逐个真机核对主题与本地化不错乱。
- `cached_network_image: ^4.0.0`、`go_router: ^18.0.0` 随之升级（它们已迁 material_ui）。
- 迁移后重测 arm64 release 体积并更新 B 的门禁阈值；增量 > 1 MB 时列出仍依赖 SDK material 的插件及替代评估。

### R2. Material 3 主题

- `useMaterial3` 取默认（M3）；`ColorScheme.fromSeed` 以 `FuncTokens` 品牌色为 seed；AppBar/NavigationBar/TabBar/Card/Chip/
  Dialog/BottomSheet 组件主题在 `lib/app/theme/` 单点定义。
- `ReplicaSwitchTile` 的 `CupertinoSwitch` 换 M3 `Switch`（或保留并在 `component-guidelines.md` 记录理由）。
- 重新生成 `test/goldens/*`；视觉变化写进 `component-guidelines.md`。

### R3. 导航架构（D-12）

- `MaterialApp.router` + `GoRouter`；`StatefulShellRoute.indexedStack` 分支 = 首页各 tab（推荐/排行/最新/搜索/我），每个分支
  独立 Navigator 与 `PagedFeedController` 生命周期；详情/用户/小说/评论/设置/登录等页面为子路由。
- 转场用 `CustomTransitionPage` 消费 `motion_tokens`（右滑 300ms 等与 C 一致）；`replicaRouteObserver` 改为 go_router `observers`。
- C 的 `routes.dart` 门面改为 `context.go/push` 实现，调用点不变；`Navigator.of(context).push(ReplicaPageRoute(` 归零。
- `intent_router.dart` 的 `pixiv://users|illusts|account`、`pixivfunc://`、`https://www.pixiv.net/...` 深链映射为 go_router 路径；
  `AndroidIntentChannel` 的初始 intent 与 `onNewIntent` 事件驱动 `router.go`。
- Web 登录、反查 WebView 等全屏页在根 Navigator 层（不属于任何 tab 分支）。

### R4. 状态恢复

- `MaterialApp.router(restorationScopeId:)` + `GoRouter(restorationScopeId:)`；tab 与各分支栈可恢复。
- 搜索词、排行模式、viewer 页码进路由路径/查询参数；全部 feed 用 `PageStorageKey` 保留滚动位置（在 `IndexedStack` 内）。
- 验证：开发者选项"不保留活动" + `adb shell am kill io.github.lopution.pixivfunc`，回到同 tab、同页面、同滚动区间；
  `history_persistence`/`download_recovery` 语义不受影响。

### R5. Predictive Back

- `AndroidManifest.xml` `<application android:enableOnBackInvokedCallback="true">`；`PopScope` 全部使用 `onPopInvokedWithResult`；
  go_router 与系统预测性返回联动。Android 14+ 真机可见预览动画；API 29 行为不变。

### R6. Hero 手势返回

- `lib/app/motion/drag_to_dismiss.dart`：viewer 与详情图片区域支持下拉/拖拽关闭并驱动 Hero 反向转场；复用 C 的 `HeroRectClip`。
- Ugoira viewer 同样接入；转场契约更新到 `component-guidelines.md`。

### R7. M3 组件

- `BottomNavigationBar` → `NavigationBar`；搜索入口 → `SearchBar`/`SearchAnchor`（保留现有 autocomplete controller）；
  设置中的三档枚举（预览/查看质量等）→ `SegmentedButton`；对话框与底部表单按 M3 样式。

### R8. 文档

- README 项目原则改写：视觉冻结终止，改为"Func 组件层 + 现代交互模型"；`.trellis/spec/frontend/component-guidelines.md`
  记录导航、恢复、转场、组件的新契约；`backend/release-artifacts.md` 更新体积阈值。

### R9. 不做

- `material_ui`/`go_router` 之外的 UI 或路由库；平板专用布局（自适应列数由 C 的 `IllustFeedGrid` 提供）；
  改变网络/下载/凭据语义；为恢复而新增持久化格式（只用路由参数与 `PageStorage`）。

## Acceptance Criteria

- [ ] `rg "package:flutter/material.dart|package:flutter/cupertino.dart" lib` 为 0；`useMaterial3` 为默认；legacy 插件经桥后主题正确。
- [ ] `cached_network_image` ≥ 4、`go_router` ≥ 18 已升级；arm64 release 体积已重测并更新 B 的门禁阈值。
- [ ] 全部页面经 `MaterialApp.router` + `StatefulShellRoute`；`rg "ReplicaPageRoute\(" lib/features` 为 0；切换 tab 各自返回栈保留。
- [ ] `pixiv://illusts/<id>`、`pixiv://users/<id>`、`https://www.pixiv.net/artworks/<id>` 深链经 go_router 到达正确页面（冷启动与运行中）。
- [ ] "不保留活动" + `am kill` 后回到同 tab/同页面/同滚动区间；搜索词与 viewer 页码恢复。
- [ ] Android 14+ 真机预测性返回可用；API 29 返回行为正常。
- [ ] viewer 拖拽关闭触发 Hero 反向转场；Ugoira viewer 同样可用。
- [ ] `NavigationBar`/`SearchBar`/`SegmentedButton` 落地；golden 全部重生成并通过。
- [ ] README 与 `component-guidelines.md` 已更新；`flutter analyze`、`flutter test` 全绿；parent 真机矩阵（登录、浏览、下载、
      Ugoira、widget、updater）无回退。

## Notes

- 复杂 child：`task.py start` 前需从 parent design §13.3 派生本目录的 `design.md`（路由表、分支与页面归属、恢复参数、桥覆盖清单、
  主题 token 映射）与 `implement.md`（F1–F8，每步一个提交；F1 与 F3 为显式回滚点）。
- 停止条件：`material_ui` 与 legacy 插件主题互不可见且桥无法覆盖 → 停在 F1；go_router 恢复与 `PagedFeedController` 生命周期
  冲突 → 停在 F4，先修契约。
