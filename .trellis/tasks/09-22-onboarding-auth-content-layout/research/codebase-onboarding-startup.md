# Codebase: 引导页与启动门（W9 调研）

基线：`main@8067b2d`（工作树在 `docs/09-22-ui-leaf-planning` 分支上，产品代码与 main 相同）。
W1（interaction-outcome-correctness）此时仍是 `planning` 状态、未实现；本文描述的是 W1 合入前的 HEAD 现状。

## 路由与入口

`lib/app/navigation/routes.dart`（行号已核对）：

- `/welcome` → `WelcomePage`，L833-836；子路由 `/welcome/language` L838-846、`/welcome/theme` L847-851。
- `/user-agreement` → `UserAgreementPage`，L854-861（根级路由，非 shell 分支）。
- `/login` → `LoginPage(isFirst: query 'first', returnToHomeOnSuccess: query 'return')`，L863-874；子路由 `web` L876-889（按 `platformCapsProvider.isDesktop` 选桌面/移动 WebView）、`callback` L890-903。
- `openLogin(context, isFirst, returnToHomeOnSuccess)` L1393-1407 —— push `/login?first=&return=`。

## StartupGate — `lib/app/startup_gate.dart`

- `StartupGate` L21-156：冷启动路由裁决。
  - `settingsPending` → 保持 child（L44）。
  - `!settings.guideCompleted` → 仅允许 `/welcome`、`/welcome/language`、`/welcome/theme`（L58-70，`_ensureLocation` allowlist）。
  - 无可用账号 → allowlist `/login`、`/login/web`、`/login/callback`、`/user-agreement`（L90-103）。
  - 已登录且仍停留在 auth/onboarding 路由或 `/splash` → postFrame `router.go('/recommended')`（L104-115）。
  - `_ensureLocation` L146-155：不在 allowlist 时 postFrame `go(roots.first)` 并返回 false。
  - 关键不变量：重定向进行中必须保持 `child`（即 Router）挂载，否则 in-flight `go()` 无人消费、死锁（L59-61、L93-95 注释）。W9 改这些页面时不能破坏该约束。
- `SplashPage` L244-259：中性启动面（96px 图标居中），路由 initialLocation。
- `_StartupProgress` L261-268：居中 CircularProgressIndicator。
- `_StartupError` L270-307：图标 + `accountReadFailed` + `$error` + TextButton retry → `accountStoreProvider.notifier.reload()`。普通 Scaffold，无限宽/无线宽，按钮是 TextButton（低重量）而非 FilledButton。

## 引导三页 — `lib/features/onboarding/`

### welcome_page.dart（99 行）

- `ReplicaScaffold` + `LayoutBuilder`（L15-22）：水平 padding = `maxWidth*.1` clamp 到 `[24,48]`；`minHeight = maxHeight-48`。
- `SingleChildScrollView`（L23）→ `Center` → `ConstrainedBox(maxWidth:520, minHeight)`（L26-30）→ `Column(spaceBetween, stretch)`。
- 标题两行 `FittedBox(scaleDown, maxLines:1, fontSize:24 bold)`（L42-56、L58-72）。注释 L39-41 明确：长译文缩字而非换行，保证各 locale 锚点一致。
- 主动作 `ReplicaButton('start')`（L76-88）→ `context.push('/welcome/language')`。
- **这是唯一已有"可滚动 + 限宽（520）+ 稳定底部主区"语义的引导页**，是 shell 的事实原型。

### language_page.dart（115 行）

- `settingsProvider.when`：loading → `FeedLoading`；error → `SettingsLoadError`（L29-40）。
- `_buildContent` L43-114：`ReplicaScaffold` + `Padding(horizontal: width*.1)` + **固定 `Column` + 4 个 `Spacer`（不可滚动）**。
- 标题 `FittedBox(scaleDown, maxLines:1)` L56-66。
- 4 个 `ReplicaSwitchTile` + `Divider`（L68-85）：zh-CN/en-US/ja-JP/ru-RU，`onTap` → `settingsProvider.notifier.selectLanguage`。
- `ReplicaButton('next')` L87-95 → `push('/welcome/theme')`。
- `'later'` 槽位 L99-108：`SizedBox(height:44)` 内 `Text("稍后您可以在设置中进行相应变更")` —— **纯展示文本，非按钮**（注释 L97-98 说明是固定高度锚点）。
- 与 welcome 不同：**不可滚动、无限宽**（`width*.1` 只随宽度比例缩放，1200dp 时单侧 120dp、内容 960dp；超宽屏继续变宽）。

### theme_page.dart（141 行）

- 结构与 language 几乎相同：`width*.1` padding（L46-47）、固定 Column+Spacer（L48-136）、FittedBox 标题（L51-61）、3 个 `ReplicaSwitchTile`（dark/light/system，L63-103）。
- `ReplicaButton('next')` L105-121：`await completeGuide()` → `openLogin(context, isFirst:true, returnToHomeOnSuccess:true)`。
- 同样的 `'later'` 展示槽位 L125-133。

### user_agreement_page.dart（81 行）

- `ReplicaScaffold(title)` + **全宽 `ListView`**（L17-18，padding 24/20/24/32），6 个 `_AgreementSection`（L56-80）。
- 正文 `Text`（bodyLarge/bodyMedium），**不可选择**；无行长限制。
- 仅两处入口：登录页 L290 `push('/user-agreement')`；StartupGate 把它当作 no-account 流程内合法路由（startup_gate L76、L91-100）。设置里没有入口。

## 已固定的测试基线

- `test/startup_gate_test.dart`（373 行）：gate 行为全部有 widget test —— guide 未完成 → WelcomePage（L124-135）；无账号 → LoginPage（L137-153）；账号 hydration pending 时登录页必须保持可见（L156-176）；已登录冷启动不闪 WelcomePage（L178-224）；reauth 失败回登录（L250-299）。
- 引导页本身**没有**专属 widget/golden 测试；`test/golden_matrix_test.dart` 只覆盖共享 widget（FeedEmpty/FeedError/TagChips/底部导航）。

## W9 相关现状问题（对应 Astra #23，状态"部分"）

- language/theme 页固定 `Column`+`Spacer`：320dp 宽、横屏矮高度或 1.3x 大字号下内容可能溢出（Spacer 到 0 后 Column 溢出），只能依赖 FittedBox 缩标题。
- `FittedBox(scaleDown)` 缩字策略与路线图 §6 门禁"不以无限缩字维持单行"精神冲突（该门禁写在 W2/W3，但验收矩阵对全部叶子适用）。
- 引导三页结构与限宽规则各不相同（welcome=520 固定；language/theme=10% 比例无上限），需要一个共用 shell 收敛。

## 复用资产

- `ReplicaScaffold`（lib/app/widgets/replica_scaffold.dart，44 行）：AppBar(canPop→maybePop 返回钮)+SafeArea body。
- `ReplicaButton`（48 行）：主 CTA，圆角 40、纵向 padding 20、headlineSmall 字号。
- `ReplicaSwitchTile`（35 行）：InkWell + Switch 的整行 tap。
- `SettingsLoadError`（53 行）：settings 读取失败共享面（key `settings-load-error`/`settings-load-retry`）。
- `FuncTokens.primary`（0xFFFF6289）+ `FuncSemanticTokens`（body/caption/title 等语义样式）。
