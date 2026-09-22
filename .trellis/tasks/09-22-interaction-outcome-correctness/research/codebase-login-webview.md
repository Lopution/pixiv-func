# Codebase — 手机/桌面授权 WebView 动作一致性（§4.1 第 9 项）

## 共享契约

`lib/features/login/login_navigation_decision.dart`：`LoginNavAllow | LoginNavExchange | LoginNavAbort | LoginNavIgnore` 四态，两端共用。

`lib/core/auth/oauth_service.dart`：
- L153–176 `beginSession()`：**先 `discardSession()`** 再生成新 PKCE verifier/challenge/state → 重复调用安全，天然支持「重新开始登录」。
- L262–265 `discardSession()` 清内存密钥。
- L184–205 `validateRedirect` 拒绝畸形/过期/重放/state 不符回调；`exchangeCode` `finally` 清会话。

路由：`routes.dart` L876–889 `/login/web` 按 `platformCapsProvider.isDesktop` 选 `LoginWebViewDesktopPage`(InAppWebView/WebView2) 或 `LoginWebViewPage`(webview_flutter)；`create=true` 走注册 URL。

## 手机端 `lib/features/login/login_webview_page.dart`

- L78 `beginSession()` → `loadRequest(authorizeUrl)`；dispose 时 `discardSession()`。
- L263–266 AppBar 关闭 X：`Navigator.pop(false)` —— 离页+弃会话（dispose 兜底），语义正确但无显式 reload/重登控件。
- L285–318 错误卡：
  - `_fatal`（PKCE 会话已死的 `LoginNavAbort`/`_abortLogin`）→ 唯一按钮 `l10n.reopen`（"Reopen"/"重新打开"）→ `Navigator.pop(false)`。**名不符实**：按钮承诺重新打开，实际只是关闭页面回到登录页，用户需再点一次「网页登录」——且标签与「离开」后果不一致。
  - 可恢复错误（HTTP/连接失败）→ 仅 `dismiss`，**没有 reload 动作**。
- `_abortLogin`（L238–246）：`discardSession` + `_fatal=true`。

## 桌面端 `lib/features/login/login_webview_desktop_page.dart`

- L70 `beginSession()` 同款；dispose `discardSession()`。
- L177–180 关闭 X → `pop(false)`。
- L290–300 错误卡与手机端同构：`reopen`→pop(false)、`dismiss`。
- InAppWebView `WebUri`/`URLRequest` API 不同但能力对等（`reload()`/`loadUrl` 均有）。

## §4.1 要求的三个动作

设计措辞：「reload、新登录和返回动作名称与结果一致」。映射到两端统一的动作集：

| 动作 | 语义 | 手机实现 | 桌面实现 |
|---|---|---|---|
| 重新加载 | 重新加载当前 WebView 页面（同一会话） | `_controller.reload()` 或 `loadRequest(_mainFrameUri)` | `controller.reload()` 或 `loadUrl(_mainFrameUri)` |
| 重新登录 | 弃当前 PKCE 会话、`beginSession()`、加载新 authorizeUrl | 同左 | 同左 |
| 返回/关闭 | `pop(false)`，dispose 弃会话 | 已有（X） | 已有（X） |

建议落地形态（draft 细化）：
- 可恢复错误卡：`重新加载`（主）+ `关闭提示`（次）。
- 致命错误卡：`重新登录`（主，原地 `beginSession`+loadRequest，`_fatal/_error/_exchanging` 复位、`_mainFrameUri` 更新）——比「pop 出去让用户再点一次」更符合「名称=结果」；保留 AppBar X 作为离开通道。若评审倾向最小改动，也可把 `reopen` 改名为 `返回`/`back` 保持 pop 行为——但「重新登录」原地重启是更诚实的实现，且 `beginSession` 已幂等。
- `create==true`（注册）无 PKCE 会话：「重新加载」= 重载注册 URL；「重新登录」不适用 → 致命态只给 `重新加载`(signupUrl) + 关闭。
- 两端 AppBar 可考虑常驻 reload icon（IconButton+tooltip），PixEz 的 WebView 页就有常驻刷新（见 external-pixez.md）——是否加常驻 reload 属 UX 决策点，错误卡内的 reload 是必须的。

## 测试

`test/login_navigation_test.dart`：
- `_FakeNavigationDelegate`（L27–72）模拟 webview_flutter 导航；页面级测试已覆盖「打开页面」「HTTP 错误可恢复」「非法回调终止登录」（L190–440 区间）。
- 需新增：可恢复错误 → 点「重新加载」→ 断言 `loadRequest`/`reload` 被调且页面未 pop；致命错误 → 点「重新登录」→ 断言 `beginSession` 再次调用、新 authorizeUrl 加载、`_fatal` 清除。
- 桌面 `flutter_inappwebview` 无 Linux 后端 —— 桌面页 widget 测试需 platform fake（spec quality-guidelines 已提示）；桌面分支的实现验证主要靠代码对称 + 手机端测试 + 手工 Windows 验证（记入未验证项）。
