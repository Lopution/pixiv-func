# 执行计划：全项目重构（parent 级）

## 执行前状态

- 当前 task：`.trellis/tasks/09-02-performance-size-maintainability-refactor/`，状态 `planning`。
- 本文件不是 `task.py start` 的授权。`prd.md` 的 D-1～D-8 得到答复、D-8 通过后按任务地图创建 child，
  每个 child 各自经 review 后 `task.py start`；parent 自身没有直接实现工作。
- 工作树（2026-09-07）含 09-01 多个 child 的未提交改动（`lib/` 15 个未跟踪文件、30+ 已修改文件、
  `android/` 8 文件 +181/−460）。任何 child 开工时先 `git status --short` 存档，不触碰这些文件。
- 研究数字以 2026-09-07 工作树为准；每个 child 开工第一步是对当时 HEAD 重跑对应研究脚本/命令。

## 工具链前置

```bash
export PATH="/opt/flutter-3.47.2/bin:$PATH"
python3 ./.trellis/scripts/task.py current --source
python3 ./.trellis/scripts/task.py list --json
git status --short > /tmp/pre-09-02-status.txt
```

## Child A：`dependency-toolchain-health`

gate：用户批准。与 09-01 仅在 `pubspec.yaml` 行级可能冲突。

- [x] A0 基线：`flutter pub outdated`、`cargo update --dry-run`、`unzip -l` 三 ABI 汇总，写入
      `research/dependency-upgrade-audit.md` / `apk-size-breakdown.md` 的"实施时复核"小节。
- [x] A1 删除 `cupertino_icons`（`pubspec.yaml`；`go_router` 保留，D-12 后由 F 升级并接线）；`flutter pub get`；确认 APK 中
      `packages/cupertino_icons` 消失。提交 `deps: drop unused cupertino_icons`。
- [x] A2 Flutter 3.47.0 → 3.47.2：本地 SDK（新目录或 `flutter upgrade`，更新 `android/local.properties`
      由工具自动重写）；`ci.yml` 两处、`release.yml` 一处 `flutter-version`。提交 `build: flutter 3.47.2`。
- [x] A3 `flutter pub upgrade`（不改约束）；`cd plugins/rhttp/rhttp/rust && cargo update`；核对新 crate
      许可证；重编 `.so` 后跑插件 `flutter test` 与 `cargo test`。提交 `deps: refresh pubspec.lock and Cargo.lock`。
- [x] A4 Android 工件：仅 `work-runtime-ktx` → `work-runtime:2.11.2`（ktx 自 2.8 为空壳）。
      `androidx.core:core:1.19.0` 因要求 compileSdk 37，并入 A7。Kotlin 2.4.10 可选（D 阶段可再定）。
      提交 `build(android): use work-runtime instead of the empty ktx artifact`。
- [x] A5 CI actions：`actions/checkout@v6`、`actions/setup-java@v5`。提交 `ci: bump actions`。
- [x] A6 耦合升级 `archive: ^4.2.0` → `image` 解析到 4.9.x；跑 `test/ugoira_test.dart`；真机 Ugoira
      播放 + GIF 导出各一次。提交 `deps: archive 4 / image 4.9`。
- [x] A7 `flutter_secure_storage: ^11.0.0` + AGP `9.1.1` + `compileSdk 37` +
      `androidx.core:core:1.19.0` + 确认 `platforms;android-37`（本机 + CI 镜像）；
      `flutter build apk --debug` 走通；真机登录凭据与翻译凭据读写。提交 `deps: flutter_secure_storage 11`。
- [x] A8 FRB 三元组：`tool/frb_check.sh`；插件 `pubspec.yaml` 的 `flutter_rust_bridge` 收紧为与
      `Cargo.toml` 相同的精确版本；`flutter pub get` 后 lock 回到 2.12.0；脚本进 CI。提交
      `build(rhttp): pin flutter_rust_bridge triplet and add check`。
- [x] A9 工具链固化：仓库根 `rust-toolchain.toml`（1.98.0；cargokit 只读 app 根目录，rustup 向上查找同一文件）；CI
      `flutter pub get --enforce-lockfile`、`cargo test --locked`；`UPSTREAM.md` 记录。提交
      `build: pin rust toolchain and enforce lockfiles`。
- [x] A10 CI 覆盖：新增 `plugin`、`android-unit`、`deps-report` job（见 design §6）；`dart format
      --output=none --set-exit-if-changed` 暂不加入（E 负责一次性格式化后再加，避免 09-01 未提交文件冲突）。
      提交 `ci: run kotlin, plugin and rust tests`。
- [x] 退出条件：`flutter analyze`/`flutter test`/插件测试/`cargo test`/Kotlin 测试/双 flavor release
      构建全部通过；`pub outdated` 直接依赖只剩 `go_router`/`cached_network_image` 的 material_ui 系大版本（留给 F）。

回滚点：A2（SDK 版本）、A6、A7 各自独立；A3 的两个 lock 与其触发的 `.so` 重编同提交。

## Child B：`release-size-per-abi`

gate：D-1～D-5 已确认；updater 多资产 schema 在 `09-01-release-blockers` PRD 定稿（或其已归档且
B 承担 schema 1→2 迁移）；A 完成。

- [ ] B0 基线：对当前 HEAD 重跑 `unzip -l` 三 ABI 汇总 + `--analyze-size`（clean `build/rhttp/jniLibs`
      后构建，避免 §0.2 的陈旧 ABI 污染）。
- [ ] B1 `libsqlite3.so`（D-4：桌面分支与主依赖保留）：`android/app/build.gradle.kts` 增加
      `packaging { jniLibs { excludes += "**/libsqlite3.so" } }`；新增测试断言 Android 平台工厂为
      `sqflite.databaseFactory` 而非 FFI。验证：release APK `lib/<abi>/` 无 `libsqlite3.so`；真机历史记录
      读写正常；debug 包同样验证。若 Flutter 的 native asset 打包绕过了 jniLibs 合并导致排除无效，记录
      结论并放弃本项（收敛目标改 ≤ 30 MB）。提交 `size(android): exclude unused sqlite3 native asset`。
- [ ] B2 cargokit 清理：`cargokit/gradle/plugin.gradle` cargo 任务前 `delete(cargoOutputDir)`；
      `UPSTREAM.md` 记录。验证：`--target-platform android-arm64` 构建后 APK 只含 arm64 的 `librhttp.so`。
      提交 `build(rhttp): clean stale ABI outputs before cargo build`。
- [ ] B3 per-ABI 流水线：`ci.yml`/`release.yml` 改为 `--split-per-abi --target-platform
      android-arm64,android-arm --obfuscate --split-debug-info=build/symbols/<flavor>`；遍历两个 APK
      验签、拒绝 debug 签名、输出体积汇总；符号目录上传为 artifact；`release.yml` 上传两个 APK。
      `build.gradle.kts` 注释更新（versionCode 偏移保留）。提交 `release: ship per-ABI APKs`。
- [ ] B4 updater 多资产（与 release-blockers 协作，代码归属以其 PRD 为准）：`tool/update_release.py`
      多 `--apk <abi>=<path>`；`update_manifest.dart` schema 2 解析（按 D-3 决定是否双读 schema 1）；
      `update_service.dart:391–404` 按 `supportedAbis` 选资产、`versionCode % 1000` 比较；Kotlin
      `platformInfo` 增加 `supportedAbis`；`tool/RELEASE.md` 更新。验证：`updater_*_test`；API 29 与
      高版本真机各一次跨 ABI 自更新。提交 `updater: multi-asset manifest (schema 2)`。
- [ ] B5 rhttp feature 裁剪（每去一个 feature 一个提交，附 `.so` 前后大小）：`multipart` → `form` →
      `socks` → `cookies` → `query` → `charset`；`tokio` features 收窄。每步：cargokit 重编 →
      插件 `flutter test` → `cargo test` → `test/rhttp_client_factory_test.dart` +
      `restricted_compat_network_test.dart` → 真机登录/图片/下载。`UPSTREAM.md` 记录。
- [ ] B6 rhttp `opt-level = "s"` 实验：记录 `.so` 大小与一次图片列表/大图下载耗时对比；差异可接受则采用，
      否则回退并记录。
- [ ] B7 阈值：按 B1–B6 后的实测设定 CI 门禁（建议 arm64 APK 上限 = 实测 + 1 MB）。
      提交 `ci: enforce per-ABI APK size budget`。
- [ ] B8 更新 `research/apk-size-breakdown.md` §6 的实测值；记录图片 `CacheManager` 的 `Config`（对象数/stalePeriod），不改配置。
      阈值注明"F 迁 material_ui 后需重测更新"。
- [ ] 退出条件：release 产物为 2 个 split APK；arm64 ≤ 32 MB（B3 后）并记录 B1–B6 后的实际值；
      updater 跨 ABI 真机通过；双 flavor 构建通过。

停止条件：任一 feature 裁剪导致 TLS 握手/解压/ECH 回归立即回退该提交；`--obfuscate` 导致 widget
后台入口失效（`@pragma('vm:entry-point')`）则去掉 obfuscate 并记录。

## Child C：`dart-architecture-convergence`

gate：触及 `lib/` 的 09-01 child 全部归档；对当时 HEAD 重跑 `dart-architecture-audit.md` 与
`modernization-gaps.md` 的计数脚本。

- [ ] C0 规则先行：`test/architecture/layering_test.dart`（import 图：`core→features` = 0；
      features 之间只允许 `lib/app/navigation/routes.dart`；repository/controller/entity 只在 `core/`）。
      首次运行以"已知违反清单"白名单通过，后续每消除一项就从白名单删除。提交
      `test(arch): layering rules with current violations allow-listed`。
- [ ] C1 删除零引用：4 个文件（`compat_network.dart`、`android_platform.dart`、
      `tag_search_repository.dart` 视测试处理、`login_intercept_controller.dart` 若 09-01 已定为删除）、
      13 个声明、8 个未用 `AppIcons` 常量（放宽 `icon_font_test`）；`unreachable_from_main` 开启。
      提交 `refactor: remove unreferenced files and declarations`。
- [ ] C2 路由与导航目录合并：`ReplicaPageRoute` 唯一；`lib/app/navigation/` 收拢；`routes.dart` 门面
      （`openIllust(context, id)`/`openUser(context, id)`/`openNovel`/`openSearch(query)`…，以 id/参数为形参，F 用 go_router 重实现）；
      `HomeShellMetrics` 静态量改 provider。提交
      `refactor(nav): single route builder and navigation facade`。
- [ ] C3 组件层（design §13）：`lib/app/widgets/feed/`：`IllustCard`、`IllustFeedGrid`（8 处瀑布流统一，列数由
      `illustColumnsFor(crossAxisExtent)` 计算，手机恒 2 列）、`FeedTail/FeedEmpty/FeedError`（替换 32 个私有类，
      先写差异矩阵）；`lib/app/motion/`：`replica_page_route.dart`、`HeroRectClip`（原 `_GlobalRectClip`）、
      `motion_tokens.dart`；共享组件带 `Semantics`/tooltip、≥48dp 触达。提交三次：
      `refactor(ui): feed grid and IllustCard under app/widgets/feed`、`refactor(ui): shared feed state widgets`、
      `refactor(ui): motion tokens and hero clip under app/motion`。
- [ ] C3b `PixivImage` 变体与 decode 策略：`PixivImageSize.feed/detail/viewer/avatar` + 命名构造器（含 `.hero(tag:)`），
      feed/avatar 传 `memCacheWidth`（布局宽 × dpr，上限 1.5× 逻辑像素），detail 按屏宽，viewer 不限；13 个调用点
      切换；`PersonAvatar` 复用 avatar 变体；widget 测试断言 feed/avatar 变体的 `ImageProvider` 带 `memCacheWidth`。
      提交 `perf(image): size-aware decode policy for PixivImage variants`。
- [ ] C3c 重建边界（静态规则）：共享组件只 `watch` 所需切片（`select`），`IllustCard` 不 `watch` 整个 feed state；
      code review 检查，可临时用 DevTools 自查但不作 gate。
- [ ] C4 repository/controller 归位：5 个 repository、6 个 controller 迁入 `core/<domain>/`，
      controller 与 repository 分文件；`startup_gate.dart` 上移到 `lib/app/`。白名单清零。
      提交 `refactor(core): move repositories and controllers under core`。
- [ ] C5 单一 owner：主机/头（`PixivClientIdentity` 派生，`PixivHeaders.image()`）；
      `sharedPreferencesProvider` + `PreferenceKeys`；`json_read.dart`；`showAppSnackBar`；`log()`；
      HTTP 客户端 provider 化。每项一个提交。
- [ ] C6 i18n（按 D-6）：迁移脚本 → ARB/或拆表 → 调用点替换 → 删除 27 个包装与 `ReplicaStrings` →
      key 校验进 CI。提交 `i18n: migrate to gen-l10n` 或 `i18n: typed keys and single accessor`。
- [ ] C7 状态范式：3 个 `ChangeNotifier` → Riverpod；`history_page` → `HistoryFeedController`；
      页面不再 `new` 平台适配器。每项一个提交，测试改 provider override。
- [ ] C7b 持久化增长：`DownloadRecoveryStore` 设记录上限（淘汰已完成项）或改增量写入，code review 决定并单独提交；
      history 三条查询在单测中 `EXPLAIN QUERY PLAN` 断言不出现 `SCAN TABLE`。
- [ ] C8 文件拆分：`settings_page.dart`（含 `MePage` 去重；抽出 `SettingsSection`/`SettingsTile`/`SettingsControl`
      原语，子页只做组合）→ `illust_detail_page.dart`
      （`_GlobalRectClip` 公开化，`hero_transition_test` 改 `find.byType`）→ `user_page.dart` →
      `network_policy.dart`（需 network-perf-ab 已归档）；5 个 >150 行方法拆分。每文件一个提交。
- [ ] C9 颜色收敛到 `FuncTokens`/主题；142 个仅本文件使用的公共声明按需私有化（随 C8 顺带）。
- [ ] 退出条件：layering_test 白名单为空（含私有 widget 名称检查）；`flutter analyze`/`flutter test` 全绿且测试数
      不减少；i18n key 校验通过；四个大文件单文件 ≤ 600 行；`rg "ChangeNotifier" lib` 为 0；`rg "crossAxisCount: 2" lib/features`
      为 0。

停止条件：任何拆分改变了 Hero 转场、Tab 动画、下拉刷新契约（`component-guidelines.md`）或
`PagedFeedController` 三相语义 → 回滚该提交。

## Child D：`native-rust-hygiene`

gate：`09-01-settings-productization`（`android/` 改动）与 `09-01-release-blockers` 归档。

- [ ] D0 对 HEAD 重跑 `native-and-build-audit.md` A.1–A.3 的清单（channel 表、`SDK_INT`、`!!`）。
- [ ] D1 `backend/android-channels.md` 契约表先写（现状），再改代码。
- [ ] D2 参数与错误：`call.argument<T>()!!` → 显式 `invalid_argument`；三种错误风格统一为细分码；
      updater Map 形态标注例外。Kotlin 单测覆盖错误码。提交 `android: uniform channel argument and error handling`。
- [ ] D3 线程：MediaStore/SAF/反查复制/updater 解析改 `TaskQueue`/IO 协程。真机下载与 SAF 写入回归。
      提交 `android: move channel IO off the main thread`。
- [ ] D4 死分支与残留：12 处 `SDK_INT` 恒真分支；`drawable-v21`；debug/profile 冗余 `INTERNET`。
      提交 `android: remove pre-API29 branches and template residue`。
- [ ] D5 updater 去重：`UpdaterPlatformInfo.kt` 进 main source set。提交 `android(updater): share platform info helpers`。
- [ ] D5b 通道分块：核对 `pixivfunc/mediastore`/SAF 的 Dart 侧写块大小，低于 256 KiB 则调整（协议不变）。
- [ ] D6 `UPSTREAM.md` 补全（构建配置差异、cargokit 清理、toolchain、tests/examples、同步七步）；
      `backend/rust-plugin.md`；`login_webview_intercept` 单侧残留清理（若 09-01 已删原生端）。
- [ ] 退出条件：Kotlin 测试与 channel 单测通过；API 29 真机下载/SAF/反查/updater 各一次；
      `rg "SDK_INT" android/app/src` 只剩 `TIRAMISU`。

## Child F：`interaction-visual-modernization`

gate：C 归档（组件层、`motion_tokens`、以 id 为形参的门面）；B 归档（体积门禁存在）。开工第一步：记录当时
`lib/` 中 `package:flutter/material.dart` import 数、`Navigator.of(context).push(` 数、golden 清单、per-ABI 体积。

- [ ] F1 `material_ui`/`cupertino_ui` 迁移：`flutter add material_ui cupertino_ui`；`dart fix --apply --code=migrate_design_widgets`；
      `localizationsDelegates` 改用 material_ui 的 `GlobalMaterialLocalizations.delegates`；`MaterialApp.builder` 包
      `MaterialUiCompatibilityBridge`（覆盖仍 import SDK material 的 `easy_refresh` 等）；`cached_network_image: ^4.0.0`、
      `go_router: ^18.0.0`；`flutter analyze`/`flutter test`；重测 per-ABI 体积，增量 > 1 MB 时列出 legacy 插件替代评估。
      提交 `ui: migrate to material_ui/cupertino_ui`。
- [ ] F2 M3 主题：`useMaterial3` 默认；`ColorScheme.fromSeed(FuncTokens.brand)`；组件主题集中 `lib/app/theme/`；
      `CupertinoSwitch` → M3 `Switch`（或记录保留）；重生成 golden。提交 `ui: Material 3 theme`。
- [ ] F3 go_router 架构：`MaterialApp.router`；`StatefulShellRoute.indexedStack` 分支 = 首页 tab；页面为子路由；
      `CustomTransitionPage` 消费 `motion_tokens`；`intent_router.dart` 深链 → go_router 路径；`routes.dart` 门面改
      `context.go/push`；`replicaRouteObserver` → `observers`；删除 `ReplicaPageRoute` 的直接调用（28 处经门面已归零）。
      提交 `nav: go_router StatefulShellRoute with per-tab stacks`。
- [ ] F4 状态恢复：`restorationScopeId`；搜索词/viewer 页码进路由参数；全部 feed 加 `PageStorageKey`；
      验证：开发者选项"不保留活动" + `adb shell am kill io.github.lopution.pixivfunc` 后回到同 tab/同页/同滚动区间。
      提交 `nav: state restoration`。
- [ ] F5 Predictive Back：manifest `enableOnBackInvokedCallback="true"`；`PopScope` 审计；Android 14+ 真机验证；API 29 回归。
      提交 `nav: predictive back`。
- [ ] F6 Hero 手势返回：`motion/drag_to_dismiss.dart`；viewer 拖拽关闭驱动 Hero 反向转场。提交 `ui: drag-to-dismiss viewer`。
- [ ] F7 M3 组件：`NavigationBar`、`SearchBar`/`SearchAnchor`、`SegmentedButton`（三档枚举）。每项一个提交。
- [ ] F8 文档：README 原则改写（视觉冻结终止、Func 组件层）；`component-guidelines.md` 记录新交互契约；
      `backend/release-artifacts.md` 更新体积阈值。提交 `docs: retire replica visual freeze`。
- [ ] 退出条件：`rg "package:flutter/material.dart" lib` 为 0；`rg "ReplicaPageRoute\(" lib/features` 为 0；深链、恢复、
      预测性返回真机通过；golden 全绿；per-ABI 体积已更新到 B 的门禁；`flutter analyze`/`flutter test` 全绿。

停止条件：`material_ui` 与 legacy 插件的主题互不可见导致可见错乱且桥无法覆盖 → 停在 F1，评估替换插件或回退；
go_router 恢复与 `PagedFeedController` 生命周期冲突 → 停在 F4，先修契约再继续。

## Child E：`spec-test-lint-hardening`

gate：E0 随 A 开始；E1–E3 在 C、D、F 之后。

- [ ] E0 文档（可提前）：两份 `index.md` 状态列修正；`backend/release-artifacts.md`（随 B 更新）；
      `backend/rust-plugin.md`、`backend/android-channels.md` 骨架（随 D 填实）。
- [ ] E1 spec 填写：`directory-structure`×2（C 的规则）、`error-handling`、`database-guidelines`、
      `type-safety`、`logging-guidelines`；`lib/core/<domain>/` library 文档。
- [ ] E2 测试：`test/helpers/fake_account.dart`（凭据/元数据/账号 store + 标准 overrides）、
      `test/helpers/test_preferences.dart`；替换重复假实现；去除类名字符串断言；删除
      `zz_diag_tabbar_geometry_test.dart`；`.gitignore` 加 `test/failures/`。测试数不减少。
- [ ] E2b 无障碍与组件层校验：共享组件（`IllustCard`、`FeedTail/Empty/Error`、`showAppSnackBar`、settings 原语、
      收藏/关注按钮）的 semantics 测试（`SemanticsTester`/`find.bySemanticsLabel`）；layering_test 增加"`features/`
      不得定义 `_*Tail/_*Error/_*Empty/_*Card`"检查。
- [ ] E3 格式与 lint：`dart format lib test` 一次性提交；CI 加 format 检查；第一批 lint
      （`sort_pub_dependencies`、`prefer_single_quotes`、`unreachable_from_main`、`prefer_final_locals`）
      → 0 issues 提交；第二批（`strict-casts/raw-types/inference`、`unawaited_futures`、
      `avoid_dynamic_calls`）→ 0 issues 提交。
- [ ] 退出条件：`flutter analyze` 在两批规则下 0 问题；CI format 检查通过；spec 无模板占位。

## 推荐提交顺序与回滚点

1. A1–A10 各一提交（依赖/工具链）。
2. B1、B2 独立提交；B3+B4 联动（流水线与 manifest 同期，若 D-3 由 release-blockers 落地则 B4 只做
   消费端）；B5 每 feature 一提交；B6、B7 各一提交。
3. C0 先于 C1–C9；C 内每项一提交；删除旧路径（旧 repository 位置、`replicaRoute()`、`ReplicaStrings`）为显式回滚点。
3b. F1 → F2 → F3 → F4 → F5 → F6 → F7 → F8；F1 与 F3 各自是显式回滚点（依赖与导航形态）。
4. D1 文档先于 D2–D5 代码。
5. E3 的格式化提交单独且不含任何逻辑改动。

失败只回滚当前提交；不使用 `git reset --hard`/`git clean`。

## 验证命令

```bash
export PATH="/opt/flutter-3.47.2/bin:$PATH"
flutter pub get --enforce-lockfile
dart format --output=none --set-exit-if-changed lib test          # E3 之后
flutter analyze
flutter test
(cd plugins/rhttp/rhttp && flutter test)
(cd plugins/rhttp/rhttp/rust && cargo fmt --check && cargo test --locked)
tool/frb_check.sh                                                  # A8 之后
(cd android && ./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest)
flutter build apk --release --flavor fdroid --split-per-abi --target-platform android-arm64,android-arm \
  --obfuscate --split-debug-info=build/symbols/fdroid              # B3 之后；github flavor 另需签名参数
python3 - <<'EOF'
import zipfile,collections,sys,glob
for apk in glob.glob('build/app/outputs/flutter-apk/app-*-release.apk'):
    z=zipfile.ZipFile(apk); agg=collections.defaultdict(int)
    for i in z.infolist():
        k=i.filename.split('/')[1] if i.filename.startswith('lib/') else i.filename.split('/')[0]
        agg[k]+=i.compress_size
    print(apk, f"{sum(agg.values())/1e6:.1f} MB", {k:f"{v/1e6:.1f}" for k,v in sorted(agg.items(),key=lambda kv:-kv[1])[:6]})
EOF
git diff --check
# F：状态恢复与深链验证（真机）
adb shell am kill io.github.lopution.pixivfunc          # 配合开发者选项"不保留活动"
adb shell am start -a android.intent.action.VIEW -d "pixiv://illusts/123456"
```

## 高风险文件与停止条件

- `lib/core/network/compat/*`：路由阶梯/ECH/证书语义；只做等价重排与文件拆分，行为差异即停。
- `lib/core/download/*`、`lib/core/ugoira/*`：恢复/取消/流式语义；`_run` 拆分需先确认测试覆盖。
- `lib/core/paging/*`、`lib/app/pull_to_refresh.dart`：全部 feed 的共享基建；三相语义与滚动位置不得变。
- `android/app/src/**/kotlin`：线程改动需真机验证 MediaStore/SAF；channel 名与方法名不得变。
- `plugins/rhttp/rhttp/rust/Cargo.toml`、cargokit：每一步重编并跑双侧测试；工具链不一致不提交生成物。
- `tool/update_release.py`、`lib/core/updater/*`：manifest 字节即签名内容；schema 变更必须与
  release-blockers 同步。
- `pubspec.yaml`/`pubspec.lock`/`Cargo.lock`/`build.gradle.kts`/workflows：删除前消费者搜索 +
  许可证 + 双 flavor 构建。

## 完成门槛

- [ ] 六个 child（A–F）全部 check/archive；parent 验收清单（`prd.md` Acceptance Criteria）逐项勾选。
- [ ] 体积、依赖、分层、i18n、channel、导航/恢复六类结论都有可复现的命令与记录文件。
- [ ] 用户完成 parent 真机矩阵（含跨 ABI 自更新）后，本 task 归档。
