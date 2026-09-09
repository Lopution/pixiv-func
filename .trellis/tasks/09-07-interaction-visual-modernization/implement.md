# 执行计划：交互与视觉现代化（child F）

## 执行前状态

- 当前 task：`.trellis/tasks/09-07-interaction-visual-modernization/`；计划分支
  `task/09-07-interaction-visual-modernization`，基线 `origin/main` `973ca71`。
- gate 已满足：B、C 已归档，A–D 均在当前基线。获得实现批准后运行
  `python3 ./.trellis/scripts/task.py start 09-07-interaction-visual-modernization --allow-empty-context`；本 task 由当前
  agent inline 执行，阶段上下文通过 `trellis-before-dev` 读取。
- 当前基线：84/4 个 Material/Cupertino import 文件、23 处 Navigator push、27 个 `ReplicaPageRoute` 引用、
  7 个 `PageStorageKey`、1 份 golden；D 记录全量 692 passed、analyze 0 issues。
- 每个 checkbox 一个提交。F1（依赖/UI 包）与 F3b（router）是显式回滚点。
- 工具链：`export PATH=/opt/flutter-3.47.2/bin:$PATH`；依赖变更后正常 pub get，其余测试使用 `--no-pub`。

## F0：基线与依赖合同

- [x] 对开工 HEAD 重跑 import/push/route/PageStorage/golden 计数；运行 `flutter pub outdated`，确认 Flutter 3.47.2
  可解析的 `material_ui`、`cupertino_ui`、`go_router >=18`、`cached_network_image >=4` 组合；对
  `migrate_design_widgets` 做 dry-run，并从解析后的依赖 source 列出仍 import SDK Material 的插件及触发页面。
  用 B 的 fdroid split 命令尝试生成迁移前 APK并运行 `tool/apk_size_report.py`。GitHub 原生资产下载阻塞已在
  `research/modernization-recount.md` 记录，未把失败构建写成体积结果。结果写
  `research/modernization-recount.md`，同步本 task 中漂移的基线。
  提交 `docs(ui): recount modernization baseline and dependency contracts`。

## F1：UI 包整体迁移

- [x] 更新 `pubspec.yaml`/`pubspec.lock` 到 F0 验证的四个版本；应用 `migrate_design_widgets`，迁移 app 自有
  `lib/`、`test/` imports；重跑 `flutter gen-l10n`。`PixivFuncApp` 使用显式 app/material/cupertino/widgets
  delegates，并在 app builder 放单一 `MaterialUiCompatibilityBridge`，覆盖 F0 清单中的 legacy plugin subtree。
  `flutter analyze --no-pub` 与全量测试通过；双 ABI fdroid release 的尝试仍受 F0 记录的 GitHub 原生资产下载
  阻塞，体积阈值继续沿用已验证基线，待网络恢复后在 F8 重建并校准。
  提交 `ui: migrate to material_ui/cupertino_ui`。

## F2：Material 3 主题

- [x] 按 `design.md` §3 更新 `replica_theme.dart`：移除 M2 开关，集中 AppBar/NavigationBar/TabBar/Card/Chip/
  Dialog/BottomSheet/SnackBar theme；`ReplicaSwitchTile` 改 M3 `Switch`。新增 theme/widget 回归测试；本阶段无受影响
  golden，light/dark 主题与现有本地化页面测试通过。`flutter analyze --no-pub`、代表性测试及全量测试（694 passed）
  通过。提交 `ui: adopt Material 3 theme`。

## F3：shell 与 go_router

- [x] 先把 `HomePage` 底部 `BottomAppBar + TabBar` 替换为 `NavigationBar`，仍沿用当前 TabController、lazy
  visited set 和 `homeShellMetricsProvider`；补齐四语言 destination label，更新 home/icon test 与受影响 golden。
  `flutter analyze --no-pub`、相关测试及全量测试（695 passed）通过。提交 `ui: replace home tabs with NavigationBar`。

- [x] 在 `app/navigation/routes.dart` 建 root key、五个 branch key、`StatefulShellRoute.indexedStack` 和完整 route
  table；`preload: false`。`PixivFuncApp` 改稳定的 `MaterialApp.router`，`StartupGate` 改 app-level gate；
  `HomePage` 的 selected index/tab switch 改由 `StatefulNavigationShell` 唯一管理。接入每 navigator observer scope，
  保持 HistoryVisibility、root double-back、lazy cold start 与 bar metrics。新增 `test/navigation_router_test.dart`。
  `flutter analyze --no-pub`、相关测试及全量测试（696 passed）通过。
  提交 `nav: go_router StatefulShellRoute with per-tab stacks`。

- [x] 把 `routes.dart` 全部 `open*` 门面改为 go/push/replace；settings、onboarding、login WebView、search
  replacement、novel previous/next、comments 和 image viewer 的直接 push 改 typed path。viewer route 使用
  `illustId/page/quality`，comment replies 使用 `illustId/rootCommentId`；实体只作可选 extra。删除已无引用的
  `ReplicaPageRoute`，并让 `rg 'ReplicaPageRoute\(' lib`、app page 的直接 push 为 0。
  提交 `nav: converge page navigation on typed routes`。`flutter analyze --no-pub`、相关测试及全量测试（696 passed）通过。

- [x] 把 Android initial intent/onNewIntent subscription 从 `HomePage` 移到 app-level bridge；继续使用现有
  `IntentRouter` 解析结果：illust/user 进入 recommended branch，ACTION_SEND push root reverse-image route，account
  callback 交给 login flow。覆盖冷启动与运行中 pixiv/pixivfunc/web link、分享图片及 rejection UI。
  `flutter analyze --no-pub`、intent/navigation/login 相关测试、全量测试（702 passed）、layering test 与
  `git diff --check` 通过。提交 `nav: route external intents through go_router`。

## F4：route 与滚动恢复

- [x] 设置 app/router/shell/branch/page 的稳定 restoration scope；ranking mode、search query/filter、viewer page
  写入 path/query 并在交互变化时 replace 当前 location。给 recommended/ranking/new/search/profile/history 等 feed
  的 ScrollView 增加稳定 `PageStorageKey` 与 `restorationId`，search input 增加 text-field restoration id。
  新增 `test/navigation_restoration_test.dart`，用 `restartAndRestore()` 覆盖当前 tab、branch stack、搜索词、viewer
  页码和代表性 feed offset。`flutter analyze --no-pub`、F4 restoration/导航相关测试、layering test 与
  `git diff --check` 通过；全量测试为 `705 passed, 1 failed`，唯一失败是已知
  `tls_sni_behaviour_test.dart` 真实 socket 超时，未见 F4 相关失败。
  提交 `nav: restore shell routes and scroll positions`。

## F5：Predictive Back

- [x] main manifest 开启 `android:enableOnBackInvokedCallback="true"`；审计两处 `PopScope` 和 go_router back
  分派：先 pop 当前 branch，branch 根才进入 `RootBackCoordinator`，ProfileEdit 未保存确认仍生效。扩展 router、
  root-back、profile-edit tests。`flutter analyze --no-pub`、navigation router、root-back、profile-edit tests、
  `git diff --check` 与 Dart format 检查通过。
  提交 `nav: enable predictive back`。

## F6：Hero 拖拽关闭

- [x] 新增 `app/motion/drag_to_dismiss.dart`；still viewer 在 1x 时接入下拉关闭，取消时回位，完成时 pop 并走
  现有 Hero reverse/`HeroRectClip`；多页横滑和 zoom pan 保持。Ugoira detail surface 接同一 wrapper。扩展
  `hero_transition_test.dart`、viewer 与 `ugoira_viewer_test.dart`。`flutter analyze --no-pub` 与上述三个聚焦测试
  均通过。
  提交 `ui: drag-to-dismiss viewer`。

## F7：其余 M3 控件

- [x] 搜索首页改 `SearchBar`，输入/建议面改 `SearchAnchor`，suggestion 仍只读取
  `searchAutocompleteProvider`，submit/filter 仍构造现有 typed `SearchQuery`；更新 search tests 与 golden（若覆盖）。
  `flutter analyze --no-pub`、`search_catalog_test.dart` 与 `navigation_restoration_test.dart` 均通过。
  提交 `ui: use Material 3 search surfaces`。

- [x] `BrowseSettingsPage` 的 preview/detail/view 三组 quality 列表改为各自 typed `SegmentedButton<T>`，调用现有
  settings notifier；`FollowSwitchButton` 保持现有实现。更新 settings tests。`flutter analyze --no-pub` 与
  `settings_test.dart` 均通过。
  提交 `ui: use segmented quality settings`。

## F8：体积与文档收尾

- [ ] 对最终 HEAD 重建 fdroid 两个 split APK，运行 B 的 size report，把 workflow 默认阈值重置为各 ABI 最终
  实测 + 1,000,000 B，并把精确数字写入 `backend/release-artifacts.md` 与 parent
  `research/apk-size-breakdown.md`。若 arm64 超过 32,000,000 B，先处理依赖/资源增量，硬上限不提高。
  提交 `ci: recalibrate per-ABI APK budgets after UI modernization`。

- [ ] 更新 README：结束 replica 视觉冻结，说明 Func 组件层、M3 与现代导航；更新
  `frontend/component-guidelines.md` 的 theme、NavigationBar、route restoration、Predictive Back、Hero drag 契约，
  修正仍指向 `backend/release-pipeline.md` 的 active 文档链接。
  提交 `docs: retire replica visual freeze`。

## 阶段验证

每个 checkbox 格式化触及文件并运行 analyzer + 相关测试；F1、F3、F4、F7 后跑全量：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
cd /root/Pixiv-func-F
flutter analyze --no-pub
flutter test -j 4 --no-pub
flutter test --no-pub \
  test/home_page_test.dart \
  test/startup_gate_test.dart \
  test/login_navigation_test.dart \
  test/intent_router_test.dart \
  test/hero_transition_test.dart \
  test/ugoira_viewer_test.dart
git diff --check
```

依赖/体积阶段：

```bash
flutter pub get
flutter build apk --release --flavor fdroid \
  --split-per-abi --target-platform android-arm64,android-arm \
  --obfuscate --split-debug-info=build/symbols/fdroid
PIXIV_APK_MAX_BYTES_ARM64_V8A=<threshold> \
PIXIV_APK_MAX_BYTES_ARMEABI_V7A=<threshold> \
python3 tool/apk_size_report.py \
  build/app/outputs/flutter-apk/app-arm64-v8a-fdroid-release.apk \
  build/app/outputs/flutter-apk/app-armeabi-v7a-fdroid-release.apk
```

结构结果：

```bash
rg -n 'package:flutter/(material|cupertino)\.dart' lib
rg -n 'ReplicaPageRoute\(' lib
rg -n 'Navigator\.of\([^)]*\)\.push|Navigator\.push' lib/features lib/app
```

前三条预期均为 0；dialog/bottom-sheet 的 `Navigator.pop` 不在该结果范围。

## 真机验收

- Android 14+：冷/热 illust、user、Pixiv web deep link；五分支各保留一层详情栈；预测性返回 preview/cancel/完成；
  still 与 Ugoira 拖拽关闭；登录 WebView、反查分享。
- API 29：同一路由返回顺序、两次返回退出、登录、下载、Ugoira、widget、updater 无回退。
- 开发者选项“不保留活动”以及 `adb shell am kill io.github.lopution.pixivfunc` 后重新进入：同 tab、同 branch
  page、同滚动区间；search query 与 viewer page 保持。

## 完成门槛与交付

- UI imports 迁移完成，M3 默认主题与 compatibility bridge 覆盖清单一致；golden 全绿。
- `MaterialApp.router` + 五分支 `StatefulShellRoute` 承载 app pages；分支 lazy、独立栈、HistoryVisibility 和 root back
  测试通过；直接 `ReplicaPageRoute`/page push 为 0。
- 深链、恢复、Predictive Back、still/Ugoira drag 和三类 M3 控件通过相关测试及真机矩阵。
- 最终两 ABI 体积低于更新后的默认阈值与 arm64 硬上限；`release-artifacts.md` 数字一致。
- `flutter analyze`、全量 `flutter test`、`trellis-check`、`git diff --check` 通过后，按 workflow 完成 journal、
  archive、rebase、PR、CI 与 merge；F 归档后再启动 E。
