# Codebase: 登录页与授权 WebView（W9 调研）

基线：`main@8067b2d`。W1 仍 `planning` 未合入 —— 本文记录 HEAD 现状，W9 规划中的"W1 已修动作语义"届时需按 W1 实际产出再核对。

## login_page.dart（448 行）

### 状态与结构

- `_LoginPageState` L58-62：`_networkMode`（automatic/directOnly）、`_help`、`_clipboardBusy`。
- `initState` L64-76：从 `networkAccessPolicyProvider.mode` 映射初值；`widget.callback != null` 时 postFrame 走 `_handleExternalCallback`。
- `build` L229-242：`settingsProvider.when` — loading → `FeedLoading`；error → `SettingsLoadError`（C11，登录主控件在 settings 失败时整体不渲染）。
- `_buildSettings` L244-262：`ReplicaScaffold`；`isFirst` 时 AppBar 无 title（标题移到正文 L277）。

### 正文布局 `_buildBody` L264-308

- `Padding(horizontal: width*.1)` + **固定 `Column` + Spacer，不可滚动**。
- 中部 `SizedBox(height: height*.4)` 固定 40% 视口高度装 `_buildNetworkOptions`（L279-286）。
- 底部：`loginAgree` 文本 + `TextButton('userAgreement')` → `push('/user-agreement')`（L288-303）。
- 矮屏风险：横屏 ~360dp 高时 40% ≈ 144dp，装不下 switch tile + 动作行 → 溢出。

### `_buildNetworkOptions` L310-345 / `_buildNetworkTitle` L347-375

- `ReplicaSwitchTile`：`_toggleNetworkMode` L377-389 在 automatic↔directOnly 间切换，同步 `networkAccessPolicyProvider.setMode` 并 `_persistNetworkMode`（L168-180，失败 SnackBar）。
- 标题行尾部 ℹ️ `IconButton`（L362-372）切换 `_help`。
- `_help == true` 时显示 `RichText` 提示（L324-340）：`networkCompatibilityHint` + **`getMoreHelp`（"获取更多帮助 >>"）染成 primary 色但没有 recognizer —— 死链接样式**。

### `_buildLoginActions` L391-447 —— Astra #24 现场

- `_help` 分支（L395-421）：`accountTransferWarning` caption + `useLoginWithClipboardHint` 大字 + 剪贴板 `ReplicaButton` —— **注册/登录主按钮整组被替换消失**。
- 非 `_help` 分支（L422-446）：`Row` 内两个 `Expanded(ReplicaButton)` —— `register`（描边次级样式）+ `login`（实心主样式）。
- `_openLoginWebview(create:)` L134-166：先 `showAppDialog<bool>` 代理提示（L135-151，已是 `app_overlays` 入口；cancel→false / FilledButton"我已开启代理"→true），确认后 `push('/login/web?create=…')`；`result==true && returnToHomeOnSuccess` → `context.go('/recommended')`。
- `_importFromClipboard` L182-206：`_clipboardBusy` 防重入但**按钮无 busy 可视态**；成功 `accountTransferImported`，剪贴板未清时再发 `accountTransferClipboardReplaced`；`AccountTransferException` 映射 7 个错误码（L210-226）。

### 外部回调 `_handleExternalCallback` L78-132

`pixiv://account?code=…` → `oauthService.validateRedirect` → `exchangeCode` → `upsertAccount` → `context.go('/recommended')`；失败全部走 `showAppSnackBar`（loginFailed/loginFailedType/loginCallbackInvalid）。

## login_webview_page.dart（323 行，webview_flutter，移动）

- 状态 L42-51：`_controller`、`_exchanging`、`_progress`、`_error`、`_fatal`、`_mainFrameUri`。
- `initState` L54-95：`create=true` → signup URL + **不带 onProgress 的 NavigationDelegate**（L57-73）；否则 `beginSession()` 建唯一 PKCE 会话 + 完整 delegate（L78-95）。**注册模式没有进度条**（onProgress 只在登录分支注册 L87-89）。
- 导航判定 `_decideNavigation` L135-153 → 共享纯函数 `decideLoginNavigation`（login_navigation_decision.dart L41-56）：`LoginNavExchange`→`_exchange`+prevent；`LoginNavAbort`→`_abortLogin`+prevent；`LoginNavAllow`→记录 `_mainFrameUri`；`LoginNavIgnore`→放行。
- 错误分级：`_onHttpError` L167-184（非主帧请求直接忽略 L172-178）与 `_onWebResourceError` L186-193（仅主帧）→ `_reportRecoverable` L251-254（设 `_error`，fatal 后不再覆盖）；`_exchange` 失败/无效回调/detached → `_abortLogin` L238-246（`discardSession` + `_fatal=true`）。
- `dispose` L98-103 `discardSession`；`didChangeAppLifecycleState` L106-114 只对 `detached` abort。
- UI L257-322：AppBar + 右上 close（`pop(false)`）+ 2px 进度条；`Stack`：`WebViewWidget`、`_exchanging` 黑色半透明 scrim、`_error != null` 时左下 `Card(errorContainer)`。
- **错误卡动作（HEAD 现状，即 W1 要改的语义）**：`_fatal` → `TextButton('重新打开')` → `Navigator.pop(false)`（L301-306，实际只是关页不重开）；非 fatal → `TextButton('知道了')` → 清除 `_error`（L307-311）。**可恢复错误没有 reload 入口**。

## login_webview_desktop_page.dart（312 行，flutter_inappwebview/WebView2）

- 结构镜像移动端：同名字段（L47-52）+ `_webView2Missing`；同 `_exchange`/`_abortLogin`/`_reportRecoverable`（L123-168）；同 AppBar+close+进度条+左下错误卡（L171-309）。
- 差异点：
  - `_probeWebView2` L78-87：Windows 上探测 WebView2 runtime，缺失则全页安装提示（L191-218，`FilledButton.icon` → `launchUrl` 外部打开微软下载页）—— 移动端无对应态。
  - 导航拦截走 `onLoadStart` + `onUpdateVisitedHistory` 双通道（L238-250，注释解释 WebView2/CDP 限制）；signup 模式也跑 `_handleNavigation`（与移动端"signup 全权放行"不对称，但 signup 不会产生 pixiv:// 回调）。
  - `onProgressChanged` L251-253 对两种模式都生效（桌面注册有进度条，移动端没有）。
  - 错误文本带 URL：`'${error.type} ${request.url}'`（L260-262）、`'${statusCode} ${request.url}'`（L268-270）；移动端只带码（L181-183、L190-192）。
- 两端错误卡是**逐行复制**的相同代码（mobile L285-318 ≈ desktop L277-307）。

## 支撑件

- `login_navigation_decision.dart`（56 行）：纯函数共享契约 + 4 个 sealed 结果，两端复用。
- `oauth_service.dart`：`beginSession` L153-176（内存态 PKCE，`discardSession` 先清旧）；`hasLiveSession` L147-150；`validateRedirect` L184-205（无 live session / 已消费 / state 不匹配 → `PixivCallbackInvalid` + discard）。
- `intent_router.dart`：`pixiv://account?code=` → `AccountCallbackRoute`（L27-34、L265-282）；routes.dart L1082-1083 push `/login/callback`。**外部浏览器完成登录的回流通道存在**，但 verifier 只在内存、WebView dispose 即销毁 —— "在外部浏览器打开授权页"目前不是完整可用路径。
- `platform_caps.dart`：`isDesktop = isWindows||isLinux||isMacOS`（L36），routes.dart L884-886 选 WebView 实现。

## 本包自有 dialog 盘点（W9 全部弹层资产）

全目录 grep：引导/Spotlight 无 dialog/sheet；登录相关只有一处 —— login_page.dart L135-151 的代理提示 `showAppDialog` + `AlertDialog`（cancel TextButton / continue FilledButton）。已走共享入口，缺的是"同信息顺序的手机 sheet 形态"决策（§4.9：不强求相同位置，但要同语义）。

## 测试基线

- `test/login_navigation_test.dart`（513 行）：`_FakeWebViewPlatform` stub；断言 HTTP 400 → recoverable 文案 + 无"重新打开"（L380-396）；无效回调 → fatal 卡有"重新打开"无"知道了"（L398-413）；兼容开关写真实 policy（L415-442）；settings 失败态 `settings-load-error`/`settings-load-retry` + 无 ReplicaButton（L444-513）。
- `test/login_navigation_decision_test.dart`（63 行）：纯函数判定表。
- `test/startup_gate_test.dart`：见 onboarding 文档。
- **桌面端 WebView 页无 widget test**（quality spec L233-248 记录了 InAppWebView 需要 platform fake 的模式）。
