# 执行计划：引导登录与内容布局（W9）

需求见 `prd.md`，技术设计见 `design.md`，逐项精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实）。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增/删除 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
`python3 tool/gen_l10n_lookup.py`。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-onboarding-auth-content-layout
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：看似不相关的
> 测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec 判定噪声再排查。

本机不可验项（#168864 `SelectableText.rich` 手势竞争、Windows WebView2 缺失态、
TalkBack/Narrator 朗读、真机 1.3x 字号与横屏目测、iPad popover 锚点）在 PR body
标"未验证"，不得由 widget test 推断通过。

## 分支拆分与硬前置

四个 stage 对应分支 `task/09-22-onboarding-auth-content-layout-s{1..4}`。

- s1–s3 不依赖 W1，可在 W1 合入前推进；若评审粒度允许也可合并为一条分支
  顺序提交。
- **s4（WebView）排最后且必须等 W1（`09-22-interaction-outcome-correctness`）
  合入 `main` 并 rebase 后启动**——两端 `login_webview*_page.dart` 与 W1 同文件
  （W1 修动作语义，本包做错误层级/布局），父 §3 同文件串行规则禁止并行分支。
  W1 当前仍为 `planning`；若 s1–s3 完成时 W1 未合入，s4 挂起等待，
  不得提前触碰 `login_webview*_page.dart`。

## 阶段 1：引导 shell + 四页迁移 + `_StartupError`（R1, R6）

分支：`task/09-22-onboarding-auth-content-layout-s1`

- [ ] **R1a**：新增 `lib/app/widgets/scrollable_form_shell.dart`：`ReplicaScaffold`
      内 `SingleChildScrollView` → `Center` → `ConstrainedBox(maxWidth:
      contentMaxWidth, minHeight: viewport - padding)` → `Column(spaceBetween,
      stretch)`；槽位 header / content / primaryAction / secondary（可选）；
      `contentMaxWidth` 默认 520 并注释「内容宽是按角色的常量，不是断点」。
      `lib/app/layout/` 新增内容宽常量文件（`kFormContentMaxWidth = 520`、
      `kArticleContentMaxWidth ≈ 700`）。
      测试：新 widget test 用 `tester.view.physicalSize` 跑 320×568 / 390 / 600 /
      840 / 1200 + 640×320 横屏 + `textScaler` 1.3，断言无 overflow exception、
      primaryAction 始终可见。
      提交：`feat(app): 新增可滚动限宽钉底 CTA 的 ScrollableFormShell`
- [ ] **R1b**：welcome/language/theme 迁入 shell：删除 `w*.1` 比例 padding 与
      `Column`+`Spacer` 固定布局；language/theme 标题改允许换行的居中标题
      （去掉 `FittedBox`）；welcome 双行 `FittedBox` 品牌 lockup 保留。
      'later' 槽改 `TextButton(setupLater)`，onPressed 与各自 `next` 同效；
      新 key `setupLater` 四语言，删除 `later` key。
      测试：三页各跑 shell 矩阵断言 + 'later' 按钮存在且推进到与 `next`
      相同路由的用例；`startup_gate_test.dart` 不回归（gate 不变量不动）。
      提交：`feat(onboarding): 引导页迁移到共用可滚动限宽 shell`
- [ ] **R1c**：`user_agreement_page.dart` 保留 `ListView`，外层
      `Align(topCenter)` + `ConstrainedBox(maxWidth: kArticleContentMaxWidth)`；
      各 section 正文 `Text` → `SelectableText`。
      测试：宽屏正文宽度被 cap、`SelectableText` 存在。
      提交：`feat(onboarding): 用户协议页限宽且正文可选择`
- [ ] **R6**：`startup_gate.dart` `_StartupError`（L270-307）retry 升
      `FilledButton` + 内容限宽居中。
      测试：`startup_gate_test.dart` 现有断言不回归。
      提交：`feat(app): 启动错误页主操作升级并限宽`

## 阶段 2：登录页分层 + 代理 dialog 复核（R2, R5）

分支：`task/09-22-onboarding-auth-content-layout-s2`

- [ ] **R2a**：`login_page.dart` 正文迁入 shell：删除 `height*.4` 硬高度
      （L279-286）与 `w*.1` padding（L271-273）；`loginAgree` + `userAgreement`
      链接（L288-303）入 secondary/正文槽；`isFirst` 标题槽沿用 shell header。
      测试：320×568 / 640×320 横屏矩阵断言主按钮可见、无 overflow。
      提交：`feat(login): 登录页迁入可滚动限宽 shell`
- [ ] **R2b**：`_buildLoginActions`（L391-447）重构为恒定分层：主行
      register（描边）+ login（实心）永远在场；`_help` 展开时
      `networkCompatibilityHint` + 剪贴板次级按钮 + `accountTransferWarning`
      追加在主行下方，不替换主行；删除 `getMoreHelp` span（L332-339）与
      l10n key；`_clipboardBusy` 期间按钮禁用 + label 区小尺寸进度指示。
      复核代理提示 `showAppDialog`（L135-151）：保持全断点 dialog，
      信息顺序/动作语义无回归，不做 sheet 分支（D7，无代码改动）。
      测试：`_help` 展开时 register/login 仍存在且可点；busy 态可辨；
      `login_navigation_test.dart`（policy 开关、settings error）不回归。
      提交：`fix(login): 登录主动作恒定存在，帮助与剪贴板导入次级化`

## 阶段 3：Spotlight 文章（R4）

分支：`task/09-22-onboarding-auth-content-layout-s3`

- [ ] **R4a**：`spotlight_article_page.dart` body `ListView` 外层
      `Align(topCenter)` + `ConstrainedBox(maxWidth: kArticleContentMaxWidth)`；
      文档头 title/description、各 `SpotlightHeading`、`SpotlightParagraph`
      `Text`/`Text.rich` → `SelectableText`/`SelectableText.rich`，`_linkSpans`
      recognizer 结构不变；`_SpotlightIllustCardView` 文字保持不可选；
      不用 `SelectionArea`。
      测试：`spotlight_article_test.dart` 断言 `SelectableText` 存在、链接 tap
      仍触发 `openIllust`/`launchUrl`、840/1200dp 正文宽度被 cap；
      #168864 长按共存标"未验证"。
      提交：`feat(spotlight): 文章正文限宽并可按段落选择`
- [ ] **R4b**：AppBar 加 `share` + `open_in_new` IconButton（或 overflow menu）：
      `SharePayload(title: resolvedTitle, author: 'pixivision', url: _url)` +
      `shareOriginOf(context)`；`copiedToClipboard` → `showAppSnackBar(linkCopied)`；
      打开原文 `launchUrl(Uri.parse(_url), mode: externalApplication)`；
      新 key `share`/`openInBrowser` 四语言。
      测试：tap share → share service 被调用且 `copiedToClipboard` 时弹
      `linkCopied` SnackBar；tap open → `launchUrl` 路径断言；
      `spotlight_article_test.dart` 现有断言不回归。
      提交：`feat(spotlight): 文章页支持分享与浏览器打开原文`

## 阶段 4：WebView 错误层级两端统一（R3）—— 排最后，W1 合入后启动

分支：`task/09-22-onboarding-auth-content-layout-s4`

**启动门禁**：W1 PR 已合入 `main` 且本分支 rebase 完成。
**复核点**（对照 W1 实际合入的契约逐项核对，不以预期为准）：

1. recoverable 错误卡的动作集与文案 key（预期「重新加载」→ `reload()`）；
2. fatal 错误卡的动作集与文案 key（预期「重新登录」→ `beginSession()` +
   `loadRequest`；`create=true` fatal 只给「重新加载」+ 关闭）；
3. l10n key 实际命名（预期 `loginReload`/`loginRestart`）与四语言文案；
4. `login_navigation_test.dart` 更新后的断言现状（W1 会改动错误卡动作断言）。

若 W1 合入契约与预期不符，以 W1 为准接线；本包只统一结构/层级/文案形态，
不改动作后果。

- [ ] **R3a**：抽 `lib/features/login/` 共享错误卡（如
      `login_webview_error_card.dart`，`{message, fatal, primaryAction,
      secondaryAction?}`），两端替换逐行复制块（mobile L285-318 /
      desktop L277-307）；统一左下 `SafeArea` + `Padding(16)` +
      `Card(errorContainer)` 布局与动作区形态；动作回调按 W1 契约接线。
      测试：mobile `_FakeWebViewPlatform` 断言 recoverable/fatal 两态动作集；
      desktop 按 quality-guidelines `InAppWebViewPlatform` fake 补 smoke test。
      提交：`refactor(login): 两端授权 WebView 共用错误层级卡`
- [ ] **R3b**：错误文案参数形态两端拉齐为 `'$type $host'`（带 host 不带完整
      query）；mobile signup 分支补 `onProgress` 进度条（对齐 desktop）。
      测试：signup 模式进度条断言；两端文案形态一致断言。
      提交：`fix(login): WebView 错误文案与注册进度条两端对齐`
- [ ] WebView2 缺失态、桌面端实机错误卡行为在 PR 标"未验证"。

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec 判定）；
      `git diff --check` 干净；`task.py validate` 通过。
- [ ] PR：`gh pr create --fill`（每 stage 独立 PR）；CI 绿后 `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随最后一个 PR 的提交）。

## 边界（不做）

- W3/W5/W6/W8 各自拥有的页面、表单与弹层；core 语义（parser/OAuth/intent/
  网络策略）；外部浏览器登录（PKCE verifier 内存态，需会话存续改动）；
  `SelectionArea`；Spotlight feed 列表改版；新断点；新 overlay/SnackBar 平行入口；
  `TwoPane`；WebView 动作后果本身（归 W1）；为未来工作包预建的未引用基件。
