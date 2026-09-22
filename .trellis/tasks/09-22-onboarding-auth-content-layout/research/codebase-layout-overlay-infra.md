# Codebase: 断点 / Overlay / 反馈基础设施 + 宽屏现状（W9 调研）

基线：`main@8067b2d`。

## 断点 — lib/app/layout/app_breakpoints.dart（28 行）

- `compact = 600`（L7）：以下用底部 NavigationBar。
- `medium = 600`（L11）：以上 shell 换 NavigationRail。
- `expanded = 1200`（L15）：以上 rail 变 extended、详情页可双栏。
- 方法：`useNavigationRail` L18、`useExtendedRail` L23、`useTwoPaneDetail` L27。
- **注意与 M3 标准不同**：M3 expanded 是 840-1199、large 1200-1599；本仓库只有 600/1200 两档。§5.5 明确 viewport 断点以此文件为唯一来源，叶子不得自造近似常量；**内容最大宽度是按角色定的内容层常量，不是断点**。
- 现有消费点：`home_page.dart` L133-138（rail/extendedRail）、`branch_slide_stack.dart` L410、`illust_detail_page.dart` L403（`useTwoPaneDetail`→`TwoPane`）。

## 双栏 — lib/app/layout/two_pane.dart（40 行）

`TwoPane{primary, secondary, primaryFraction=0.55}`：Row+Expanded(flex)+VerticalDivider(1px)。注释要求经 `useTwoPaneDetail` 门控。W9 的引导/登录/文章页均为单栏限宽场景，**用不到 TwoPane**。

## Overlay — lib/app/motion/app_overlays.dart（53 行）

- `showAppBottomSheet<T>` L9-31：`showModalBottomSheet` 封装，`MotionTokens.enabled` 正常 → `MotionTokens.sheet`+`sheetCurve`，reduced motion → `AnimationStyle.noAnimation`（状态仍落地只去掉位移）。参数透传 isScrollControlled/useSafeArea/showDragHandle。
- `showAppDialog<T>` L36-52：`showDialog` 封装，同上 reduced-motion 门控（`MotionTokens.dialog`）。
- **两函数没有宽度自适应逻辑**：sheet 恒为底部 sheet，dialog 恒为居中 dialog。"手机 sheet / 宽屏 dialog"的语义切换目前**不存在共享实现**——§5.5 只允许"经现有入口统一"，W9 若需要按宽度换形态，要么扩展 owner（需同 stage 接入真实消费者，且 §5.5 要求≥3 个已明确消费者才许新增跨 feature helper），要么在包内页面级选择。
- 现有消费者（grep 全 lib，共 18 个调用点）：`showAppDialog` —— settings_page L213、account_settings L151、about_settings L347、login_page L135、user_page L260/L544、profile_edit L146、comments L108/L242、backup_settings L99；`showAppBottomSheet` —— search_filter_sheet L15、bookmark_switch_button L46、novel_page L458/L537、card_action_sheet L13、history_page L461、follow_switch_button L38。**本包自有仅 login_page L135 一处 dialog**。

## 反馈 — lib/app/widgets/app_snack_bar.dart（92 行）

`buildAppSnackBar`（floating、统一 margin、animation style）+ `showAppSnackBar`（branch root 时抬升 bottom nav 高度）+ `showAppSnackBarOn`。§5.6：叶子不得新增平行反馈通道。登录页 L112-128、L174-176、L191-197 已全走该入口。

## Replica 组件族（引导/登录共用视觉语言）

- `ReplicaScaffold`（44 行）：AppBar(canPop→`maybePop`，无 canPop 不显示 leading)+SafeArea(top:false) body。
- `ReplicaButton`（48 行）：唯一 CTA 形态，**无 busy/loading 态**（剪贴板导入 busy 只防重入不可见）。
- `ReplicaSwitchTile`（35 行）：行 tap 即切换。
- `SettingsLoadError`（53 行）/`FeedLoading`/`FeedError`/`FeedEmpty`/`FeedTail`（feed_states.dart 212 行）：共享 loading/error/empty，文案为 required 参数（英文泄漏防线）。

## 宽屏现状扫描（哪些页面全宽）

grep `maxWidth:|ConstrainedBox|width*.` 全 lib 结果（已核对）：

| 页面 | 现状 |
|---|---|
| welcome_page L18/L26-30 | padding `w*.1` clamp[24,48] + `maxWidth:520` —— **唯一有限宽的页面** |
| language_page L48-52 / theme_page L41-47 | `w*.1` 比例 padding，无上限（1920dp→192dp 边距、1536dp 内容） |
| login_page L271-273 | 同上 `w*.1` 无上限 |
| user_agreement_page L17-18 | 全宽 ListView，固定 24dp padding |
| spotlight_feed L120 / spotlight_article L61 | 全宽列表/正文 |
| bookmark_switch_button L243、reverse_image_search_page L495 | 局部 ConstrainedBox（弹层/图片高度），非页面限宽 |
| 其余 feature 页（settings、profile…） | 未见页面级 maxWidth——属 W3/W5/W6/W8 各自范围 |

正文可选择性：`SelectableText` 仅 3 处（info_block L106 作品 ID、user_page L447/L554、frame_probe L77）；**`SelectionArea` 全 app 零使用**——PR#50/`324f3d6` 注释（info_block L102-105）：SelectionArea 会独拉 SelectableRegion/context-menu 机制，armeabi-v7a AOT 超 180KB cap，故只能用 `SelectableText`（含 `.rich`）逐块做可选择。

## Spec 约束（.trellis/spec/frontend/）

- directory-structure：`features/` 不得自建 `_*Tail/_*Error/_*Empty/_*Card/...` 命名族（layering_test 检查）；共享 widget 需 ≥3 处重复才入 `app/widgets/`；页面间导航必须走 `routes.dart` 门面。
- component-guidelines：文案必须经 generated l10n/lookup（共享 widget 文案 required 参数）；`PopScope.onPopInvokedWithResult`（无 WillPopScope）；SafeArea 规则（edge-to-edge chrome 时 inset 在 surface 内）。
- quality-guidelines：`material_ui` 阴影 Flutter material（测试 import 规则）；InAppWebView/webview_flutter widget test 需要 platform fake；`flutter analyze --no-pub` 零 issue。
