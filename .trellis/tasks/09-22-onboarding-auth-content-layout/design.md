# 技术设计：引导登录与内容布局（W9）

设计基线：`main@8067b2d`。逐文件精确改动方案、现状行号与代码草案见
`research/implementation-draft.md` 与 `research/codebase-*.md`；决策点、风险与
运行时证据缺口见 `research/risks.md` 和 `research/external-*.md`。
本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

按 AGENTS.md 的 leaf stage 规则拆 4 个分支
`task/09-22-onboarding-auth-content-layout-s{1..4}`。s1–s3 不依赖 W1，
可在 W1 合入前推进；**s4 触碰 `login_webview*_page.dart`（与 W1 同文件），
必须等 W1（`09-22-interaction-outcome-correctness`）合入 `main` 并 rebase 后
启动**——父 §3 同文件串行规则禁止并行分支。WebView stage 固定排最后。

| stage | 范围（PRD 条目） | 依赖 | 风险 |
|---|---|---|---|
| s1 | R1 引导 shell + 四页迁移 + R6 `_StartupError` | 无 | 中（矮屏/大字号溢出是真实回归风险） |
| s2 | R2 登录页分层 + R5 代理 dialog 复核 | s1 的 shell | 中 |
| s3 | R4 Spotlight 文章 + 新 l10n | 无 | 中（#168864 手势竞争只能真机验） |
| s4 | R3 WebView 错误层级统一 | **W1 合入** + s2（文件相邻） | 高（同文件串行、桌面端不可验） |

若评审粒度允许，s1–s3 可合并为一条分支顺序提交；s4 必须独立。
W1 当前仍为 `planning`——若 s1–s3 完成时 W1 未合入，s4 挂起等待，
不得先于 W1 对 `login_webview*_page.dart` 做任何改动。

## 二、关键契约

### ScrollableFormShell（新，`lib/app/widgets/`）

- 签名草拟：`ScrollableFormShell({header, content, primaryAction, secondary,
  contentMaxWidth = kFormContentMaxWidth, ...})`；结构为 `ReplicaScaffold` 内
  `SingleChildScrollView` → `Center` → `ConstrainedBox(maxWidth: contentMaxWidth,
  minHeight: viewport - padding)` → `Column(mainAxisAlignment: spaceBetween,
  crossAxisAlignment: stretch)`。
- 钉底语义：内容矮于视口时 CTA 靠 `spaceBetween` + `minHeight` 贴底；高于视口时
  CTA 随流位于文末——两种情形 CTA 均可达。welcome_page.dart L19-30 的
  `minHeight` 写法是现成实现参考。
- `contentMaxWidth` 是按角色的内容层常量（form/引导 = 520），不是断点；
  `user_agreement` 不经过 shell（`ListView` 与 `SingleChildScrollView` 嵌套
  冲突），而是保留 `ListView` 外层 `Align(topCenter)` + `ConstrainedBox`
  复用文章角色常量——两者共享的是「居中 + 限宽」语义而非同一 widget 树。
- 消费者清点：welcome / language / theme / login = 本包 4 个真实消费者且同
  stage 接入，满足父 §5.5 新增跨 feature shell 的门槛；命名不与 layering_test
  的 `_*Tail/_*Error/_*Empty/_*Card/_*Status/_*Placeholder` 族冲突。

### 登录动作分层

- 主行（register 描边 + login 实心）**恒定在场**；`_help` 只控制次级区显隐，
  永不替换主行。次级区 = `networkCompatibilityHint` 纯文本 + 剪贴板导入次级按钮
  + `accountTransferWarning` caption。
- FormState 归类（父 §5.3）：网络开关 = immediate（`setMode` 立即生效，失败
  SnackBar 已存在）；剪贴板导入 = action（idle/busy/success/error，busy 期间
  `onPressed` 置空 + label 区小尺寸进度指示——`ReplicaButton` 无 loading 变体，
  禁用 + 指示是本包内的等价表达）；代理提示 = 主操作前的 confirm dialog
  （cancel/continue 完整，不构成独立状态机）。
- `getMoreHelp` span 删除（无目标，component-guidelines：可见但不可点 = 缺陷）。

### WebView 错误层级（s4，动作语义以 W1 合入为准）

- 共享件放 `lib/features/login/`（如 `login_webview_error_card.dart`）：
  `LoginWebViewErrorCard{message, fatal, primaryAction, secondaryAction?}`——
  2 个同 feature 消费者，feature 私有共享，不触发跨 feature 组件规则。
- 布局统一：左下 `SafeArea` + `Padding(16)` + `Card(errorContainer)` 同构；
  动作区形态两端一致。
- **动作后果归 W1**：W9 只统一结构、层级与文案形态。s4 启动时对照 W1 实际
  合入的契约接线（W1 预期产出：recoverable → 「重新加载」`reload()`；
  fatal → 「重新登录」`beginSession()` + `loadRequest`；`create=true` fatal
  只给「重新加载」+ 关闭；l10n key 预期 `loginReload`/`loginRestart`——
  以 W1 实际命名为准）。
- 文案形态统一为 `'$type $host'`：带 host 定位错误来源，不带完整 query
  （避免长 URL 撑爆卡片）。
- mobile signup 分支补 `onProgress` 进度条，与 desktop 对齐（对 signup 无副作用）；
  desktop `_webView2Missing` 全页态保留。

### Spotlight 文章

- 行长：命名常量（~680–720dp，建议 700）放 `lib/app/layout/` 内容宽常量文件，
  注释引用 `research/external-m3-adaptive-layout.md` 的换算（bodyMedium 14sp
  ≈ 45-55 CJK 字/行 ≈ 640-760dp）；**不进 `AppBreakpoints`**（内容宽 ≠ 断点）。
  `user_agreement` 复用同一常量。
- 可选择范围 = 段落级：`_SpotlightBlockView` 按块产出 `SelectableText` /
  `SelectableText.rich`；`_linkSpans` 保持纯 `TextSpan` 树（`SelectableText.rich`
  的 `children` 只许 `TextSpan`，现状已符合）；`SpotlightIllustCard` 内文字
  不可选。禁用 `SelectionArea`：独拉 SelectableRegion/context-menu 机制，
  armeabi-v7a AOT 超 ~180KB cap（PR #50 / `324f3d6`，info_block.dart L102-105）；
  跨段选择、图片穿插区选择不可得，验收范围即段落级。
- `SelectableText.rich` 内 span recognizer 与长按选择/contextMenuBuilder 的
  手势竞争（flutter/flutter#168864）widget test 只能钉 tap 路径；
  「点链接 vs 长按选择」共存须真机/桌面验证，验证不到就标"未验证"。
- AppBar actions：`IconButton(share)` + `IconButton(open_in_new)`（或 overflow
  menu）。分享：`SharePayload(title: resolvedTitle, author: 'pixivision',
  url: _url)` 裸构造（public，不加 factory——pixivision 文章没有真作者槽，
  core 改动不值）+ `shareOriginOf(context)` 给 iPad popover 锚点；
  `copiedToClipboard` → `showAppSnackBar(linkCopied)`。打开原文：
  `launchUrl(Uri.parse(_url), mode: externalApplication)`。

### AppBreakpoints 事实与内容宽常量

`lib/app/layout/app_breakpoints.dart` 只有两档有效断点：`compact`/`medium` = 600、
`expanded` = 1200（与 M3 的 600/840/1200/1600 不同，§5.5 禁止以 M3 数值替换）。
本包不新增断点、不自造近似宽度常量；若需页内宽度分支只能读这两档。
内容限宽是按角色的常量（form/引导 520、article ~680–720），放
`lib/app/layout/` 下独立的内容宽常量文件（如 `content_max_width.dart`），
注释写明「内容宽 ≠ 断点」；本包消费者 = shell 默认值 + user_agreement +
spotlight_article，真实存在、非预建。

### 术语与 l10n（消费父 §5.7）

- 新增：`setupLater`（暂不设置/稍后设置）、`share`、`openInBrowser`——四语言
  `app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` → `python3 tool/gen_l10n_lookup.py`。
- 复用：`next`、`start`、`linkCopied`；s4 的动作文案 key 以 W1 合入的实际命名为准
  （预期 `loginReload` / `loginRestart`）。
- 删除：`later`（仅 language/theme 两处消费）、`getMoreHelp`（仅 login 一处），
  随消费点移除在四语言 arb 中清理并让 lookup 重生成。
- 'later' 改按钮（D3）：`TextButton(setupLater)` 与各自 `next` 同效推进——
  language → `push('/welcome/theme')`，theme → `completeGuide()` + `openLogin`；
  语义为「跳过本步、稍后可改」的真实 affordance，不是给说明文本加假 affordance。

## 三、复用与禁止

- 复用：`ReplicaScaffold` / `ReplicaButton` / `ReplicaSwitchTile`、
  `FeedLoading` / `FeedError` / `FeedEmpty` / `FeedTail`、`SettingsLoadError`、
  `showAppDialog`、`showAppSnackBar`、`shareServiceProvider` / `SharePayload` /
  `shareOriginOf`、`AppBreakpoints`、`decideLoginNavigation`、现有 l10n 工具链。
- 禁止：新断点常量塞进 `app_breakpoints.dart`、`SelectionArea`、平行
  SnackBar/overlay 入口、`TwoPane`（本包均单栏限宽）、parser/OAuth/intent/
  网络策略语义改动、外部浏览器登录（PKCE verifier 内存态，需会话存续改动）、
  为后续工作包预建的未引用基件。
- `StartupGate` 不变量：重定向进行中必须保持 `child`（Router）挂载
  （startup_gate.dart L59-61/L93-95）；shell 化不得触碰 allowlist 与裁决逻辑。

## 四、测试策略

- shell 矩阵 widget test：`tester.view.physicalSize` 覆盖 320×568 / 390 / 600 /
  840 / 1200 + 640×320 横屏矮高度 + `textScaler` 1.3（含 ru/en 长翻译）；
  断言无 overflow exception、主 CTA 可见可达——language/theme/login 当前
  不可滚动是存量缺陷，测试必须先钉住再迁移。
- 登录：`_help` 展开时 register/login 仍存在且可点（`widget.onRegister` /
  `widget.onLogin` 注入点现成）；剪贴板 busy 禁用 + 进度指示；
  `login_navigation_test.dart` 的 settings 失败态、policy 开关用例不回归。
- WebView：mobile 沿用 `login_navigation_test.dart` 的 `_FakeWebViewPlatform`
  stub；desktop 按 quality-guidelines 的 `InAppWebViewPlatform` fake 模式
  （`createPlatformInAppWebViewWidget` → `SizedBox.expand()`）补 smoke test；
  WebView2 缺失态仅 Windows 可验，标"未验证"。
- Spotlight：沿用 `spotlight_article_test.dart` 的 fixture/provider 模式
  （autoDispose `.future` 先挂 listen；CJK mock body 用 `Response.bytes` +
  utf8——两个坑见 quality-guidelines）；断言 `SelectableText` 存在、链接
  recognizer 仍触发 `openIllust`/`launchUrl`、840/1200dp 正文宽度被 cap、
  分享失败 → 剪贴板 + `linkCopied` SnackBar。
- 回归测试须先对缺陷证伪（quality-guidelines）：新断言在旧实现上应失败
  （如 `_help` 展开主行消失的 HEAD 行为）。

## 五、运行时证据缺口（PR 标"未验证"）

`SelectableText.rich` 点链接 vs 长按选择共存（#168864，需真机/桌面）、
Windows WebView2 缺失态与桌面错误卡实机表现、TalkBack/Narrator 朗读路径、
真机 1.3x 字号与横屏目测、iPad share popover 锚点、桌面端鼠标选择正文手感。
