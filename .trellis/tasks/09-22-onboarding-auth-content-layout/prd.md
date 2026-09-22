# 引导登录与内容布局（Roadmap W9）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图，
本 leaf 对应 design.md §4.9、追踪矩阵 §4 的 Astra #22/#23/#24/#25 与 §5 弹层、
页面宽度两条横向条目）。
规划基线：`main@8067b2d`；全部代码断言与行号以本 leaf `research/codebase-*.md`
在当前 HEAD 的逐条复核为准。
前置任务：W1（`09-22-interaction-outcome-correctness`）。两端
`login_webview*_page.dart` 与 W1 同文件——W1 修动作语义，本包做错误层级/布局，
按父 §3 同文件串行规则，R3 的 stage 必须等 W1 合入 main 后再启动。
已吸收差异：PR #48（共享 SnackBar 入口）、PR #50（详情元信息排版与
`SelectionArea` 因 ~180KB AOT cap 被否决的历史结论），本包直接消费、不重复建设。

## Goal

收敛引导、登录、授权 WebView、Spotlight 文章四类页面的表现层：引导页共用
「可滚动 + 限宽 + 钉底 CTA」shell；登录主动作恒定在场、帮助与剪贴板导入次级化；
WebView 错误层级两端一致；Spotlight 正文受控行长、段落级可选择、可分享/打开原文。
本 leaf 只做布局、层级、文案与可选择性迁移：不改 parser/OAuth/intent/网络策略语义
（父 §6：UI 只表达现有状态），不新增断点，不为未来工作包预建未引用基件（§4.11）。

## 所有权（owning files）

- `lib/features/onboarding/`：welcome_page.dart、language_page.dart、theme_page.dart、
  user_agreement_page.dart
- `lib/features/login/`：login_page.dart、login_webview_page.dart、
  login_webview_desktop_page.dart（`login_navigation_decision.dart` 纯函数不改）
- `lib/features/spotlight/`：spotlight_article_page.dart；
  spotlight_feed_page.dart 属所有权边界但列表结构保留不改
- `lib/app/startup_gate.dart` 的 `_StartupError` 表现层（路由裁决不变量不动）
- 新增：`lib/app/widgets/scrollable_form_shell.dart`（跨 feature shell，D1）、
  `lib/app/layout/` 内容限宽常量（D10）、`lib/features/login/` 内 WebView 错误卡
  共享件（feature 私有，D5）
- l10n：`lib/l10n/app_{en,ja,ru,zh}.arb` + `tool/gen_l10n_lookup.py` 产物

消费的既有 owner（不扩展、不绕行）：`AppBreakpoints`、`showAppDialog` /
`showAppBottomSheet`、`showAppSnackBar`、`shareServiceProvider` / `SharePayload` /
`shareOriginOf`、`FeedLoading` / `FeedError` / `FeedEmpty` / `FeedTail`、
`ReplicaScaffold` / `ReplicaButton` / `ReplicaSwitchTile`、`SettingsLoadError`。

明确不碰：收藏编辑（W3）、阅读设置（W5）、管理列表（W6）、设置表单（W8）的
文件与弹层；`article_parser.dart` / `oauth_service.dart` / `intent_router.dart`
等 core 语义。

## Requirements

### R1. 引导页共用 shell：可滚动 + 限宽 + 钉底 CTA

现状：`welcome_page.dart` L15-95 已是「`LayoutBuilder` + `SingleChildScrollView` +
`Center` + `ConstrainedBox(maxWidth:520)` + `Column(spaceBetween)`」，是唯一达标页；
`language_page.dart` L43-114 / `theme_page.dart` L36-140 是固定 `Column`+`Spacer`
**不可滚动**，水平 padding 用 `width*.1` 无上限（1200dp 下单侧 120dp、超宽继续变宽），
标题靠 `FittedBox(scaleDown)` 缩字兜底——320×568、横屏矮高度、1.3x 字号下溢出是
存量缺陷，不只是观感问题。

- 抽共用 shell 到 `lib/app/widgets/`：本包内 welcome/language/theme/login 共 4 个
  真实消费者，满足父 §5.5「≥3 消费者 + 同 stage 接入」才允许新增跨 feature shell。
  shell 结构：`ReplicaScaffold` 内 `SingleChildScrollView` → `Center` →
  `ConstrainedBox(maxWidth)` → `Column(spaceBetween)`；槽位 = header / content /
  钉底主 CTA / 可选 secondary；`maxWidth` 是 typed 参数（默认 520，即 welcome 现值），
  属于按角色的内容层常量，不是断点。
- welcome / language / theme 迁入 shell，删除 `w*.1` 比例 padding 与
  `Column`+`Spacer` 固定布局；矮屏/大字号下内容可滚动、CTA 始终可达。
- `FittedBox(scaleDown)` 收敛：welcome 的双行品牌 lockup 保留（L39-41 注释钉住
  「各 locale 锚点一致」的设计意图）；language/theme 标题改为允许换行的居中标题，
  不再以无限缩字维持单行（D2）。
- 'later' 槽（language L99-108、theme L125-133 的 44px 纯文本
  「稍后可在设置中变更」）改为 shell secondary 槽内的真实次级按钮（D3）。
- `user_agreement_page.dart`（L17-80 全宽 `ListView`、正文 `Text` 不可选）：
  保留 `ListView`，外层加居中 + 文章角色限宽；正文 `Text` → `SelectableText`。
  此项不在 Astra 条目内，属 §4.9「这些页面的布局迁移」覆盖范围，随包做掉。

### R2. 登录页：主动作恒定存在，帮助/剪贴板次级化

现状：`_buildLoginActions`（login_page.dart L391-447）在 `_help` 展开时把
注册/登录主按钮**整组替换**为剪贴板入口（Astra #24）；`getMoreHelp` span
（L332-339）染 primary 色但无 recognizer，是「可见但不可点」的死链接样式；
正文固定 `Column` + `height*.4` 硬高度（L279-286）不可滚动；`_clipboardBusy`
只防重入、按钮无 busy 可视态（`ReplicaButton` 无 loading 变体）。

- 正文迁入 R1 的 shell（同 520 限宽、可滚动、钉底动作区），删除 `height*.4`
  硬高度与 `w*.1` padding。
- 动作区重构为恒定分层：**主行 = 注册（描边）+ 登录（实心）永远在场**；
  `_help` 展开时 `networkCompatibilityHint` 文本 + 「使用剪贴板数据登录」次级按钮
  追加在主行下方，不替换主行；`accountTransferWarning` caption 紧随剪贴板入口。
  断言形态：Given `_help` 已展开，When 页面渲染，Then `register` 与 `login`
  两按钮仍存在且 `onPressed` 非空。
- `getMoreHelp` 死链接删除该 span（D4）；`networkCompatibilityHint` 保留为纯文本。
- `_clipboardBusy` 期间剪贴板按钮禁用 + label 区显示小尺寸进度指示
  （父 §5.3：剪贴板导入 = action，需 idle/busy/success/error 可辨）。
- `loginAgree` + `userAgreement` 链接（L288-303）上移入 shell secondary/正文槽。
- 外部浏览器登录明确不做（D9）。

### R3. WebView 错误层级两端统一（W1 合入后启动）

现状：两端各有一份逐行复制的左下 `Card(errorContainer)`（mobile L285-318 ≈
desktop L277-307）；文案形态不一致（desktop 错误文本带完整 URL，mobile 只带码）；
mobile 注册模式无进度条（`onProgress` 只在登录分支 L87-89），desktop 两模式都有；
desktop 独有 `_webView2Missing` 全页态（真实平台状态）。

- 两端错误卡抽 `lib/features/login/` 内共享 widget（同 feature 私有，不进组件层）：
  统一位置（左下 SafeArea 内）、Card 结构、动作区布局（D5）。
- **动作集与动作语义以 W1 合入后的契约为准**，本包只统一结构、层级与文案形态，
  不改动作后果；s4 启动时的逐项复核点见 `implement.md` 阶段 4。
- 错误文案参数形态两端拉齐：统一带 host、不带完整 query（避免长 URL 撑爆卡片）。
- mobile signup 模式补 `onProgress` → 两端都显示进度条（对 signup 无副作用）。
- desktop `_webView2Missing` 全页安装提示保留，不属于层级差异。

### R4. Spotlight 文章：受控行长 + 可选择 + 分享/打开原文

现状：`spotlight_article_page.dart` body 为全宽 `ListView`（L59-61，padding 16，
1200dp 下每行 ~1168dp）；标题/描述/段落/小标题全部不可选；AppBar（L42-48）
无 actions。`_linkSpans`（L144-175）已是纯 `TextSpan` + `TapGestureRecognizer`
结构（L141-143 注释记录 WidgetSpan+GestureDetector 曾破坏选择/读屏）。

- 正文限宽：`ListView` 外层 `Align(topCenter)` + `ConstrainedBox(maxWidth)`，
  取值 ~680–720dp 写成命名常量并注释换算依据（bodyMedium 14sp ≈ 45-55 CJK 字/行，
  见 `research/external-m3-adaptive-layout.md`）。图片、卡片同列限宽；
  v1 不做大图破栏。
- 逐块可选择：`SpotlightParagraph` 的 `Text.rich` → `SelectableText.rich`
  （`_linkSpans` recognizer 结构不变，`SelectableText.rich` children 只许
  `TextSpan`，现状已符合）；标题/描述/heading `Text` → `SelectableText`。
  **不用 `SelectionArea`**——它独拉 SelectableRegion/context-menu 机制，
  armeabi-v7a AOT 超 ~180KB cap，已被 PR #50 / `324f3d6` 否决
  （info_block.dart L102-105 注释）。选择范围 = 段落级，跨段选择不可得；
  `_SpotlightIllustCardView` 内文字保持不可选（卡片是导航件）。
- AppBar 增加 `share` 与 `open_in_new` IconButton（或合入 overflow menu）：
  分享走 `shareServiceProvider.share(SharePayload(title: resolvedTitle,
  author: 'pixivision', url: _url), sharePositionOrigin: shareOriginOf(context))`，
  `copiedToClipboard` 时 `showAppSnackBar(linkCopied)`；打开原文走
  `launchUrl(Uri.parse(_url), mode: externalApplication)`（文件已 import
  url_launcher）。
- 新 l10n key：`share`、`openInBrowser`（四语言 arb + `lookup.dart` 重生成）；
  `linkCopied` 复用。
- `spotlight_feed_page` 列表结构保留，不做限宽（Astra：「列表基本结构可保留」）；
  `_category` 页面本地 state 保持。

### R5. 本包自有弹层语义复核

本包唯一弹层是 login_page.dart L135-151 的代理提示（已走 `showAppDialog` +
`AlertDialog`，cancel/continue 动作完整）。`showAppDialog` 在 compact 与宽屏同形，
信息顺序与动作语义天然一致，满足父 §4.9「手机与宽屏保持相同信息顺序和动作语义，
不强求相同位置」。默认保持 dialog 全断点（D7）；只有一个消费者，不扩展共享
overlay owner。

### R6. `_StartupError` 附带修（startup_gate.dart L270-307）

retry 从 `TextButton` 升级为与 `FeedError` 同级的 `FilledButton` 主操作，
内容限宽居中；不引入新组件。`SplashPage` / `_StartupProgress` 无改动；
`StartupGate` 的 allowlist 与「重定向中保持 child 挂载」不变量不动。

## Acceptance Criteria

- [ ] shell 页 widget test 矩阵：320×568、390/600/840/1200dp、横屏矮高度
      （如 640×320）、`textScaler` 1.3 + ru/en 长翻译——无 overflow exception、
      主 CTA 始终可见可达；`language`/`theme` 标题不再用 `FittedBox` 缩字。
- [ ] 'later' 次级按钮存在且触发与各自 `next` 相同的推进路由；
      `user_agreement` 宽屏正文被 cap 且段落可选择。
- [ ] 登录页：`_help` 展开时 `register`/`login` 两按钮仍存在且可点（widget test
      断言）；剪贴板 busy 态可辨（禁用 + 进度指示）；`getMoreHelp` 不再有链接样式；
      网络开关 immediate 语义与失败 SnackBar 不回归。
- [ ] WebView：mobile widget test（现有 `_FakeWebViewPlatform`）断言
      recoverable/fatal 两态动作集与 W1 契约一致且两端同构；desktop 按 quality
      spec 的 `InAppWebViewPlatform` fake 补 smoke test；WebView2 缺失态标"未验证"。
- [ ] Spotlight：`SelectableText` / `SelectableText.rich` 存在、链接 recognizer
      仍触发 `openIllust` / `launchUrl` 路径；840/1200dp 下正文宽度被 cap；
      分享失败走剪贴板 + `linkCopied` SnackBar。
- [ ] 弹层门禁：代理提示全断点 dialog 的信息顺序与动作语义一致（§4.9）；
      immediate（网络开关）/ action（剪贴板 busy）/ confirm（代理提示）
      三类状态分别有断言。
- [ ] `startup_gate_test`、`login_navigation_test`、`spotlight_article_test` 等
      现有断言不回归。
- [ ] 新 l10n key 四语言齐 + `lookup.dart` 重生成；随消费点移除不再使用的
      `later` / `getMoreHelp` key。
- [ ] 每 commit 对应 `implement.md` 一个勾选框；分支按 stage 拆分规则命名
      （`task/09-22-onboarding-auth-content-layout-s<N>`）。
- [ ] `flutter analyze --no-pub`、聚焦 + 全量 `flutter test`、`git diff --check`、
      `task.py validate` 全绿；真机/桌面不可验项在 PR 显式标"未验证"。

## Decisions（规划定案）

- D1 shell 放 `lib/app/widgets/scrollable_form_shell.dart`：本包 4 个真实消费者
  （welcome/language/theme/login）满足 §5.5 的 ≥3 + 同 stage 接入；welcome 为原型。
  命名避开 layering_test 检查的 `_*Tail/_*Error/_*Empty/_*Card/_*Status/_*Placeholder`
  命名族。
- D2 `FittedBox(scaleDown)` 收敛策略：welcome 保留双行品牌 lockup（locale 锚点
  注释背书）；language/theme 标题改允许换行的居中标题，验收以 1.3x 字号 +
  ru/en 长翻译不丢可达性为准。
- D3 'later' 纯文本改次级按钮：新 l10n key `setupLater`（暂不设置/稍后设置，
  四语言），`TextButton` 入 shell secondary 槽，onPressed 与各自 `next` 同效
  （language → `/welcome/theme`；theme → `completeGuide()` + `openLogin`）。
  语义 = 「跳过本步、稍后可在设置中变更」的真实 affordance；原说明句的语义并入
  按钮标签，`later` key 随消费点移除清理。
- D4 `getMoreHelp` span 删除：它没有跳转目标，保留染色样式 = 「可见但不可点」
  缺陷；`networkCompatibilityHint` 保留为纯文本，`getMoreHelp` key 清理。
- D5 WebView 错误卡为 `lib/features/login/` 内共享件（2 个同 feature 消费者，
  不满足跨 feature 规则，不进 `app/widgets`）；错误文案统一带 host 不带完整
  query；动作语义以 W1 合入为准，W9 不做动作后果改动。
- D6 Spotlight 正文逐块 `SelectableText`/`SelectableText.rich`（不用
  `SelectionArea`，PR#50 ~180KB AOT cap）；行长常量 ~680–720dp 注释依据；
  `SharePayload` 裸构造 `author: 'pixivision'`，不加 factory（core 改动不值，
  pixivision 文章无真作者槽）。
- D7 代理提示 dialog 全断点保持 `showAppDialog`：信息顺序/动作语义天然一致、
  单消费者不值得页面级 sheet 分支，不扩展共享 overlay owner。
- D8 `_StartupError` retry 升 `FilledButton` + 限宽：本包文件所有权内的低成本
  一致性收益，随 s1 做掉。
- D9 外部浏览器登录明确不做：`pixiv://account` 回流通道虽存在，但 PKCE verifier
  只在 `OAuthService._session` 内存里，WebView dispose 即 `discardSession`，
  外部完成授权会 "no live session" 失败；支持它需要会话存续策略改动，超出
  布局包范围。
- D10 断点只用现有 `AppBreakpoints`——它只有 `compact`/`medium` = 600 与
  `expanded` = 1200 两档有效断点（与 M3 的 600/840/1200/1600 不同，不得以 M3
  数值替换）。本包按现有档接入，不新增断点；内容限宽是按角色的常量
  （form/引导 520、article ~680–720），放 `lib/app/layout/` 的内容宽常量文件
  并注释「内容宽 ≠ 断点」，不塞进 `app_breakpoints.dart`。

## Out of scope

- W3/W5/W6/W8 各自拥有的页面、表单与弹层；任何 core 语义改动
  （parser / OAuth / intent / 网络策略 / 持久化 schema）。
- 外部浏览器登录（D9）；`SelectionArea` 整页选择（D6）；Spotlight feed 列表
  结构改版；新断点、新 overlay 入口、第二套 SnackBar/反馈通道；`TwoPane`
  （本包均为单栏限宽场景）。
- WebView 错误卡的动作后果本身——reload/重新登录/返回的语义归 W1，本包只在
  W1 契约上做结构与层级统一。
