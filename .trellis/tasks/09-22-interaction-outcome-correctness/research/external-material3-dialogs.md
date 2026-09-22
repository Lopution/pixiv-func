# External — Material 3 对话框/按钮文案与可访问性

## 对话框动作文案

- 源：https://m3.material.io/components/dialogs/guidelines
- 要点：
  - 确认动作应写清结果（"Send"/"Create"/"Discard"），避免含糊的 "OK"/"Done"/"Close"。
  - 对话框最多两个动作：一个确认、一个 dismiss；dismissive 动作不放在确认动作右侧；dismissive 动作**永不禁用**。
  - 破坏性确认（discard）是合法模式，文案直接说后果。
- 复用点：
  - profile_edit「放弃更改？」对话框：`确认=Discard/放弃`、`dismiss=Cancel/取消` 的命名符合规范，W1 只需让无改动时不再触发它。
  - browse_settings 若选「应用并测试」改名，按钮文案同样遵循「动作=后果」原则。
  - login WebView 错误卡按钮（重新加载/重新登录/返回）属错误呈现区而非对话框，但同一命名原则适用。

## 按钮层级

- 源：https://m3.material.io/components/all-buttons
- 要点：Filled 用于流程的最终/推进动作，Outlined/Text 用于次级；一屏一个 Filled 主动作。
- 复用点：reverse-image 进度卡「取消」应为次级（Outlined/Text），不能与「离开/搜索」等主动作同级；login 错误卡「重新登录/重新加载」是推进动作可 Filled，「关闭提示」次级。

## 对话框可访问性

- 源：https://m3.material.io/components/dialogs/accessibility
- 要点：焦点进入对话框；Tab/Shift+Tab/Enter/Escape 键盘路径可用；动作按钮有可读标签。
- 复用点：`showAppDialog`（`lib/app/motion/app_overlays.dart` L36+）已是统一入口，W1 新增的确认/错误 UI 一律走它和 `material_ui` 控件，不自造 dialog；icon-only 按钮必须配 tooltip（widget-spec 规则，profile_edit/reverse-image 的 leading 按钮都涉及）。

## 断言边界

M3 是设计规范不是行为证明；「焦点顺序/键盘可达」需在 widget 测试或运行时验证，静态代码不能断言。
