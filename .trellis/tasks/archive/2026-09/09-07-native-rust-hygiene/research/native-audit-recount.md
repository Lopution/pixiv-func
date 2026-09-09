# Research: native-audit-recount（D0，HEAD 重跑 A.1–A.3 / A.4–A.7 / B.1,B.6–B.7）

- **Query**: 对当时 HEAD 重跑 parent `native-and-build-audit.md` A.1–A.7、B.1、B.6–B.7，供 child D1–D6 开工
- **Scope**: internal（`/root/Pixiv-func-D` @ `8c91c37`，branch `task/09-07-native-rust-hygiene` = `origin/main`）；无网络、无构建
- **Date**: 2026-09-08
- **对照基线**: tag `baseline-2026-09-07` 上的 parent 研究（2026-09-07）；命令均在本 worktree 执行

---

## Findings

### A.1 文件清单（`wc -l`）

`find android/app/src/{main,github,fdroid,debug,profile} -type f` → 37 个文件。`LoginWebViewPlatformView.kt` 不在树中。`android/app/src/main/java/.../GeneratedPluginRegistrant.java` 不存在（`.gitignore` 忽略，本树未生成）。

`android/app/src/main/kotlin/io/github/lopution/pixivfunc/`（`find … -name '*.kt' | xargs wc -l` **2030**，与基线相同）：

| 文件 | LOC | 用途 |
|---|---|---|
| `MainActivity.kt` | 35 | `configureFlutterEngine` 注册 8 个 channel（10–17）；`onNewIntent`→`AndroidIntentChannel.dispatch`；`onActivityResult` 分派 reverse-image / SAF。无 `onCreate` |
| `MediaStoreChannel.kt` | 247 | `pixivfunc/mediastore` |
| `ReverseImageInputChannel.kt` | 233 | `pixivfunc/reverse_image_input` |
| `SafTreeChannel.kt` | 147 | `pixivfunc/saf_tree` |
| `AccountTransferClipboardChannel.kt` | 146 | `pixivfunc/account_transfer_clipboard` |
| `AndroidIntentChannel.kt` | 124 | `pixivfunc/android_intents` + `/events` |
| `WebProfileChannel.kt` | 49 | `pixivfunc/webprofile` |
| `WidgetForegroundChannel.kt` | 67 | `pixivfunc/widget` |
| `appwidget/WidgetUpdateCoordinator.kt` | 178 | WorkManager 周期/一次性 |
| `appwidget/WidgetHeadlessRunner.kt` | 115 | `pixivfunc/widget_background` **接收端** |
| `appwidget/WidgetRenderer.kt` | 284 | RemoteViews |
| `appwidget/WidgetSnapshotState.kt` | 201 | `files/widget_snapshot/active.json`（类名 `WidgetSnapshotReader`） |
| `appwidget/WidgetBudget.kt` | 89 | 位图预算 |
| `appwidget/WidgetClickReceiver.kt` | 37 | 刷新广播 |
| `appwidget/RecommendWidgetProvider.kt` | 45 | 推荐 widget |
| `appwidget/RefreshWidgetProvider.kt` | 33 | 刷新 widget |

Flavor / debug / profile：

| 文件 | LOC |
|---|---|
| `github/.../DistributionUpdaterChannel.kt` | **234**（基线 233） |
| `fdroid/.../DistributionUpdaterChannel.kt` | **75**（基线 74） |
| `github/AndroidManifest.xml` | 5 |
| `fdroid/AndroidManifest.xml` | 1 |
| `debug/AndroidManifest.xml` / `profile/AndroidManifest.xml` | 7 / 7 |

`android/app/src/test/kotlin/.../appwidget/{WidgetBudgetTest 85, WidgetSnapshotStateTest 37, WidgetWorkContractTest 26}.kt` = **148**。无 `src/test*` 其它目录、无 `androidTest`。

Gradle 源：`android/app/build.gradle.kts` **191**（基线 174）、`android/build.gradle.kts` 24、`settings.gradle.kts` 26、`gradle.properties` 6。Wrapper：`android/gradle/wrapper/gradle-wrapper.properties` → `gradle-9.3.1-all.zip`。

**vs 基线 A.1 文件集合**：未增未删。仅 updater 各 +1 行、`app/build.gradle.kts` +17。`main` 无 `updater/`。

---

### A.2 通道契约表（当前）

`rg -n "MethodChannel\(|EventChannel\(|setMethodCallHandler" android/app/src` 命中 10 个绑定（9 Method + 1 Event）。`rg -n "MethodChannel\(|EventChannel\(" lib` 与常量名对齐。无 `pixivfunc/notification*`、无独立 share/deeplink/platform-info channel（`ACTION_SEND`/深链走 intents；`getPlatformInfo` 属 updater）。

线程：全部 `setMethodCallHandler` 未传 `BinaryMessenger.TaskQueue`（默认 **main**）。无 `makeBackgroundTaskQueue` / 业务协程。

| Channel | Kotlin | Dart | 方法 / 参数 / 返回 / 错误 |
|---|---|---|---|
| `pixivfunc/mediastore` | `MediaStoreChannel.kt:24,32-33` | `lib/core/platform/media_store_channel.dart:11,30` | **begin** `displayName:String!!` `mimeType:String!!` `ownerId:String?` `relativePath:String?` → `Int`；**write** `id:Int!!` `bytes:ByteArray!!` → null；**finalize** `id!!` → `String` uri；**abort** `id!!` → null；**listPending** → `List<Map{id,displayName,ownerId}>`；**abortPending** `id!!` `ownerId:String!!` → `Boolean`。错误仅 `result.error("mediastore_error", …)` `:65`。`!!` NPE 落入同一码 |
| `pixivfunc/saf_tree` | `SafTreeChannel.kt:15,23-24` | `lib/core/platform/saf_tree.dart:40` | **pickTree** → `String?`；**create** `treeUri!!` `displayName!!` `mimeType!!` → doc uri（Dart `:61` 可传 `ownerId`，Kotlin **不读**）；**write** `uri!!` `bytes!!`；**close/delete** `uri!!`。细分：`saf_busy:29` `saf_unavailable:47` `saf_launch_failed:51` `saf_permission:143`；笼统 `saf_error:79` |
| `pixivfunc/reverse_image_input` | `ReverseImageInputChannel.kt:21,29-30` | `reverse_image_platform.dart:38,47`；`reverse_image_external.dart:10` | **pickImage**；**copyToTemp** `uri:String`；**deleteTemp** `path`；**openExternal** `url`（仅 https）。返回 Map/`Boolean`。显式 `invalid_uri:36` `invalid_path:46` `invalid_url:56` `cleanup_failed:50` `external_unavailable:60` `malformed_response:84` `picker_busy:95` `picker_failed:108`；`codeFor:212-216` → `permission_denied` / `input_unavailable` / `input_failed` |
| `pixivfunc/account_transfer_clipboard` | `AccountTransferClipboardChannel.kt:24,32-33` | `account_transfer_clipboard.dart:51,63` | **write** `text` `fingerprint` `clearAfterMs`；**read** → `{text,fingerprint}?`；**clearIfCurrent** `fingerprint`；**capabilities** → `{sensitiveMarkSupported:Boolean}`。`invalid_payload:74` `too_large:103` `invalid_fingerprint:116`。无 `!!` |
| `pixivfunc/android_intents` | `AndroidIntentChannel.kt:19,25-26` | `android_intent_channel.dart:10,30,60` | **getInitialIntent** → encode Map；**openUrl** `url` → `true`。`invalid_url:56` `no_handler:64` |
| `pixivfunc/android_intents/events` Event | `:20,33-42` | `:11,61` | `onNewIntent` → 同 encode：`action,uri,mimeType,hasReadUriPermission,sizeBytes,extraKeys`（`:77-84`）。`ACTION_SEND` 用 `EXTRA_STREAM`（`:70,87-93`） |
| `pixivfunc/webprofile` | `WebProfileChannel.kt:20,24-25` | `web_profile_session.dart:43,11` | **readSession** → cookie `String?`；**clearSession** → `true`。`webprofile_error:45` |
| `pixivfunc/widget` | `WidgetForegroundChannel.kt:22,25-26` | `widget_channel.dart:12,54` | **notifySnapshotChanged** 标量 `Number?`→`Long`（`:29`）；**clearSnapshot**；**requestRefresh**；**hasAnyWidget** → `Boolean`。无 `result.error` |
| `pixivfunc/widget_background` **方向反转** | `WidgetHeadlessRunner.kt:30,57-67` | `widget_background.dart:17-19,67` | Dart **调用** `result{outcome:String}`；原生 **接收**。无 flavor 差 |
| `pixivfunc/updater` | github `:25,37-38`；fdroid `:14,17-18` | `update_platform.dart:27` | 见下 |

**updater 方法**（github 实现 / fdroid 除前两项外 `result.error("disabled", …)` `:28-32`，**仅 fdroid 可达该码**）：

- **getCapability** → `{flavor, enabled, storeManaged}`（github `:40-46` / fdroid `:20-26`）
- **getPlatformInfo** → `{packageName, version, versionCode, signingCertificateSha256, supportedAbis}`（`:109-115` / `:45-51`）
- **verifyManifestSignature** `message:ByteArray` `signature:ByteArray` → `{valid:true}` 或 `{valid:false, errorCode}`：`public_key_missing` `message_missing` `signature_missing` `signature_mismatch` `algorithm_unavailable`（`:63-98`）
- **verifyApk** `path` `packageName` `signingCertificateSha256` → `{valid}` / `{valid:false, errorCode}`：`apk_path_invalid` `apk_identity_missing` `apk_missing` `apk_parse_failed` `apk_package_mismatch` `apk_signer_mismatch` `apk_verification_failed`（`:118-143`）
- **installApk** `path` → `{status:permission_required|started}` 或 `{status:failed, errorCode}`：`apk_path_invalid` `apk_missing` `installer_unavailable`（`:146-176`）
- **deleteApk** `path` → `{deleted:Boolean}`（`:179-183`）

**仅 github 可达**：`verifyManifestSignature` / `verifyApk` / `installApk` / `deleteApk` 的成功与 Map-`errorCode` 路径。`REQUEST_INSTALL_PACKAGES` 仅 `github/AndroidManifest.xml:4`。

**`widget_snapshot/active.json`**：Dart 写 `lib/core/widget/widget_snapshot.dart:119-125` + `widget_snapshot_store.dart:14,16-21`（`getApplicationSupportDirectory()/widget_snapshot/active.json`，Android = `filesDir/widget_snapshot`）。键：`schemaVersion`（必须 `1`）、`accountKey`（`[0-9a-fA-F]{1,128}`）、`accountRevision`、`generatedAtMs`、`items[]`：`illustId,title,userId,userName,imageFile`（文件名，无路径）。原生读 `WidgetSnapshotState.kt:53-54,75-155`：`MAX_SNAPSHOT_BYTES=64KiB:44`、`MAX_IMAGE_BYTES=1MiB:45`、`MAX_AGE_MS=24h:46`、`MAX_ITEMS=8:198`、`MAX_TEXT_LENGTH=512:199`。锁文件 `.write.lock`。

---

### A.3 约定

**`rg -n '!!' android/app/src/main/kotlin`**（flavor 零命中）：

- `call.argument<…>()!!` **15 处**：MediaStore `:37,38,44,45,49,51,58,59`（8）；SafTree `:55,56,57,61,62,67,72`（7）。
- 其它 `!!` **1 处**：`SafTreeChannel.kt:134` `data.data!!`（`pickTree` 结果，`:130` 已判 null）。

**三种错误风格**（仍并存）：

1. 笼统 `*_error` + `catch Exception`：`mediastore_error` `MediaStoreChannel.kt:64-66`；`saf_error` `SafTreeChannel.kt:78-80`；`webprofile_error` `WebProfileChannel.kt:44-46`。
2. 细分码 `result.error`：reverse-image / intents / clipboard / SAF 非笼统码（上表）。
3. updater Map `{valid:false, errorCode}` / `{status:failed, errorCode}`：github `:68-97,229-233`；Dart `update_platform.dart:171-172` 把 `PlatformException.code` 映成 `UpdatePlatformException`。fdroid 另用风格 2 的 `"disabled"`。

**`rg -n "SDK_INT" android/app/src`** 共 **15** 处（`minSdk=29` = `build.gradle.kts:31`）：

| 分类 | 处 | 位置 |
|---|---|---|
| 恒不触发 `SDK_INT < Q`（Q=29） | 3 | `MediaStoreChannel.kt:71,154,169` |
| 恒真 `>= P`（P=28） | 8 | clipboard `:134`；github `:103,200,209,217`；fdroid `:39,56,64` |
| 恒真 `>= O`（O=26）包一层 | 1 | github `:151`（内层 `canRequestPackageInstalls()` 仍有意义） |
| **KEEP** `TIRAMISU` | 3 | `AndroidIntentChannel.kt:88`；clipboard `:55,79` |

死分支合计 **12**（与基线/PRD 一致）。

**四处同步 IO（平台线程 = 默认 main）**：

| 站点 | 文件:行 | API |
|---|---|---|
| MediaStore write | `MediaStoreChannel.kt:134` | `OutputStream.write(ByteArray)` |
| SAF write | `SafTreeChannel.kt:114` | 同上 |
| 反查复制 | `ReverseImageInputChannel.kt:147-158` | `ContentResolver.openInputStream` + `File.outputStream`，缓冲 `64*1024` `:150` |
| updater APK 解析 | github `:128` 调 `:207-213` | `PackageManager.getPackageArchiveInfo` + `GET_SIGNING_CERTIFICATES` |

额外主线程 IO（不在 R3 四处内）：`CookieManager.flush()` `WebProfileChannel.kt:39`。

**残留**：`drawable-v21/launch_background.xml` 12 行与 `drawable/launch_background.xml` 11 行同为 `?android:colorBackground`（minSdk 29 ≥ 21，v21 永不被选）。debug/profile 各声明 `INTERNET`（main `:2` 已有）。`http://pixiv.net` filter 在 `AndroidManifest.xml:43-50`（保留，R4）。

**login 残留**（`rg -n -i "loginwebview|login_webview|login_intercept" android lib test`）：**零** `login_webview_intercept` / `LoginWebViewPlatformView` / `shouldInterceptRequest`。仅 `LoginWebViewPage`（`webview_flutter`，`login_webview_page.dart:3,20`）+ `login_page.dart` + `test/login_navigation_test.dart`。

---

### A.4–A.7 相对基线的变化

- **A.4 Manifest**：main 仍 124 行；权限仍仅 `INTERNET` + github `REQUEST_INSTALL_PACKAGES`；filters/FileProvider/receivers/`<queries>` 未变。
- **A.5 res**：仍 16 文件、`du -sb` **10929** B；`drawable-v21` 仍在。
- **A.6 Gradle**：AGP **9.1.0→9.1.1**（`settings.gradle.kts:22`）；Kotlin 仍 `2.4.0`；`compileSdk` 从 `flutter.compileSdkVersion` 改为 **37**（`app/build.gradle.kts:11`）；`androidx.core` **1.15.0→1.19.0** `:185`；`work-runtime-ktx` → `work-runtime:2.11.2` `:188`；新增 `packaging.jniLibs.excludes += "**/libsqlite3.so"` `:129-133`。`gradle.properties` 增加 `-XX:MaxMetaspaceSize=1G -XX:ReservedCodeCacheSize=256m -XX:+HeapDumpOnOutOfMemoryError`。插件 `plugins/rhttp/rhttp/android/build.gradle.kts` 仍 AGP classpath `8.11.2` / Kotlin `2.2.20` / `compileSdk=36` / `minSdk=24` / 注释 38–41 AGP 9 built-in Kotlin；`android/gradle.properties` `builtInKotlin=false` `useAndroidX=false`。
- **A.7**：本树无 `hs_err_pid*.log` / `.iml` / `GeneratedPluginRegistrant.java`。
- **CI（基线 C 的连带变化）**：`.github/workflows/ci.yml:201-216` 已有 `android-unit`：`./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest`。

---

### Updater 重复（R5）

`python3` 抽取 `private fun` 正文：

| 函数 | github | fdroid | 正文 |
|---|---|---|---|
| `platformInfo` | `101-116` | `37-52` | **identical** |
| `packageInfo` | `198-205` | `54-61` | **identical** |
| `signerSha256` | `216-227` | `63-74` | **identical** |
| `packageInfoFromArchive` | `207-214` | — | **仅 github**（`verifyApk` 用） |

`main` 无 `updater/UpdaterPlatformInfo.kt`、无 updater 源。github 独有：验签/验包/安装/删除 + `ownedApk` + `invalid`/`installFailed`。fdroid 独有：`disabled` 错误。

---

### 通道分块（R6b）与启动路径

- `media_store_channel.dart:123-126` 与 `saf_tree.dart:80-83`：**无分块循环、无字节常量**；`write(List<int> bytes)` 原样 `invokeMethod`。
- 调用方 `download_manager.dart:589-594` `await for (final chunk in openedResponse.stream)` → `sink.write(chunk)`（`download_sink.dart:154,252` 再透传）。
- 流来自 rhttp：`io_request.dart:45` `requestStream`；Rust `http.rs:319-330` `response.bytes_stream()` 逐 chunk `stream_sink.add`，**无 256 KiB 再切**。插件里 `1024*1024` 只在 `stream_listener.dart:3`（聚合上限，非写块）。
- **结论**：Dart 写块大小 **不是 ≥ 256 KiB 的约定常量**；块长 = reqwest/hyper 帧。`rg` `256 * 1024` 在 `lib/core/platform` / `android/app` 对 mediastore/SAF **零命中**。

**启动**：`MainActivity` 无 `onCreate`。`configureFlutterEngine:8-17` 仅 8 次 `*.configure`（channel 注册）。**不**调用 `WorkManager`。`WidgetForegroundChannel.configure` 只挂 handler；`ensurePeriodic` 在 `notifySnapshotChanged:31-34`（Dart 稍后）或 `RecommendWidgetProvider.onUpdate:20-23`（系统 widget 回调，非首帧 Activity 路径）。

---

### B.1 / B.6 / B.7 rhttp fork

`plugins/rhttp/UPSTREAM.md` 现有节：Upstream 元数据、Fork policy（ECH-only 功能声称）、**Build configuration diffs (parent D-5)** `:36-91`、**Sync guide** `:93-103`（**4 步**）。

Child B **已写入** UPSTREAM：cargokit `compileSdk` `tokenize('.')[0]`；FRB `pubspec.yaml` 精确 `2.12.0` + `tool/frb_check.sh`；根 `rust-toolchain.toml`；`lib.rs` rustfmt skip；cargokit 删 `jniLibs/<buildType>`；reqwest 丢 `multipart/form/socks/cookies/charset`、**保留 `query`**（B5e 回退，`:80-84`）；tokio 收窄。`Cargo.toml:35-46` 现 features：`http2, query, rustls, stream, brotli, deflate, gzip, zstd`；`opt-level = 3` `:52`。

Child B **已测未写入 UPSTREAM**：`opt-level = "s"`（B6：`.so` −1,539,000 / −884,944，**未采用**；`release-artifacts.md` 同述）。

**相对 child PRD R6 / design §5.2 仍缺**：

- `android/build.gradle.kts` AGP 9 适配（`:38-41` 已存在）**未列入** D-5 列表（只写了 cargokit parse）。
- `rust/tests/ech_config_test.rs`（21 行）、`ech_live_handshake.rs`（128）、`rust/examples/ech_reqwest_probe.rs`（89）**未列入**。
- 同步指引不是七步：缺独立「重打构建配置」、**`tool/frb_check.sh`**、插件 `flutter test` + `cargo test`。现 4 步把 ECH+构建配置并在第 2 步。
- `.trellis/spec/backend/android-channels.md`、`rust-plugin.md` **不存在**。

**FRB 三元组**（已对齐，相对基线 B.6 的 lock 2.13.0 漂移 **已消失**）：

| 位置 | 版本 |
|---|---|
| 根 `pubspec.lock` / 插件 `pubspec.lock` | **2.12.0** |
| 插件 `pubspec.yaml:21` | `flutter_rust_bridge: 2.12.0`（无 `^`） |
| `frb_generated.dart:74` `codegenVersion` | **2.12.0**（`rustContentHash = 2036241710`） |
| `rust/Cargo.toml:11` / `Cargo.lock` | `=2.12.0` |
| `tool/frb_check.sh` | **存在**（比 lock/yaml/codegen/Cargo.toml/Cargo.lock；codegen 二进制若在 PATH 也比） |
| `rhttp.dart:19` | 仍 `forceSameCodegenVersion: false` |
| `rust_builder` / 独立 `frb_codegen` crate | **无**；codegen 命令在 `flutter_rust_bridge.yaml` + UPSTREAM `:102` |

---

### Kotlin 测试

| 文件 | 覆盖 |
|---|---|
| `WidgetBudgetTest.kt` | `WidgetBudget` 采样/字节/总预算/slot |
| `WidgetSnapshotStateTest.kt` | 仅 `WidgetSnapshotReader.isStaleAt` |
| `WidgetWorkContractTest.kt` | unique work 名 + `shouldRetry` |

跑法：`./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest`（CI `ci.yml:215`）。`android/app/src/test` 对 `MethodCall` / `MethodChannel` **零命中**。handler 是 `configure(engine)` 内 lambda，无独立 `onMethodCall`。无 Activity 时 **不能** 把 `MethodCall` 派进现有 handler；无 harness。`MediaStoreChannel.configure` 只需 `Context`，但测试未抽逻辑。

---

## Deltas vs baseline

1. `LoginWebViewPlatformView` / `pixivfunc/login_webview_intercept`：**已从 Kotlin+Dart 消失**（基线仍记「未提交删除 + Dart 单侧」）。
2. Channel 集合仍 10 名；无 notifications；share/deeplink 不是独立 channel。
3. `!!` / 三种错误 / 12 处死 `SDK_INT` / 4 处主线程 IO / `drawable-v21` / 冗余 `INTERNET` / `http://` filter：**未变**。
4. updater 仍三函数逐字重复；LOC 234/75（+1/+1）；`supportedAbis` 已在两份 `platformInfo`。
5. Gradle：compileSdk 37、AGP 9.1.1、core 1.19.0、`work-runtime`、排除 `libsqlite3.so`。
6. FRB 三元组 **全 2.12.0**；`frb_check.sh` + `rust-toolchain.toml` 已在。
7. UPSTREAM 已有大块 D-5 构建差异 + `query` kept；缺 plugin `build.gradle.kts`、ech tests/examples、opt-level 实验、七步同步。
8. CI 已跑 Kotlin JVM 测试。
9. Dart mediastore/SAF **无** 256 KiB 写块常量（基线未测此项）。

---

## Implications for D1–D6

| 步 | 必须触及 | 计数 |
|---|---|---|
| **D1** 契约文档 | 新建 `.trellis/spec/backend/android-channels.md`（现状 10 channel + `active.json` + `widget_background` 方向 + updater Map 例外 + fdroid `disabled`） | 10 名；0 现有 spec |
| **D2** 参数/错误 | 改 15 处 `call.argument!!`；统一风格 1 的 3 个笼统码；**新写** channel JVM 单测（现 0 harness） | 15 `!!`；3 笼统码；1 其它 `!!` |
| **D3** 线程 | 4 个 IO 站点改 TaskQueue/IO；`result` 回主线程 | 4 站点 |
| **D4** 死分支/残留 | 删 12 处死 `SDK_INT`；留 3×`TIRAMISU`；删 `drawable-v21` + debug/profile `INTERNET`。**不必**清 `login_webview_intercept`（已 0） | 12+3；1 xml；2 manifest |
| **D5** 去重 | 抽 `platformInfo`/`packageInfo`/`signerSha256` → `main/.../updater/UpdaterPlatformInfo.kt`；flavor 只留差异。`packageInfoFromArchive` 仅 github | 3 函数 identical |
| **D5b** 分块 | **无现成常量可调**；若要坚持 ≥256 KiB 需在 Dart 下载→`write` 路径**新增**切块（协议字段不变）。启动路径已满足「仅注册 channel」 | 块长未钉死 |
| **D6** rhttp 文档 | 补 UPSTREAM：plugin AGP 9 `build.gradle.kts`、ech tests/examples、opt-level 未采用、四步→七步；新建 `rust-plugin.md`。FRB 对齐与 `frb_check.sh` **已完成** | 缺 4 类记录；三元组已齐 |

---

## Related Specs

- `.trellis/spec/backend/index.md` — 尚无 `android-channels` / `rust-plugin` 行
- `.trellis/spec/backend/release-artifacts.md` — per-ABI、updater schema 2、`query` 必须保留、opt-level `"s"` 未采用
- parent `design.md` §5（约 165–199）
- child `prd.md` R1–R7
- `plugins/rhttp/UPSTREAM.md`
- parent `research/native-and-build-audit.md`

## Caveats / Not Found

- 未跑 Gradle/Flutter/cargo；未测 reqwest `bytes_stream` 实际字节分布。
- `GeneratedPluginRegistrant.java` 本树不存在，插件列表未复核。
- github/fdroid updater +1 行未做 `git show baseline` 逐行 diff（基线表为 233/74）。
- 无独立 notifications / document-picker channel（SAF 即 document tree picker）。
