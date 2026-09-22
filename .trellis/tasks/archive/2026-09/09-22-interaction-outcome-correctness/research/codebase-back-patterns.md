# Codebase — 全库返回拦截模式清单 + 横切事实

## PopScope / 返回相关清单（全库）

| 文件 | 位置 | 模式 | W1 相关性 |
|---|---|---|---|
| `lib/features/novel/novel_page.dart` | L229–233 | `PopScope(canPop: !_chromeVisible)` + `maybePop` 显式按钮 | **修复对象**：显式按钮被自己的 canPop 拦死 |
| `lib/features/profile/profile_edit_page.dart` | L175–179 | `PopScope(canPop: session==null \|\| !dirty)` + `_attemptPop` | **修复对象**：`_attemptPop` 缺 dirty 短路 |
| `lib/features/home/home_page.dart` | L152–154 | 根 `PopScope<void>(canPop:false)` → `_handleRootBack`（双击退出/返回首页） | 参考实现，无需改 |
| `lib/app/widgets/replica_scaffold.dart` | L32–38 | leading 用 `Navigator.canPop` + `maybePop` | 使用者均无 canPop:false PopScope → 行为正确，不改 |
| `lib/features/profile/profile_header_delegate.dart` | L653 | `maybePop` | user page 无 PopScope → 正确 |
| `lib/features/search/reverse_image_search_page.dart` | L167–171 | leading = `cancel` tooltip + `_cancelAndPop` | **修复对象**（见 codebase-search-pages.md §8） |

`onPopInvokedWithResult` / `maybePop` / `canPop` 全库 grep 命中与上表一致，无遗漏的拦截点。

## 框架语义备忘（flutter 3.47.2 源码 + go_router 18.0.1 delegate）

1. `maybePop` 征询 `Route.popDisposition`：`canPop:false` → `doNotPop` → `onPopInvokedWithResult(didPop:false)`。系统返回手势也只走这条路。
2. `Navigator.pop`/`context.pop`（命令式）**不征询** `popDisposition`：go_router 页走 `onPopPage`→`route.didPop`；仍会触发 `onPopInvokedWithResult(didPop:true)`（flutter/flutter#163052）。拦截逻辑必须写在 `!didPop` 分支内才安全（本库现有写法都满足）。
3. `routes.dart` 无 `GoRoute.onExit`，命令式 pop 不会被二次确认拦截。
4. 结论模式：**显式「离开」按钮用 `pop()`；系统返回拦截用 `PopScope(canPop:false)` + `onPopInvokedWithResult` 里做局部动作（关 chrome/弹确认）**。这两条路在 flutter 层面是正交的，混用 `maybePop` 做「显式离开」会把拦截逻辑套到不该套的地方（novel_page 正是此 bug）。

## 测试基建横切

- 系统返回模拟：`tester.binding.handlePopRoute()`（`test/novel_reader_chrome_test.dart` L124–170 在用）。
- 路由注入：`appRootNavigatorKey`/`appRootRouteObserver` 可被测试覆盖（login_navigation_test 已用）。
- `ReaderHandle`/`NovelReaderHandle` 调试句柄存在于 novel_reader.dart（测试驱动翻页/chrome）。
- `ProviderScope.overrides` 模式：settingsRepositoryProvider、localNovelDatabaseProvider、translationCredentialStoreProvider、pixivNetworkFactoryProvider、reverseImage*Provider、oauthServiceProvider 全部可 override。
- `material_ui` 包：页面 import `package:material_ui/material_ui.dart`，测试必须用它的 `MaterialApp`/widget 类型（spec quality-guidelines）。
- WebView：`webview_flutter` 用 `_FakeNavigationDelegate`（login_navigation_test L27–72）；`flutter_inappwebview` 桌面页无 Linux 后端 → 桌面行为只能代码对称 + 手工验证。

## l10n / 工具链

- ARB 源：`lib/l10n/app_en.arb` + ja/ru/zh；生成 `app_localizations*.dart` 走 `flutter gen-l10n`；`tool/gen_l10n_lookup.py` 生成动态 key `lookup.dart`（改 ARB 后两者都要跑）。
- 现存可复用 key：`cancel`/`confirm`/`dismiss`/`retry`/`reopen`/`discard`/`searchApply`("Apply")/`searchReverseCancel`("Cancel")/`backButtonTooltip` 走 `MaterialLocalizations`。
- 需要新增的 key（候选）：`imageSourceApplyAndTest`（若选改名方案）、`loginRestart`/`loginReload` 或通用 `reload`/`restartLogin`、`clearImage`/`removeImage`（若重命名 ready 态取消）。**ARB 四语言同步是硬性工作量**，新 key 每个 ×4。
- `AGENTS.md`：一个 leaf 一个分支 `task/09-22-interaction-outcome-correctness`；本研究不动产品码。
