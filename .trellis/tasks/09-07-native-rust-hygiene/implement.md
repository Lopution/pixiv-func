# 执行计划：原生层与 Rust 插件整理（09-02 child D）

设计依据：parent `design.md` §5（5.1 MethodChannel 约定、5.2 rhttp fork 策略、5.3 FRB 三元组）；
事实来源：`research/native-audit-recount.md`（2026-09-08 对 HEAD `8c91c37` 重数）。
本 child 不改 channel 名、方法名、payload 字段，不改 updater 验签算法与 manifest 字节语义，不改 Rust 源码逻辑。

## 开工条件

- [x] `09-01-settings-productization`（`android/` 改动）与 `09-01-release-blockers`（updater Kotlin）已归档并合入 `main`
      （PR #3、PR #6）。
- [x] D0 已对 HEAD 重跑 A.1–A.3：10 个 channel、15 处 `call.argument!!`、3 个笼统 `*_error`、12 处死 `SDK_INT`、
      4 处主线程 IO、updater 三函数逐字重复；`login_webview_intercept` 两侧已为 0；FRB 三元组已全 2.12.0。
- [x] `implement.md` 已按 D0 事实修订（D5b、D6 范围收窄，见各项）。

## 阶段 D1：契约文档先写现状

- [x] **D1** 新建 `.trellis/spec/backend/android-channels.md`：10 个 channel 的方法 / 参数（名、类型、必填）/ 返回 /
      当前错误码 / 线程（现状全部 main）/ Dart 调用点；`widget_snapshot/active.json` 文件契约
      （`schemaVersion=1`、`accountKey`、`accountRevision`、`generatedAtMs`、`items[]`、64 KiB / 1 MiB / 24 h / 8 项 / 512 字上限、
      `.write.lock`）；`pixivfunc/widget_background` 方向反转说明；updater `{valid:false, errorCode}` / `{status:failed, errorCode}`
      Map 形态标注为例外；fdroid `disabled` 仅 fdroid 可达。`backend/index.md` 加行。先写**现状**，D2/D3 改完后更新为目标状态。
      提交 `docs(spec): record the Android channel contracts as shipped`。

## 阶段 D2：参数与错误

- [x] **D2a** `MediaStoreChannel.kt`（8 处）与 `SafTreeChannel.kt`（7 处）的 `call.argument<T>()!!` 改为显式
      `result.error("<channel>_invalid_argument", "<name> missing", null)` 提前返回；`SafTreeChannel.kt:134` `data.data!!` 改为空判 +
      `saf_launch_failed`。`SafTreeChannel.create` 若继续忽略 Dart 传的 `ownerId`，在文档标明。
- [x] **D2b** 三个笼统码细分：`mediastore_error` → `mediastore_<reason>`（如 `mediastore_insert_failed` / `mediastore_write_failed` /
      `mediastore_finalize_failed` / `mediastore_not_found` / `mediastore_permission`）；`saf_error` → `saf_<reason>`；
      `webprofile_error` → `webprofile_<reason>`。全部码符合 `<channel>_<reason>` 小写下划线。Dart 侧若按码分类
      （`rg -n "mediastore_error|saf_error|webprofile_error" lib test`）同步改映射并保留旧码兼容分类或一次性替换（协议字段不变）。
- [x] **D2c** Kotlin JVM 单测：把 handler 逻辑抽成可无 Activity 调用的纯函数/对象（例如 `MediaStoreChannel.handle(call, result, deps)`
      或参数解析器 `ChannelArgs`），用 fake `MethodCall` / `MethodChannel.Result` 覆盖：缺参 → `*_invalid_argument`；非法值；平台异常 → 细分码。
      至少覆盖 mediastore / saf_tree / webprofile 三个 channel 的错误路径。`./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest` 通过。
      提交 `android: uniform channel argument and error handling`。

## 阶段 D3：线程

- [ ] **D3** 四处同步 IO 移出主线程：`MediaStoreChannel.kt:134` write、`SafTreeChannel.kt:114` write、
      `ReverseImageInputChannel.kt:147-158` 复制、github `DistributionUpdaterChannel.kt:128/207-213` `getPackageArchiveInfo`。
      方式：`BinaryMessenger.makeBackgroundTaskQueue()` 注册对应 channel（`MethodChannel(messenger, name, StandardMethodCodec.INSTANCE, taskQueue)`），
      或 IO 协程/executor 后 `Handler(Looper.getMainLooper()).post { result.success(...) }`。同一 channel 内的 begin/write/finalize 顺序必须保持
      （TaskQueue 默认串行；若用协程需按 `id` 串行）。`WebProfileChannel.kt:39` `CookieManager.flush()` 一并移出（顺带、低风险）。
      真机回归留给用户：下载到 MediaStore、SAF 目录写入、反查复制、自更新 APK 解析。
      提交 `android: move channel IO off the main thread`。

## 阶段 D4：死分支与残留

- [ ] **D4** 删除 12 处死 `SDK_INT` 分支：`MediaStoreChannel.kt:71,154,169`（`< Q`）；`AccountTransferClipboardChannel.kt:134`、
      github updater `:103,200,209,217`、fdroid updater `:39,56,64`（`>= P`）；github `:151`（`>= O` 外层，内层
      `canRequestPackageInstalls()` 保留）。保留 3 处 `TIRAMISU`（`AndroidIntentChannel.kt:88`、clipboard `:55,79`）。
      删 `res/drawable-v21/launch_background.xml`；删 debug/profile manifest 冗余 `INTERNET`。`http://pixiv.net` filter 不动。
      `rg "SDK_INT" android/app/src` 只剩 3 处 `TIRAMISU`。
      提交 `android: remove pre-API29 branches and template residue`。

## 阶段 D5：updater 去重与分块核对

- [ ] **D5** 新建 `src/main/kotlin/.../updater/UpdaterPlatformInfo.kt`，承载 `platformInfo` / `packageInfo` / `signerSha256`
      （三者两 flavor 逐字相同）；`packageInfoFromArchive` 仅 github 使用，随 github 文件或进 main 均可（记录选择）。
      两份 `DistributionUpdaterChannel.kt` 只保留差异逻辑；`updater_flavor_contract_test.dart` 若断言文件内容需同步
      （`SHA256withECDSA` 与五个错误码仍须在 github 文件可见）。Kotlin 测试通过。
      提交 `android(updater): share platform info helpers`。
- [ ] **D5b** 分块核对结论（不改协议）：Dart `media_store_channel.dart:123-126` / `saf_tree.dart:80-83` 无分块循环，块长即 rhttp
      `bytes_stream` 帧（reqwest/hyper 决定，无 256 KiB 常量）。**决定**：在 `DownloadSink` → `write` 路径加最小聚合（≥ 256 KiB 或流结束再
      `invokeMethod`），减少每帧一次 channel 往返；协议字段不变；单测断言聚合边界（小于阈值累积、跨阈值切分、结束冲刷、取消不写）。
      若实现风险大于收益（例如内存峰值），改为在本条记录「不做」及理由。启动路径复核已满足（`configureFlutterEngine` 仅注册 channel，
      不调 WorkManager）。提交 `download: coalesce channel writes to 256 KiB`（或记录不做）。

## 阶段 D6：rhttp fork 文档与 spec

- [ ] **D6a** `plugins/rhttp/UPSTREAM.md`：「Build configuration diffs」补 `android/build.gradle.kts` AGP 9 适配（`:38-41`）、
      `rust/tests/ech_config_test.rs` / `ech_live_handshake.rs`、`rust/examples/ech_reqwest_probe.rs`、`opt-level = "s"` 实验
      （已测 −1,539,000 / −884,944，未采用，见 `release-artifacts.md`）；Sync guide 扩为七步（diff 上游 → 重打 ECH → 重打构建配置差异 →
      `tool/frb_check.sh` → 必要时 codegen → 插件 `flutter test` + `cargo test` → 更新记录）。
- [ ] **D6b** 新建 `.trellis/spec/backend/rust-plugin.md`：fork 策略、FRB 三元组（全 2.12.0，`frb_check.sh`，`forceSameCodegenVersion: false` 保留）、
      再生成命令、cargokit ABI / 输出目录行为（`jniLibs/<buildType>` 清理）、插件 AGP 8.11.2 / Kotlin 2.2.20 / compileSdk 36 / minSdk 24 与
      app AGP 9.1.1 / Kotlin 2.4.0 / compileSdk 37 / minSdk 29 的差异与原因。`backend/index.md` 加行。
      `login_webview_intercept` 残留：两侧已 0，本条无事可做（记录）。
      提交 `docs(rhttp): complete fork record and seven-step sync; add rust-plugin spec`。
- [ ] **D6c** D1 的 `android-channels.md` 更新为目标状态（错误码表、线程列、参数校验）。提交 `docs(spec): channel contracts after D2-D5`。

## 最终验证

- [ ] `flutter analyze`
- [ ] `flutter test`（全量；含 `updater_flavor_contract_test`、下载相关）
- [ ] `(cd android && ./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest)`
- [ ] `(cd plugins/rhttp/rhttp && flutter test)`；`(cd plugins/rhttp/rhttp/rust && cargo test)`
- [ ] 双 flavor split release 构建（fdroid；github 带 `-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`）
- [ ] `rg '!!' android/app/src/main/kotlin` 与 `call.argument` 相关为 0；`rg "SDK_INT" android/app/src` 只剩 3 处 `TIRAMISU`；
      `rg -n "mediastore_error|saf_error|webprofile_error" android lib` 为 0
- [ ] `git diff --check`
- [ ] 用户真机（API 29）：下载到 MediaStore、SAF 目录写入、反查复制、自更新解析各一次

## 回滚点

D1 / D2 / D3 / D4 / D5 / D5b / D6 各自独立提交；channel 超时、线程阻塞或权限行为变化 → 回滚该提交。
D3 是唯一有运行时风险的一步：若 TaskQueue 串行导致 begin/write/finalize 顺序问题，回退到主线程注册并改用按 `id` 串行的协程。
