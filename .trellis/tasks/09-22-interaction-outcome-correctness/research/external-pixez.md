# External — PixEz 同类实现对照

仓库：https://github.com/Notsfsssf/pixez-flutter（同类 Pixiv 客户端，Flutter；分 material `lib/page/` 与 Fluent `lib/fluent/page/` 两套 UI）

## 登录页

- 源文件：`lib/page/login/login_page.dart`、`lib/fluent/page/login/login_page.dart`
- 观察：登录页常驻「登录 / 注册」两个并列按钮 + Token 登录入口；注册是独立动作而非藏在 WebView 里。
- 复用点：本库 `create=true` 模式把注册塞进同一 WebView 页——致命错误后的「重新登录」必须按 `create` 分流（注册模式重载 signup URL 而非 authorize URL），PixEz 把两者分开的思路印证了「动作集按会话类型区分」。

## WebView 页

- 源文件：`lib/page/webview/webview_page.dart`、`lib/fluent/page/webview/webview_page.dart`
- 观察：AppBar 常驻动作 = 「外部浏览器打开」+「刷新（`controller.reload()`）」；`pixiv://` 回调被拦截后直接 pop 出结果。
- 复用点：常驻 reload 图标是该类 app 的既定模式，支持 W1 在 login WebView AppBar 加常驻 reload IconButton（或至少错误卡内给 reload）；回调拦截→pop 的路径本库已由 `LoginNavExchange`/`validateRedirect` 覆盖且更严格（PKCE/state/重放校验）。

## 小说阅读器

- 源文件：`lib/page/novel/viewer/novel_viewer.dart`（另有 `novel_web_viewer.dart`）
- 观察：leading 返回按钮直接 `Navigator.of(context).pop()`（约 L203–209），无 chrome 拦截；阅读位置用滚动偏移做书签。
- 复用点：「显式返回=直接 pop」与本库修复方向一致；本库多出「系统返回先关 chrome」是增强，PixEz 无此交互，无可比负面证据。

## 反向搜图

- 源文件：`lib/page/saucenao/saucenao_page.dart`
- 观察：页面级 store、无「取消当前搜索」控件（仅重新选图），取消/离页不分。
- 复用点：仅作对照——本库 `ReverseImageController` 的状态机（idle…searching→ready/canceled）已显著强于 PixEz，W1 的 `stopSearch` 是在已有状态机上加一个合法转换，不是新架构。

## 断言边界

PixEz 代码只证明「这些交互在同类产品里成立」，不证明其正确性；其 WebView 无错误呈现、无取消语义，恰是本库已超越的部分。引用时只借「动作呈现模式」（常驻 reload、显式返回直接 pop）。
