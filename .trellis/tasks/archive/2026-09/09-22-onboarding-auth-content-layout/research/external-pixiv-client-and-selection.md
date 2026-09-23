# External: PixEz/官方登录引导流程 + Flutter 可选择文本事实

调研日期：2026-09-22。

## PixEz Flutter 引导/登录流程（Notsfsssf/pixez-flutter）

公开教程与仓库一致描述的顺序：

1. 首次启动：选语言 → 确认网络环境（直连/代理能力声明）→ 登录页。
2. 登录页点击"登录"先弹提示（网络环境/代理须知），确认后进 OAuth WebView。
3. WebView 工具区除刷新外提供**"使用浏览器打开"**：系统浏览器完成登录后跳 `pixiv://` 回 App 完成回调。
4. 账号迁移：PixEz 的"长按头像复制账号数据"产出剪贴板文本；本仓库 `useLoginWithClipboardHint`（"或使用\n长按头像复制账号数据"）即兼容该格式，`accountTransferService.importFromClipboard` 是消费者。

与本仓库现状的差异点（供分层决策，不是要求照抄）：

- PixEz 把"浏览器打开"放 WebView 页内作逃生口；本仓库 `pixiv://account` intent → `/login/callback` 通道已存在（intent_router L265-282、routes L1082-1083），但 PKCE verifier 是内存态、WebView dispose 即 `discardSession`——若把授权页放外部浏览器且 WebView 页被销毁，回调会 "no live session" 失败。**要支持外部浏览器登录，需要会话存续策略改动，超出布局包范围，列为风险/待决策。**
- 官方 pixiv app：登录/注册本身走系统浏览器（Chrome Custom Tab/Safari）回跳 `pixiv://account`——同样是"主流程在外部浏览器"的模型。

## Flutter 可选择正文的技术事实

- `SelectableText.rich`（api.flutter.dev/flutter/material/SelectableText/SelectableText.rich.html）支持 `TextSpan` 树，但 **`children` 只允许 `TextSpan`**（不能有 WidgetSpan）——本仓库 `_linkSpans` 已是纯 TextSpan 结构，符合。
- span 上的 `TapGestureRecognizer` 在 selectable 文本中可用：flutter/flutter#43494 由 PR #54479 修复（recognizer 接入 RenderEditable/RenderParagraph 选择体系）。**保留风险**：较新 issue #168864 报告 TapGestureRecognizer 与长按选择/contextMenuBuilder 的手势竞争在某些版本仍有问题——需在真机验证"点链接 vs 长按选择"共存。
- `SelectionArea` 方案已被仓库否决：info_block.dart L102-105 记录其独拉 SelectableRegion/context-menu 机制，armeabi-v7a AOT 超 ~180KB cap（PR #50, commit 324f3d6）。**Spotlight 正文可选择只能按块用 `SelectableText`/`SelectableText.rich`**，不能包整页 SelectionArea；这也意味着跨段落选择、图片穿插区选择不可得——验收时应明确"段落级可选择"为范围。
- 等价替代：`SelectionArea` 若未来真需要整页选择，需重测 APK size gate（backend/release-artifacts.md 是 gate owner）。

## 参考链接

- https://github.com/Notsfsssf/pixez-flutter
- https://www.himiku.com/archives/pixez-flutter.html（登录流程截图描述）
- https://api.flutter.dev/flutter/material/SelectableText/SelectableText.rich.html
- https://github.com/flutter/flutter/issues/43494 / PR #54479 / issue #168864
