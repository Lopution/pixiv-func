# External — Flutter PopScope / 预测性返回 / Navigator.pop 语义

## 1. 预测性返回与 PopScope

- 源：https://docs.flutter.dev/platform-integration/android/predictive-back
- 源：https://docs.flutter.dev/release/breaking-changes/android-predictive-back
- 要点：
  - `PopScope.canPop` 必须在返回手势发生**之前**确定（预测性返回需要预先知道该路由能否弹）；动态判断必须放 `onPopInvokedWithResult`。
  - `canPop:false` 时系统返回触发 `onPopInvokedWithResult(didPop:false, result)`；官方示例就在回调里弹确认对话框，确认后调 `Navigator.of(context).pop()` 强制离页。
  - `WillPopScope` 已废弃，`onWillPop` 时机太晚无法支持预测动画。
- 复用点：本库 `novel_page`/`profile_edit`/`home_page` 的 `canPop:false + onPopInvokedWithResult` 结构符合官方模式；novel_page 的 bug 是显式按钮也走 `maybePop`（会被自己的 canPop 拦），改 `pop()` 即与官方「确认后强制 pop」示例同构。

## 2. maybePop vs pop 的分派差异

- 源：https://api.flutter.dev/flutter/widgets/Navigator/pop.html
- 源：https://api.flutter.dev/flutter/widgets/Route/popDisposition.html
- SDK 源码核实（flutter 3.47.2，`widgets/navigator.dart` L5605–5670 区间）：
  - `maybePop` 先问 `route.popDisposition`：`doNotPop` → 不弹，回调 `onPopInvokedWithResult(false)`；`pop`/`bubble` → 继续。
  - `pop` 直接走 `onPopPage`/`entry.pop(imperativeRemoval: true)`，**不征询** `popDisposition`；go_router 页路径 `_handlePopPageWithRouteMatch` → `route.didPop`（delegate.dart L140/152），随后仍以 `didPop:true` 触发 `onPopInvokedWithResult`。
  - flutter/flutter#163052 确认：命令式 pop 也会触发 `onPopInvoked`（didPop=true）——所以 PopScope 回调里所有「系统返回专属」动作必须写 `if (!didPop)`。
- 复用点：这是 W1 两条修复（novel 显式返回、reverse-image leading）正确性的根基——「显式离开=pop()，系统返回拦截=PopScope」。

## 3. 嵌套导航

- 源：https://docs.flutter.dev/cookbook/effects/nested-nav
- 要点：嵌套 Navigator 场景系统返回默认作用于最内层可弹栈；`routes.dart` 里 `/novel/:id`、`/reverse-image`、`/me` 是 root navigator 上的 GoRoute（L906–939 区间），home 五个 tab 在 StatefulShellBranch——返回键作用域静态上已正确（novel 页 pop 回 branch 栈），运行时需在真机/模拟器验证预测性返回动画路径（记入未验证项）。

## 4. 断言边界

以上均为官方文档+SDK 源码级事实；不证明运行时手势/动画行为，预测性返回的真机表现须列「未验证」。
