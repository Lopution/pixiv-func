# Implementation Draft — W9 onboarding-auth-content-layout

逐条对应父任务 design.md §4.9。基线 `main@8067b2d`；前置 W1 未合入（见 risks.md）。所有行号见同目录 codebase-*.md。

## 范围边界（先钉死）

- 拥有：`features/onboarding/`（4 页）、`features/login/`（4 文件）、`features/spotlight/`（2 页）、`app/startup_gate.dart` 的表现层、以及这些页面自有 dialog。
- 不碰：收藏编辑（W3）、阅读设置（W5）、管理列表（W6）、设置表单（W8）；`article_parser.dart`/`oauth_service.dart`/`intent_router.dart` 等 core 语义不改（§6：UI 只表达现有状态）。
- 消费现有 owner：`AppBreakpoints`、`showAppBottomSheet`/`showAppDialog`、`showAppSnackBar`、`shareServiceProvider`、`FeedLoading/FeedError/FeedEmpty/FeedTail`、`ReplicaScaffold/ReplicaButton/ReplicaSwitchTile`、`SettingsLoadError`。

## ① 引导 shell：可滚动 + 限宽 + 稳定下一步区

**现状**：welcome 已是"scroll+520 限宽+spaceBetween 钉底 CTA"（L23-89）；language/theme 是固定 Column+Spacer（不可滚动、`w*.1` 比例 padding 无上限、FittedBox 缩字）。

**方案**：
1. 抽一个共用 shell。消费者清点：welcome、language、theme、（可含 login 正文）= 本包内 ≥3 个真实消费者，满足"新增跨 feature shell 需 ≥3 消费者 + 同 stage 接入"的条件 → 允许放 `lib/app/widgets/`（如 `scrollable_form_shell.dart`）。若只想收敛引导三页、登录页自己写等价结构，则放 `features/onboarding/widgets/` 即可且不动组件层。**建议前者**：登录页与本包引导页同结构，避免两份近似实现（W10 会查平行组件）。
   - shell 契约草拟：`ReplicaScaffold` 之内 `SingleChildScrollView` → `Center` → `ConstrainedBox(maxWidth: 520)` → `Column(minHeight: viewport-padding, spaceBetween)`；header 槽、content 槽、固定底部 action 槽（±secondary 文本槽）。welcome 的 minHeight trick（L19-30）是现成实现参考。
   - 命名避开 layering_test 检查的 `_*Tail/_*Error/_*Empty/_*Card/_*Status/_*Placeholder` 族。
2. 标题：`FittedBox(scaleDown)` 改允许换行的居中标题（或 maxLines:2 仍 scaleDown）。welcome 的双行 lockup 有"各 locale 锚点一致"注释背书（L39-41）——**决策点**：保留 welcome 的 FittedBox 作品牌锁区块，只统一 language/theme 的标题行为；或三页统一为多行不换字号。验收要过 1.3x 字号 + ru/en 长翻译。
3. language/theme 的 `Spacer` 布局改为 shell 内自然流：选项列表在上、`next` 按钮在稳定底槽、'later' 说明文字保留（它是纯信息文案"稍后您可以在设置中变更"，不是按钮——不要给它加假 affordance）。
4. `user_agreement_page`：全宽 ListView → 同 shell/同文章限宽；正文 `Text`→`SelectableText`（文档页成本低、收益直接）。此项不在 Astra 条目里，属于 §4.9"这些页面的布局迁移"覆盖范围，建议随包做掉。

**验收**：320/390/600/840/1200dp + 横屏矮高度 + 1.3x 字号无 overflow、CTA 始终可见可达；`startup_gate_test` 现有断言不回归。

## ② 登录页：主动作始终存在，帮助/剪贴板为次级

**现状**：`_help` 分支整组替换注册/登录按钮（login_page L395-421，Astra #24）；`getMoreHelp` 是染色的死链接样式（L332-339）；中部固定 `height*.4` + 整页不可滚动（L279-307）。

**方案**：
1. 正文迁入 ① 的 shell（同 520 限宽、可滚动、底槽稳定）。
2. `_buildLoginActions` 重构为**分层恒定结构**：主行 = 注册（描边）+ 登录（实心）永远在场；次级区 = ℹ️ 帮助展开的内容 + 剪贴板导入入口，作为独立行/块出现，**不替换**主行。
   - 建议：`_help` 展开时，hint 文本 + "使用剪贴板数据登录"以次级按钮（TextButton/OutlinedButton 或次级 ReplicaButton）追加在主行下方；`accountTransferWarning` caption 紧随剪贴板入口。
   - `getMoreHelp` 要么删要么真做事：**建议删除该 span**（它没有目标），或让它等价于关闭帮助——不要保留不可点的链接样式（component-guidelines：可见但不可点 = 缺陷）。
3. `_clipboardBusy` 需要可视态：ReplicaButton 无 loading 变体 → busy 时 `onPressed` 置空 + label 区换小尺寸进度指示（或禁用态），避免双击无反馈。属 action 状态（§5.3 idle/busy/success/error）。
4. 底槽放 `loginAgree`+`userAgreement` 链接（现 L288-303 上移进 shell 的 secondary 槽）。

**验收**：widget test 断言 `_help` 展开时 register/login 两按钮仍存在且可点；剪贴板 busy 态可辨；现有 `login_navigation_test`（policy 开关、settings error）不回归。

## ③ WebView 错误层级两端统一（W1 合入后做布局层）

**现状**：两端各有一份逐行复制的左下 `Card(errorContainer)`（mobile L285-318 / desktop L277-307）；fatal→"重新打开"实为 `pop(false)`，recoverable→"知道了"清除，无 reload；文案细节不一致（desktop 错误带 URL，mobile 不带）；mobile 注册模式无进度条（onProgress 只在登录分支 L87-89）、desktop 两模式都有。

**方案**：
1. **抽同 feature 内共享 widget**：`features/login/widgets/`（或文件内共有）`LoginWebViewErrorCard{message, fatal, onPrimary, onDismiss}`——同 feature 私有组件，不触发跨 feature 规则。两端都改为用它，消掉复制。
2. 布局统一：同位置（左下 SafeArea 内）、同 Card 结构、同动作区；动作语义以 W1 合入后的契约为准（预期：recoverable 有明确 reload、fatal 的"重新打开"重建会话而非仅 pop——**W9 只负责层级/布局一致，动作后果归 W1**；若 W1 未合入或语义未定，本项保持现语义只做结构统一并在 PRD 写明）。
3. 消息统一：`loginPageLoadFailed`/`loginNetworkError` 的参数形态两端拉齐（是否带 URL 二选一——建议带 host 不带全 query，避免 URL 过长撑爆卡片；属文案细节决策点）。
4. desktop 独有 `_webView2Missing` 全页态保留（平台真实状态，不是层级差异）。
5. 注册模式进度条两端对齐（都显示或都不显示；建议都显示——`onProgress` 对 signup 无副作用）。

**验收**：widget test（mobile 已有 fake 模式）断言 recoverable/fatal 两态动作集两端一致；desktop 端按 quality spec L233-248 的 InAppWebView fake 补一个 smoke test；真机/WebView2 检查标记"未验证"若无设备。

## ④ Spotlight 文章：行长 + 可选择 + 分享/打开原文

**现状**：全宽 ListView（L59-81）、全文不可选、AppBar 无 actions（L42-48）。

**方案**：
1. **限宽**：`body` 的 ListView 外层 `Align(alignment: topCenter) + ConstrainedBox(maxWidth: ~680–720)`（padding 16 内收）。图片、卡片同列限宽即可（fit:contain 已自适应）；若想让大图突破文字栏宽，属可选增强，建议 v1 同栏。
   - 数值依据：external-m3 文档——14sp 中文 ~45-55 字/行 ≈ 640-760dp；取 680 或 720 并写注释说明。
2. **可选择**：逐块 `SelectableText`/`SelectableText.rich`：
   - `SpotlightParagraph`：`Text.rich`→`SelectableText.rich`，`_linkSpans` 的 recognizer 结构不变（Flutter #54479 后 span recognizer 在 selectable 中有效；注意 #168864 的 tap/长按竞争需真机验证）。
   - heading/title/description：`Text`→`SelectableText`。
   - **不用 `SelectionArea`**（AOT ~180KB cap，PR#50 已否决；见 external 文档）。范围 = 段落级选择，跨段选择不可得，PRD 写清。
   - 风险备查：SelectableText 在 ListView 内长按弹 context menu；确认 `illust card` 内文字保持不可选（卡片是导航件）。
3. **AppBar actions**：`IconButton(share)` + `IconButton(open_in_new)`（或合进 overflow menu）。
   - 分享：`shareServiceProvider.share(SharePayload(title: resolvedTitle, author: 'pixivision', url: _url), sharePositionOrigin: shareOriginOf(context))`；`copiedToClipboard`→`showAppSnackBar(linkCopied)`。`SharePayload` 裸构造是 public——可直接用；若要 `SharePayload.spotlight()` factory 让 "#Pixiv" 前缀语义更准，属 core 小改，**决策点**（author 槽对 pixivision 文章没有真作者，建议传 'pixivision' 或改用 title+url 格式）。
   - 打开原文：`launchUrl(Uri.parse(_url), mode: externalApplication)`（文件已 import url_launcher）。
   - 新 l10n：通用 `share`、`openInBrowser` 四语言 + `lookup.dart` 映射（现有 `cardActionShare`/`linkCopied` 可复用）。
4. **feed 页**：§4.9 只点名"文章"——列表结构保留；可选把 `SliverPadding` 列表限宽 ~840（管理列表风格），建议不做（Astra："列表基本结构可保留"）。`_category` 是本地 state 可保持。

**验收**：widget test 断言 `SelectableText` 存在、链接 recognizer 仍触发 `openIllust`/`launchUrl` 路径；840/1200dp 下正文宽度被 cap；分享失败走剪贴板+SnackBar；`spotlight_article_test` 现有断言不回归。

## ⑤ 本包 dialog 的手机 sheet / 宽屏 dialog 语义

**现状**：本包仅一个弹层——login_page L135-151 代理提示，已走 `showAppDialog`+`AlertDialog`（cancel/continue 动作完整）。

**方案**：`showAppDialog` 在 compact 与宽屏同形 = 信息顺序与动作语义天然一致，满足"不强求相同位置"。**建议 v1 保持 dialog 全断点**。若产品要手机改 sheet：页面级分支 `width >= AppBreakpoints.medium ? showAppDialog : showAppBottomSheet` 即可，无需扩展共享 owner（只有一个消费者）；但要保证按钮顺序/默认焦点一致并各跑一次 widget test。列入决策点，默认不做。

## ⑥ 附带项（本包文件所有权内、低成本）

- `startup_gate.dart` `_StartupError`（L270-307）：TextButton retry → 与 FeedError 同级的 FilledButton 主操作、内容限宽；不引入新组件。
- `SplashPage`/`_StartupProgress`：无改动需求。

## 建议 stage 拆分（供 implement.md 参考）

- stage-1：引导 shell + 三页迁移 + user_agreement（自包含，无依赖）。
- stage-2：登录页分层 + 代理 dialog 复核。
- stage-3：WebView 错误卡统一（**等 W1 合入**后定动作文案）。
- stage-4：Spotlight 文章（限宽/可选择/分享/打开原文）+ 新 l10n。
每 stage 独立分支 `task/09-22-onboarding-auth-content-layout-<stage>`。

## 门禁对照（implement.md §6 "W8/W9"）

- immediate/draft/action/destructive 状态测试：网络开关=immediate、剪贴板导入=action(busy 可视)、代理提示=confirm——分别补断言。
- 320/390/600/840/1200 + 横屏 + 大字号：shell 页 widget test 用 `tester.view.physicalSize` 矩阵断言 CTA 可见、无 overflow exception。
- Spotlight 正文可选择 + 登录主动作不消失：已有对应断言计划。
- sheet/dialog 语义顺序一致：dialog 全断点同形天然满足。
