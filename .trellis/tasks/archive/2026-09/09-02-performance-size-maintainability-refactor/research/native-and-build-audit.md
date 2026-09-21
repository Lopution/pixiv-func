# Research: Android 原生层、Gradle/Flutter/CI 构建流水线与 vendored Rust 插件 `plugins/rhttp` 审计

- **Query**: 盘点 Android 原生（Kotlin/Manifest/res/Gradle）、构建与 CI 流水线、`plugins/rhttp` Rust 插件；聚焦"什么在推高 APK 体积"以及"什么让这些层难维护/难升级"。
- **Scope**: internal（仓库源码 + 本机已存在的构建产物 + 本机 Flutter SDK 源码 + cargo 元数据）；未做网络检索。
- **Date**: 2026-09-07
- **约束**: 本文只记录观测到的事实，每条都附文件路径/行号/字节数；未跑新的长构建。体积预期一律标注"已测量 / 待测量"，不编造数字。

---

## 0. 体积基线（引用主会话数据 + 本次 `unzip -l` 复核）

### 0.1 `build/app/outputs/flutter-apk/app-github-release.apk`（88,590,531 B，2026-09-07 16:11，3 ABI）

| 条目 | 字节 | 备注 |
|---|---|---|
| `lib/x86_64/` 合计 | 31,810,096 | libflutter 13,051,040 / libapp 10,224,520 / librhttp 6,702,128 / libsqlite3 1,709,544 / libdartjni 116,640 |
| `lib/arm64-v8a/` 合计 | 29,020,544 | libflutter 11,747,528 / libapp 9,962,376 / librhttp 5,439,920 / libsqlite3 1,732,360 / libdartjni 131,248 |
| `lib/armeabi-v7a/` 合计 | 25,060,556 | libflutter 8,615,468 / libapp 10,994,248 / librhttp 3,651,244 / libsqlite3 1,713,736 / libdartjni 81,444 |
| 三 ABI native 合计 | **85,891,196**（≈85.9 MB，占 APK 97%） | 与主会话结论一致 |
| `classes.dex` | 1,564,888 | R8 已开启（见 §A.6） |
| `assets/flutter_assets/packages/cupertino_icons/assets/CupertinoIcons.ttf` | 257,628 | `lib/` 中 `rg CupertinoIcons|cupertino_icons` 零命中；`pubspec.yaml:15` 仍是直接依赖 |
| `assets/flutter_assets/NOTICES.Z` | 123,349 | Flutter 许可证包，不可移除 |
| `assets/flutter_assets/assets/stamps/*.jpg` | 40 个文件，磁盘合计 347,323 | `pubspec.yaml:54` 声明 |
| `assets/flutter_assets/assets/emojis/` | 38 个文件，磁盘合计 30,879 | `pubspec.yaml:53` |
| `assets/icon.ttf` | 5,964 | `pubspec.yaml:57-60` 自定义 iconFont |
| `MaterialIcons-Regular.otf` | 13,176 | 已 tree-shake（构建日志：1,645,184 → 13,176） |
| `resources.arsc` | 91,884 | 含 androidx.preference/appcompat 资源（来源见 §A.5） |
| `AndroidManifest.xml` | 14,904 | |
| `kotlin/kotlin.kotlin_builtins` | 30,920 | Kotlin stdlib 元数据 |

`build/app/outputs/apk/github/release/output-metadata.json`：`"type": "SINGLE"`, `"filters": []`, `"versionCode": 1`, `"versionName": "0.1.0"`, `"minSdkVersionForDexing": 29` —— 即当前是单一 universal APK。

### 0.2 `app-fdroid-release.apk`（42,210,620 B，2026-09-07 16:59，主会话 `--target-platform android-arm64 --analyze-size` 产物）

终端记录（terminals/945322.txt 第 4 行）命令为
`flutter build apk --release --flavor fdroid --target-platform android-arm64 --analyze-size --code-size-directory=/tmp/pixiv-size`。

`unzip -l` 结果：

| ABI 目录 | 字节 | 内容 |
|---|---|---|
| `lib/arm64-v8a/` | 29,020,544 | 与 github APK 的 arm64 集合逐字节相同 |
| `lib/x86_64/` | 6,824,992 | **只有** librhttp.so 6,702,128 + libdartjni.so 116,640 + libdatastore_shared_counter.so 6,224 |
| `lib/armeabi-v7a/` | 3,737,104 | **只有** librhttp.so 3,651,244 + libdartjni.so 81,444 + libdatastore_shared_counter.so 4,416 |

**这说明两件事（都有直接证据）：**

1. `--target-platform` 只约束 Flutter 自己的 `libflutter.so`/`libapp.so` 和 native asset（`libsqlite3.so`）。插件 AAR 自带的 jniLibs（`jni` 的 `libdartjni.so`、`shared_preferences_android`→datastore 的 `libdatastore_shared_counter.so`）和 cargokit 的输出目录只受 `abiFilters` 约束，而 Flutter Gradle 插件在非 split 模式下把 `abiFilters` 固定为 `PLATFORM_ABI_LIST = [armeabi-v7a, arm64-v8a, x86_64]`（`/opt/flutter-3.47.0/packages/flutter_tools/gradle/src/main/kotlin/FlutterPlugin.kt:561-599` `configureAbiWithoutSplits`；`FlutterPluginConstants.kt:48-63`），与 `--target-platform` 无关。
2. 这次 arm64 构建里的 x86_64 / armeabi-v7a `librhttp.so` 是**上一次 3-ABI 构建的陈旧产物**：
   - `build/rhttp/jniLibs/release/armeabi-v7a/librhttp.so` mtime `16:09:39`、`x86_64/librhttp.so` `16:09:39`（属于 16:11 的 github 构建）；`arm64-v8a/librhttp.so` mtime `16:58:11`（属于 16:59 的 fdroid 构建）。
   - cargokit 输出目录按 buildType 而非 flavor/ABI 集合划分：`plugins/rhttp/rhttp/cargokit/gradle/plugin.gradle:134` `cargoOutputDir = "${project.buildDir}/jniLibs/${buildType}"`；`cargokit/build_tool/lib/src/build_gradle.dart:34-47` 只 `copySync` 本次请求的 target，**从不删除**未请求的 ABI 子目录。
   - 这意味着在非 clean 树上做 `--target-platform`/`--split-per-abi` 构建时，Rust 库会跨 ABI 集合泄漏（本次泄漏 10,353,372 B）。CI 每次 fresh checkout 不受影响；本地和任何复用 `build/` 的流程受影响。

---

## A. Android 原生层（Kotlin / Manifest / res / Gradle）

### A.1 文件清单（LOC 用 `wc -l`）

`android/app/src/main/kotlin/io/github/lopution/pixivfunc/`：

| 文件 | LOC | 用途 |
|---|---|---|
| `MainActivity.kt` | 35 | `FlutterActivity`；`configureFlutterEngine` 依次注册 8 个 channel 对象（10-17 行）；`onNewIntent`→`AndroidIntentChannel.dispatch`；`onActivityResult` 分派给 `ReverseImageInputChannel`/`SafTreeChannel` |
| `MediaStoreChannel.kt` | 247 | `pixivfunc/mediastore`：scoped MediaStore pending 写入（begin/write/finalize/abort/listPending/abortPending），`LruCache<Int, OutputStream>(32)` 持有流 |
| `ReverseImageInputChannel.kt` | 233 | `pixivfunc/reverse_image_input`：`ACTION_OPEN_DOCUMENT` 选图、`copyToTemp`（10 MiB 上限，64 KiB 缓冲）、`deleteTemp`、`openExternal`（仅 https） |
| `SafTreeChannel.kt` | 147 | `pixivfunc/saf_tree`：`ACTION_OPEN_DOCUMENT_TREE` + `takePersistableUriPermission`；`DocumentsContract.createDocument` 写入流 |
| `AccountTransferClipboardChannel.kt` | 146 | `pixivfunc/account_transfer_clipboard`：write/read/clearIfCurrent/capabilities；SHA-256 指纹；`Handler.postDelayed` 延迟清除 |
| `AndroidIntentChannel.kt` | 124 | `pixivfunc/android_intents` MethodChannel（getInitialIntent/openUrl）+ `pixivfunc/android_intents/events` EventChannel |
| `WebProfileChannel.kt` | 49 | `pixivfunc/webprofile`：`android.webkit.CookieManager` 读/清 `https://www.pixiv.net` cookie |
| `WidgetForegroundChannel.kt` | 67 | `pixivfunc/widget`：notifySnapshotChanged/clearSnapshot/requestRefresh/hasAnyWidget |
| `appwidget/WidgetUpdateCoordinator.kt` | 178 | WorkManager：30 min 周期唯一任务 + 一次性刷新（KEEP 策略）；内含 `WidgetBackgroundWorker` |
| `appwidget/WidgetHeadlessRunner.kt` | 115 | 在主线程创建 headless `FlutterEngine` 跑 `widgetBackgroundMain`；作为 `pixivfunc/widget_background` 的**接收端**；4 min 闩锁 |
| `appwidget/WidgetRenderer.kt` | 284 | RemoteViews 渲染、位图预算、PendingIntent（`pixivfunc://illusts/<id>` 深链） |
| `appwidget/WidgetSnapshotState.kt` | 201 | 读取 Dart 写的 `files/widget_snapshot/active.json`（`org.json`），schemaVersion=1，多重边界校验 |
| `appwidget/WidgetBudget.kt` | 89 | 纯位图/IPC 预算数学 |
| `appwidget/WidgetClickReceiver.kt` | 37 | 刷新按钮广播接收器 |
| `appwidget/RecommendWidgetProvider.kt` | 45 | 推荐 widget provider |
| `appwidget/RefreshWidgetProvider.kt` | 33 | 刷新 widget provider |
| **main 合计** | **2,030** | |

Flavor source sets：

| 文件 | LOC | 用途 |
|---|---|---|
| `android/app/src/github/kotlin/.../DistributionUpdaterChannel.kt` | 233 | `pixivfunc/updater` 完整实现：getCapability/getPlatformInfo/verifyManifestSignature（`SHA256withECDSA`）/verifyApk/installApk/deleteApk |
| `android/app/src/fdroid/kotlin/.../DistributionUpdaterChannel.kt` | 74 | 同名 channel，仅 getCapability/getPlatformInfo；其余 `result.error("disabled", …)`（28-32 行） |
| `android/app/src/github/AndroidManifest.xml` | 5 | 仅 `REQUEST_INSTALL_PACKAGES` |
| `android/app/src/fdroid/AndroidManifest.xml` | 1 | 空 manifest |
| `android/app/src/debug/AndroidManifest.xml`、`profile/AndroidManifest.xml` | 7 / 7 | Flutter 模板：`INTERNET`（main 已声明，属冗余模板） |

其他：

| 文件 | LOC | 备注 |
|---|---|---|
| `android/app/src/main/java/io/flutter/plugins/GeneratedPluginRegistrant.java` | 54 | Flutter 生成（`android/.gitignore:7` 忽略）；注册 7 个插件：`flutter_secure_storage`、`jni`、`jni_flutter`、`rhttp`、`shared_preferences_android`、`sqflite_android`、`webview_flutter_android` |
| `android/app/src/test/kotlin/.../appwidget/{WidgetBudgetTest 85, WidgetSnapshotStateTest 37, WidgetWorkContractTest 26}.kt` | 148 | JUnit4 纯 JVM 测试；**CI 未执行**（`.github/workflows/ci.yml` 只有 `flutter analyze`/`flutter test`，`rg gradlew` 零命中） |

两份 `DistributionUpdaterChannel.kt` 中 `platformInfo`/`packageInfo`/`signerSha256` 三个函数逐字重复（github 101-115、197-204、215-226 ↔ fdroid 37-51、53-60、62-73）。

### A.2 平台通道契约表（Kotlin 处理端 ↔ Dart 调用端）

| Channel 名 | Kotlin 端 | Dart 端 | 方法 |
|---|---|---|---|
| `pixivfunc/mediastore` | `MediaStoreChannel.kt:24` | `lib/core/platform/media_store_channel.dart:11,30` | begin / write / finalize / abort / listPending / abortPending |
| `pixivfunc/reverse_image_input` | `ReverseImageInputChannel.kt:21` | `lib/core/reverse_image/reverse_image_platform.dart:38,47`；`reverse_image_external.dart:10` | pickImage / copyToTemp / deleteTemp / openExternal |
| `pixivfunc/saf_tree` | `SafTreeChannel.kt:15` | `lib/core/platform/saf_tree.dart:40` | pickTree / create / write / close / delete |
| `pixivfunc/account_transfer_clipboard` | `AccountTransferClipboardChannel.kt:24` | `lib/core/platform/account_transfer_clipboard.dart:51,63` | write / read / clearIfCurrent / capabilities |
| `pixivfunc/android_intents` | `AndroidIntentChannel.kt:19` | `lib/core/platform/android_intent_channel.dart:10,30,60` | getInitialIntent / openUrl |
| `pixivfunc/android_intents/events`（EventChannel） | `AndroidIntentChannel.kt:20,33-42` | `android_intent_channel.dart:11,61` | onNewIntent 推送 encode(intent) |
| `pixivfunc/webprofile` | `WebProfileChannel.kt:20` | `lib/core/profile/web_profile_session.dart:43,11` | readSession / clearSession |
| `pixivfunc/widget` | `WidgetForegroundChannel.kt:22` | `lib/core/widget/widget_channel.dart:12,54` | notifySnapshotChanged / clearSnapshot / requestRefresh / hasAnyWidget |
| `pixivfunc/widget_background`（方向反转：Dart 调用、原生接收） | `WidgetHeadlessRunner.kt:30,57-67` | `lib/core/widget/widget_background.dart:17-19,67` | result{outcome} |
| `pixivfunc/updater` | github `DistributionUpdaterChannel.kt:25`；fdroid 同名 `:14` | `lib/core/updater/update_platform.dart:27` | getCapability / getPlatformInfo / verifyManifestSignature / verifyApk / installApk / deleteApk |
| `pixivfunc/login_webview_intercept` | **未找到任何 Kotlin 处理端**（`rg login_webview_intercept|shouldInterceptRequest android/ plugins/` 零命中） | `lib/features/login/login_intercept_controller.dart:29,41-43`（Dart 端 `setMethodCallHandler`，等待原生调用 `fetchWithPolicy`） | fetchWithPolicy |

`login_webview_intercept` 的 Dart 注释（9-15 行）称"Native `shouldInterceptRequest` calls `fetchWithPolicy(url)`"，但**工作树**内没有对应实现；该 channel 目前只有 Dart 单侧。原因可从 git 看到：`git status` 显示 `D android/app/src/main/kotlin/io/github/lopution/pixivfunc/LoginWebViewPlatformView.kt`（HEAD `36433d7` 中存在，430 行，定义 `CHANNEL = "pixivfunc/login_webview_intercept"`、`EVENTS_CHANNEL = "pixivfunc/login_webview_intercept_events"`、`shouldInterceptRequest` 于 98 行，`LoginWebViewFactory` 于 417 行；HEAD 的 `MainActivity.kt:18-21` 注册了该 PlatformView），它是**未提交的本地删除**。同一批未提交改动还包括 `android/app/build.gradle.kts`（+97）、github `DistributionUpdaterChannel.kt`（±44）、`MainActivity.kt`（±12）、`MediaStoreChannel.kt`（±25）、`WidgetForegroundChannel.kt`（+20）、`launch_background.xml`、`lib/core/platform/android_platform_interfaces.dart`（`git diff --stat`：8 files, +181/−460）。本文描述的是工作树状态；若这些改动被回退，A.1/A.2 需重新核对。

### A.3 载荷 / 线程 / 错误传播 / API 分支约定

**载荷**：全部走 `StandardMessageCodec` 的 `Map<String, Any?>`；字节用 `ByteArray`/`Uint8List`（`MediaStoreChannel.kt:45`、`SafTreeChannel.kt:62`、`DistributionUpdaterChannel.kt:66-67`，Dart 端 `update_platform.dart:80-81` 显式 `Uint8List.fromList`）。无 JSON 字符串载荷。唯一标量参数：`WidgetForegroundChannel.kt:29` `call.arguments as? Number`。`listPending` 返回 `List<Map>`（`MediaStoreChannel.kt:179-213`）。Widget 快照跨进程数据不走 channel，而是文件 `filesDir/widget_snapshot/active.json`（`WidgetSnapshotState.kt:53-54`）。

**线程**：所有 `MethodChannel` 处理器在主线程执行（默认行为，无 `BinaryMessenger.TaskQueue`）。主线程上发生的同步 IO：MediaStore `stream.write(bytes)`（`MediaStoreChannel.kt:134`）、SAF `stream.write`（`SafTreeChannel.kt:114`）、反查图片复制循环（`ReverseImageInputChannel.kt:147-158`）、updater `getPackageArchiveInfo` + 全量 APK 签名解析（`DistributionUpdaterChannel.kt:127,206-213`）、`CookieManager.flush()`（`WebProfileChannel.kt:39`）。后台线程只有 WorkManager 的 `WidgetBackgroundWorker.doWork`（`WidgetUpdateCoordinator.kt:150`），它把引擎创建/销毁 post 回主线程（`WidgetHeadlessRunner.kt:43,100`）。

**错误传播**三种风格并存：
1. 全捕获 `catch (error: Exception) { result.error("<prefix>_error", error.message, null) }`：`MediaStoreChannel.kt:64-66`（`mediastore_error`）、`SafTreeChannel.kt:78-80`（`saf_error`）、`WebProfileChannel.kt:44-46`（`webprofile_error`）。这些处理器大量使用 `call.argument<…>("…")!!`（`MediaStoreChannel.kt:37-38,44-45,49,58-59`；`SafTreeChannel.kt:55-57,61-62,67,72`），缺参会以 NPE 落入同一个笼统错误码。
2. 细分错误码：`ReverseImageInputChannel.kt:212-222` `codeFor/messageFor`；`AndroidIntentChannel.kt:56,64`；`AccountTransferClipboardChannel.kt:74,103,116`；`SafTreeChannel.kt:29,47,51,143`。
3. updater 不用 `result.error`，而是返回 `{valid:false, errorCode}` / `{status:"failed", errorCode}` 的 Map（`DistributionUpdaterChannel.kt:63-99,228-232`），Dart 端 `update_platform.dart:153-170` 再把 `PlatformException.code` 映射为 `UpdatePlatformException`。

**API 级分支**（`rg Build.VERSION.SDK_INT`，共 15 处，`minSdk = 29` 见 `build.gradle.kts:29`）：
- 在 minSdk 29 下**恒为真/恒不触发的死分支**（`P`=28、`O`=26、`Q`=29）：`MediaStoreChannel.kt:71,154,169`（`< Q`）；`AccountTransferClipboardChannel.kt:134`（`>= P`）；github `DistributionUpdaterChannel.kt:103,150,199,208,216`；fdroid `:39,55,63`。共 12 处。
- 仍有意义的：`TIRAMISU`(33) 3 处：`AndroidIntentChannel.kt:88`、`AccountTransferClipboardChannel.kt:55,79`。

### A.4 `AndroidManifest.xml`（`android/app/src/main/AndroidManifest.xml`，124 行）

| 项 | 观测 |
|---|---|
| 权限 | main 仅 `INTERNET`（2 行）；github 追加 `REQUEST_INSTALL_PACKAGES`；debug/profile 重复 `INTERNET` |
| `<application>` | `label="Pixiv Func"`、`name="${applicationName}"`、`icon=@mipmap/ic_launcher`；**未声明** `networkSecurityConfig`、`usesCleartextTraffic`、`allowBackup`、`dataExtractionRules`、`fullBackupContent` |
| `MainActivity` | `exported="true"`、`singleTop`、`taskAffinity=""`；intent-filter：MAIN/LAUNCHER（24-27）、`SEND image/*`（29-33）、`VIEW https://pixiv.net|www.pixiv.net`（35-42）、`VIEW http://…`（43-50）、`pixiv://users|illusts|account`（52-60）、`pixivfunc://users|illusts`（62-69）。无 `android:autoVerify` |
| `FileProvider` | `androidx.core.content.FileProvider`，`${applicationId}.fileProvider`，`exported=false`，`grantUriPermissions=true`；paths：`cache/shared_images/`、`files/updates/`（`res/xml/file_provider_paths.xml`） |
| receivers | `RecommendWidgetProvider`（APPWIDGET_UPDATE + OPTIONS_CHANGED）、`RefreshWidgetProvider`（APPWIDGET_UPDATE）、`WidgetClickReceiver`——三者均 `exported="false"` |
| services / providers（自定义） | 无 |
| `<queries>` | `PROCESS_TEXT text/plain`（Flutter 模板） |
| 网络安全配置 | 无 `res/xml/network_security_config.xml` |

值得标记的历史/模板残留：`http://pixiv.net` 明文 scheme 深链 filter（43-50 行）；`drawable-v21/launch_background.xml` 与 `drawable/launch_background.xml` 内容等价（都只有 `?android:colorBackground`），minSdk 29 ≥ 21 使非 v21 版本永不被选中；debug/profile manifest 的 `INTERNET` 重复。

### A.5 `android/app/src/main/res/`

16 个文件，`du -sb` 合计 **10,929 B**：

| 目录 | 字节 | 内容 |
|---|---|---|
| `layout/` | 1,566 | `recommend_app_widget.xml`(1,129)、`refresh_app_widget.xml`(437) |
| `xml/` | 1,478 | `recommend_app_widget_info`、`refresh_app_widget_info`、`file_provider_paths` |
| `mipmap-{mdpi…xxxhdpi}/ic_launcher.png` | 442 / 544 / 721 / 1,031 / 1,443 | 5 个极小 PNG |
| `values/` + `values-night/` | 1,330 + 995 | `strings.xml`（4 个 widget 字串）、`styles.xml`（LaunchTheme/NormalTheme） |
| `drawable/` + `drawable-v21/` | 941 + 438 | `icon_refresh.xml` 矢量、两份等价 `launch_background.xml` |

**无大图/raw 资源。** R8 资源可达性（`build/app/outputs/mapping/githubRelease/resources.txt`）显示 app 自有资源全部 reachable（行 18、35、49、77、107、108、113、130、159、191、229、281、284、286、292、298、300、321）；"is not reachable" 条目全部来自 `androidx.preference`（例如 902-906 行 `layout:preference_widget_*`），它们由 `shared_preferences_android 2.4.27` 的 Gradle 依赖带入（`~/.pub-cache/hosted/pub.dev/shared_preferences_android-2.4.27/android/build.gradle.kts:62-64`：`androidx.datastore:datastore:1.1.7`、`datastore-preferences:1.1.7`、`androidx.preference:preference:1.2.1`），并被 `shrinkResources` 移除。`libdatastore_shared_counter.so` 亦来自 datastore。

### A.6 Gradle 配置

`android/app/build.gradle.kts`（174 行）：

| 项 | 观测（行号） |
|---|---|
| SDK | `compileSdk = flutter.compileSdkVersion`、`ndkVersion = flutter.ndkVersion`（9-10）；`minSdk = 29`（29，注释 25-28 说明 MediaStore 依赖）；`targetSdk = flutter.targetSdkVersion`（30） |
| versionCode | `flutter.versionCode`/`versionName`（35-36），来源 `pubspec.yaml:7` `0.1.0+1` → `local.properties` `flutter.versionName=0.1.0`/`flutter.versionCode=1`。注释 31-34 明确写着 split APK 时 Flutter 自动加 `1000 * ABI_VERSION`，可用 `-Pforce-version-code-ignoring-abi=true` 关闭 |
| flavor | 维度 `distribution`：`github`（71-103）、`fdroid`（104-109）；BuildConfig 字段 `UPDATE_SELF_UPDATER_ENABLED`、`UPDATE_PUBLIC_KEY_DER_B64`、`RELEASE_SIGNED_OFFICIALLY` |
| 签名 | 由 Gradle 属性 `PIXIV_RELEASE_KEYSTORE*` 组装 `pixivRelease` signingConfig（43-68）；`-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true` 可用 debug key；`verifyGithubReleaseSigning` 任务挂在 `assembleGithubRelease`/`bundleGithubRelease`/`packageGithubRelease` 前（125-156） |
| ABI | **无** `splits`、**无** `ndk.abiFilters`、**无** `packaging`/`packagingOptions`、**无** `bundle {}` 块 |
| R8 | 未在 app 里配置；由 Flutter Gradle 插件对 release 开启：`FlutterPlugin.kt:218-223` `isMinifyEnabled = true`、`isShrinkResources = isBuiltAsApp`、`proguard-android-optimize.txt` + `flutter_proguard_rules.pro`；`android/app/proguard-rules.pro` **不存在**（`FlutterPlugin.kt:224-226` 只在存在时追加） |
| 依赖（168-173） | `androidx.core:core:1.15.0`（仅用于 `FileProvider`：manifest 78 行 + `DistributionUpdaterChannel.kt:11,160`）；`androidx.work:work-runtime-ktx:2.11.2`；`testImplementation junit:junit:4.13.2` |
| Kotlin | `jvmTarget = JVM_17`（158-162）；Java 17（18-21） |

**`work-runtime-ktx` 能否换成 `work-runtime`**：本机 Gradle 缓存中 `androidx.work/work-runtime-ktx/2.11.2/…/work-runtime-ktx-2.11.2.aar` 只有 6,103 B，`classes.jar` 199 B（空壳，仅 manifest/许可证/`aar-metadata`）；`work-runtime-2.11.2.aar` 的 `classes.jar` 已包含 `androidx/work/CoroutineWorker.class`、`ListenableFutureKt.class` 等 Kotlin 扩展。因此 ktx→非 ktx 是**纯声明层面的整理，不影响体积**（已测量：ktx 工件本身几乎为空）。

`android/settings.gradle.kts`（26 行）：从 `local.properties` 读取 `flutter.sdk`（`require` 强制，3-9 行）；`includeBuild("$flutterSdkPath/packages/flutter_tools/gradle")`；插件版本 `dev.flutter.flutter-plugin-loader 1.0.0`、`com.android.application 9.1.0`、`org.jetbrains.kotlin.android 2.4.0 apply false`。
`android/gradle.properties`：`-Xmx3G`、`android.useAndroidX=true`、`android.newDsl=false`、`android.builtInKotlin=true`（AGP 内建 Kotlin，KGP 未应用）。
`android/gradle/wrapper/gradle-wrapper.properties`：`gradle-9.3.1-all.zip`。
`android/local.properties`：`sdk.dir=/opt/android-sdk`、`flutter.sdk=/opt/flutter-3.47.0`、`flutter.buildMode=release`、`flutter.versionName=0.1.0`、`flutter.versionCode=1` —— 被 `android/.gitignore:6` 忽略，由 flutter 工具每次重写；对 `/opt/flutter-3.47.0` 的耦合只存在于本机，CI 用 `subosito/flutter-action@v2 flutter-version: "3.47.0"` 重新生成。

插件侧 `plugins/rhttp/rhttp/android/build.gradle.kts`（171 行）：`compileSdk = 36`、`minSdk = 24`（低于 app 的 29）；`buildscript.dependencies.classpath` 声明 AGP `8.11.2`、Kotlin `2.2.20`、`kotlinx-serialization-json 1.11.0`（13-25 行，与 app 侧 AGP 9.1.0 / Kotlin 2.4.0 不同版本）；注释 38-41 说明为 AGP 9 built-in Kotlin 做过适配（"Upstream 0.18.0 predates this"）；配置期执行 `cargo --version`（153-166）与 `cargo metadata --format-version 1`（120-124）定位 `rustls-platform-verifier-android` 的 AAR 并解出 `classes.jar`（139-151）。`android/gradle.properties` 设 `android.builtInKotlin=false`、`android.useAndroidX=false`。

### A.7 工作树中的杂项文件

`android/hs_err_pid2072614.log`（358,524 B，8/28）、`android/hs_err_pid2335473.log`（837,614 B，8/29）为 JVM 崩溃日志；`android/pixiv_func_android.iml`（1,601 B）；`android/.kotlin/sessions/`。`git status --ignored` 显示三者均为 `!!`（已忽略，未跟踪），只是本地噪音。

---

## B. Rust 插件 `plugins/rhttp/rhttp`

### B.1 fork 现状（`plugins/rhttp/UPSTREAM.md`）

- 上游 `https://codeberg.org/Tienisto/rhttp`，commit `9871f1c25d0cf75553af85698b9f03d38f438aed`（2026-07-08），版本 0.18.0（`pubspec.yaml:3`），MIT。
- 声明的唯一功能性差异是 ECH：`rust/Cargo.toml` 加 `rustls = "0.23"`（aws-lc-rs provider）；`rust/src/api/client.rs` `TlsSettings.ech_config_list: Option<Vec<u8>>`（58-74 行）、`tls_backend_preconfigured(build_ech_tls_config(...))` 注入（215-219 行）、`build_ech_tls_config`（文档注释 376 起、函数 387 起）。`rg -i ech client.rs` 命中约 40 行。
- UPSTREAM.md 未列出的 fork 侧文件：`android/build.gradle.kts` 的 AGP 9 适配（见 §A.6）、`rust/tests/ech_config_test.rs`（17 行）、`rust/tests/ech_live_handshake.rs`（124 行）、`rust/examples/ech_reqwest_probe.rs`（84 行）。
- 明确不做：HTTP/3/QUIC（UPSTREAM.md 34-35："compiled upstream, unused by this app"；但见 §B.3——本 fork 的依赖图里**并没有** quinn/h3）。
- 同步指引（UPSTREAM.md 37-44）：diff 上游 → 重打两处 diff → 如 Rust 签名变化则 `dart run flutter_rust_bridge_codegen generate` → 更新 UPSTREAM.md。

### B.2 Cargo features 与 App 实际用量对照

`rust/Cargo.toml`（66 行）关键声明：`reqwest 0.13.4 default-features=false`，features = `charset, cookies, form, http2, query, rustls, stream, multipart, socks, brotli, deflate, gzip, zstd`（32-51）；`rustls 0.23` features `aws_lc_rs, std, tls12, logging`（14-19）；`tokio 1.52 features=["full"]`（21）；`flutter_rust_bridge = "=2.12.0" features=["chrono"]`（11）；`chrono 0.4.45`；`webpki-root-certs 1.0`；Android 目标额外 `jni 0.22`、`ndk-context 0.1`、`rustls-platform-verifier 0.7`（27-30）。

App（`lib/`，非插件）中 rhttp 的引用只有 4 个文件：`lib/main.dart`、`lib/core/widget/widget_background.dart`、`lib/core/network/compat/rhttp_client_factory.dart`、`lib/core/network/compat/network_contracts.dart`（`rg -l package:rhttp lib`）。注意 `lib/` 中 119 处 `CancelToken` 是 app 自己的类（`lib/core/network/pixiv_http_client.dart:23`），20 处 `baseUrl` 也不是 rhttp 的 `ClientSettings.baseUrl`。

| rhttp/reqwest 能力 | App 是否使用 | 证据 |
|---|---|---|
| `Rhttp.init()` | 是 | `lib/main.dart:24`（`RhttpGate.ready = rhttp.Rhttp.init()`）、`widget_background.dart:35` |
| `RhttpCompatibleClient.createSync` (package:http 兼容层) | 是（唯一的客户端入口） | `rhttp_client_factory.dart:68-74` |
| `ClientSettings` | 是 | `rhttp_client_factory.dart:101-111`（只传 `httpVersionPref/redirectSettings/timeoutSettings/tlsSettings/dnsSettings`） |
| `TlsSettings{sni, verifyCertificates, echConfigList}` | 是 | `rhttp_client_factory.dart:85-92` |
| `TlsSettings.rootCertSource` | 未显式设置 → Dart 默认 `RootCertSource.webpki`（`plugins/rhttp/rhttp/lib/src/model/settings.dart:226-227,262`） | 注释 `rhttp_client_factory.dart:14-16`："we keep webpki (bundled Mozilla roots) for determinism" |
| `DnsSettings.static(overrides)` | 是 | `rhttp_client_factory.dart:94-100` |
| `TimeoutSettings{connectTimeout, timeout}` | 是 | `rhttp_client_factory.dart:125-130` |
| `RedirectSettings.none()` | 是 | `rhttp_client_factory.dart:108` |
| `HttpVersionPref.all`（ALPN h2/h1.1） | 是 | `rhttp_client_factory.dart:107`（注释：HTTP/3 remains disabled） |
| 异常族 `Rhttp*Exception` 分类 | 是 | `network_contracts.dart:366-477` |
| 流式响应（reqwest `stream`） | 间接是 | 兼容层内部 `compatible_client.dart:74` `client.requestStream(...)`；图片/下载走该路径 |
| `ProxySettings`（reqwest `socks`） | 否 | `rg ProxySettings lib` 零命中；`socks` 只在 `socks://` 代理 URL 时用到 |
| `HttpBody.multipart` / `HttpBody.form`（reqwest `multipart`/`form`） | 否 | `rg "HttpBody\." lib` 零命中；Rust 侧 `http.rs:413-424` 仅插件内部使用 |
| `CookieSettings`（reqwest `cookies`） | 否 | `rg CookieSettings|cookieSettings lib` 零命中；`client.rs:166` 仅在传入时启用 |
| `query` 参数（reqwest `query`） | 否 | app 自行拼 URL；`http.rs:376` 为插件通用路径 |
| `charset`（`text()` 解码） | 否 | 兼容层返回字节，`http.Response.fromStream` 在 Dart 解码 |
| `onSendProgress` / `onReceiveProgress` / `requestStream` 直调 / `RetryInterceptor` / `Interceptor` / `IoCompatibleClient` / `BaseUrl` / 证书 pinning / 客户端证书 / `HttpVersionPref.http3` | 否 | `rg` 对应符号在 `lib/` 零命中 |
| `brotli/deflate/gzip/zstd` 透明解压 | 间接（reqwest 自动加 `Accept-Encoding` 并解压，app 无感知） | 无 app 侧代码引用；是否影响 Pixiv 响应传输量**待测量** |
| `rustls-platform-verifier`（系统根证书） | 运行时初始化但 TLS 路径未用 | `init.rs:70` 每进程调用一次 `rustls_platform_verifier::android::init_with_env`；`client.rs:455` 只在 `RootCertSource::Platform` 时构造 Verifier，app 走 webpki |
| `chrono` + FRB `chrono` feature | 是 | `TimeoutSettings` 的 `Duration` 映射 |
| `tokio full` | 部分 | 插件运行时需要 rt/time/sync/macros；`full` 额外拉入 fs/process/signal/io-std 等 |

### B.3 依赖树（`cargo tree -e normal --target aarch64-linux-android`，cargo 1.98.0）

解析后 **183 个唯一 crate**（`/tmp/rhttp-tree-aarch64.txt`）。存在/不存在：

| crate | 版本 | 在 aarch64 图中 | 由谁拉入（`cargo tree -i`） |
|---|---|---|---|
| `aws-lc-rs` / `aws-lc-sys` | 1.17.0 / 0.41.0 | 是 | `rustls`（feature `aws_lc_rs`）← `reqwest rustls`、`rhttp` 直接依赖、`rustls-platform-verifier`、`tokio-rustls`。**唯一的 TLS 加密后端**（`ring` 不在图中） |
| `quinn` / `quinn-proto` / `h3` | 锁文件有 quinn 0.11.9 | **否** | 未启用 `http3`，Cargo.lock 里的条目只是 reqwest 的可选依赖记录 |
| `brotli` / `brotli-decompressor` | 8.0.3 / 5.0.1 | 是 | `compression-codecs` ← `async-compression` ← `tower-http decompression-br` ← `reqwest brotli` |
| `zstd` / `zstd-safe` / `zstd-sys`（C 源码经 `cc` 编译） | 0.13.3 / 7.2.4 / 2.0.16+zstd.1.5.7 | 是 | 同上链路 ← `reqwest zstd` |
| `flate2` / `miniz_oxide` | 1.1.9 / 0.8.9 | 是 | ← `reqwest gzip`/`deflate` |
| `cookie_store` / `publicsuffix` / `cookie` | 0.22.1 / 2.3.0 / 0.18.1 | 是 | ← `reqwest cookies`（`publicsuffix` 仅由 cookie_store 拉入） |
| `idna` / `idna_adapter` / `icu_normalizer` / `icu_properties` | 1.1.0 / 1.2.2 / 2.2.0 / 2.2.0 | 是 | ← `url`（reqwest/tower-http 必需）**和** cookie_store；去掉 cookies 不会去掉 icu |
| `mime_guess` | 2.0.5 | 是 | ← `reqwest`（multipart 路径） |
| `serde_urlencoded` | 0.7.1 | 是 | ← `reqwest`（query/form） |
| `encoding_rs` | 0.8.35 | 是 | ← `reqwest charset` |
| `hyper 1.10.1` / `hyper-util 0.1.20` / `hyper-rustls 0.27.9` / `h2 0.4.14` / `tower 0.5.3` / `tower-http 0.6.11` | — | 是 | reqwest 0.13 核心 |
| `tokio-socks` | — | 否 | reqwest 0.13 的 `socks` 不再依赖该 crate |
| `rustls-platform-verifier` / `-android` | 0.7.0 / 0.1.1 | 是 | `rhttp` 直接 + `reqwest` |
| `webpki-root-certs` | 1.0.7 | 是 | `rhttp` 直接（client.rs:244,288,365-366,492） |
| `jni` / `ndk-context` | 0.22.4 / 0.1.1 | 是 | Android 初始化（init.rs:23-28,63-70） |
| `flutter_rust_bridge` / `allo-isolate` / `dart-sys` | 2.12.0 / 0.1.27 / 4.1.5 | 是 | FFI 桥 |
| `chrono` | 0.4.45 | 是 | 直接 |

Rust 源码规模：`rust/src/api/client.rs` 638、`http.rs` 525、`init.rs` 88、`stream.rs` 37、`error.rs` 36、`utils/socket_addr.rs` 40、`frb_generated.rs` 146,198 B（生成）。Dart 生成物：`lib/src/rust/frb_generated.dart` 4,352 行、`frb_generated.io.dart` 1,597、`frb_generated.web.dart` 1,557、`api/*.freezed.dart` 合计 2,939 行。

### B.4 `[profile.release]`（`Cargo.toml:56-61`）

```toml
opt-level = 3
strip = true
panic = "abort"
lto = true
codegen-units = 1
```

已具备 `strip`/`lto`/`codegen-units=1`/`panic=abort`。体积导向的剩余选项是 `opt-level = "s"` 或 `"z"`（以换取吞吐/延迟为代价，对 TLS/解压热路径的影响需要实测；本次未构建，**待测量**）。`[profile.dev]` 设 `opt-level = 3`、`debug-assertions = false`（62-66），debug 构建的 `librhttp.so` 为 107–135 MB/ABI（`build/rhttp/jniLibs/debug/*`），仅影响本地 debug 包。

### B.5 cargokit 构建哪些 ABI，以及与 app ABI 集合的关系

- 目标集来自 Flutter 插件：`plugin.gradle:138` `FlutterPluginUtils.getTargetPlatforms(project)`；未传 `--target-platform` 时为 `DEFAULT_PLATFORMS = [android-arm, android-arm64, android-x64]`（`FlutterPluginConstants.kt:48-53`）。debug 额外加 `android-x86`、`android-x64`（`plugin.gradle:141-144`）。所以 release 下 cargokit 与 Flutter 一致地构建 3 个 ABI；`--split-per-abi`/`--target-platform` 会让 cargokit 只编对应 target（`build_gradle.dart:23-30`）。
- Rust target 映射：`cargokit/build_tool/lib/src/target.dart:21-43`（`armv7-linux-androideabi`/`aarch64-linux-android`/`i686-linux-android`/`x86_64-linux-android`）。
- 输出目录复用问题见 §0.2（陈旧 ABI 不清理；按 buildType 共享）。`plugin.gradle:170-176` 把 `merge<BuildType>NativeLibs` 标记为 `upToDateWhen { false }`，保证每次重新合并——恰好也保证陈旧文件每次都被合并进去。
- 无 `cargokit.yaml`（`find … -name cargokit.yaml` 零命中）、无 `rust-toolchain*`；工具链取自本机 `~/.cargo`（`build.gradle.kts:80-118` `findCargo()`），CI 依赖 `ubuntu-latest` 镜像自带的 rustup/cargo（工作流未安装 Rust target，见 §C.4）。

### B.6 flutter_rust_bridge 版本一致性

| 位置 | 版本 |
|---|---|
| Dart 运行时（`pubspec.lock` `flutter_rust_bridge`） | **2.13.0**（`~/.pub-cache/hosted/pub.dev/flutter_rust_bridge-2.13.0/lib/src/misc/version.dart:2` `kFlutterRustBridgeRuntimeVersion = '2.13.0'`） |
| 生成代码 `codegenVersion`（`plugins/rhttp/rhttp/lib/src/rust/frb_generated.dart:74`） | **2.12.0**（`rustContentHash = 2036241710`，77 行） |
| `rust/Cargo.toml:11` | `flutter_rust_bridge = "=2.12.0"` |
| 本机 `~/.cargo/bin/flutter_rust_bridge_codegen --version` | 2.12.0 |
| 插件 `pubspec.yaml:21` | `flutter_rust_bridge: ^2.12.0`（允许解析到 2.13.0，这就是漂移来源） |

漂移当前"能跑"的原因：`plugins/rhttp/rhttp/lib/src/rhttp.dart:16-19` 调用 `RustLib.init(... forceSameCodegenVersion: false)`；FRB 的 `_sanityCheckCodegenVersion`（`entrypoint.dart:108-128`）在该标志为 false 时只在 `assert` 中 print（release 静默）。
再生成命令（`flutter_rust_bridge.yaml`：`rust_input: crate::api`、`rust_root: rust/`、`dart_output: lib/src/rust`、`build_runner: false`）：在 `plugins/rhttp/rhttp` 下 `flutter_rust_bridge_codegen generate`（UPSTREAM.md 43 行写法为 `dart run flutter_rust_bridge_codegen generate`）。会漂移的三元组：Cargo 端 `=2.12.0` 钉死、codegen 二进制版本、`pubspec.lock` 解析到的 Dart 运行时；三者不同时升级就会再次出现 "Bypassing mismatched codegen version"。

### B.7 可维护性：测试与升级成本

- Rust 测试：`rust/tests/ech_config_test.rs`（17 行）、`rust/tests/ech_live_handshake.rs`（124 行，名字表明是在线握手测试）、`rust/examples/ech_reqwest_probe.rs`（84 行）。CI 不跑 `cargo test`（§C.4）。
- Dart 插件测试：`plugins/rhttp/rhttp/test/`（7 个文件，1,479 行：`interceptor_test` 598、`retry_interceptor_test` 254、`cancel_retry_test` 210、`mocks.dart` 212、`cancel_token_test` 100、`base_url_test` 56、`compatible_client_test` 49）。上游 `.forgejo/workflows/ci.yml` 用 Flutter 3.35.7/3.41.9 跑 `flutter test`；本仓库 CI 只跑根项目 `flutter test`，不进入插件目录。
- App 侧对 rhttp 的离线测试：`test/rhttp_client_factory_test.dart`、`test/restricted_compat_network_test.dart`（`rg -l package:rhttp test`）。
- 升级流程成本（UPSTREAM.md 37-44 + 本次观察）：① 对比上游 diff；② 重打 `Cargo.toml` + `client.rs` 两处；③ 同步 `android/build.gradle.kts` 的 AGP 9 适配（UPSTREAM.md 未提，但实际存在差异）；④ 运行 codegen 并同时对齐 §B.6 三元组；⑤ 重新验证 ECH 在线测试。没有自动化脚本承担 ①–④。

---

## C. 构建与 CI 流水线

### C.1 `.github/workflows/ci.yml`（68 行）与 `release.yml`（117 行）

**ci.yml**（push main / PR）：
- job `analyze-and-test`：`flutter pub get` → `flutter analyze` → `flutter test`（16-19）。
- job `android-release`：JDK 17 + Flutter 3.47.0；解码 `PIXIV_RELEASE_KEYSTORE_B64` 到 `~/.pixivfunc-release/pixivfunc-release.jks`（36-44）；`flutter build apk --release --flavor github -P…`（55-59）；断言 `build/app/outputs/flutter-apk/app-github-release.apk` 非空；`apksigner verify --verbose --print-certs`；若签名者含 `Android Debug` 则失败（60-68）。

**release.yml**（仅 `workflow_dispatch`，输入 `version`、`versionCode`）：
- 同样的 keystore 解码 + `flutter build apk --release --flavor github`，额外传 `-PPIXIV_UPDATE_PUBLIC_KEY_DER_B64`（55-60）。
- 读取签名者 SHA-256 → `FINGERPRINT`（62-78）。
- `cp $APK pixiv-func-v${version}-github.apk`；`python3 tool/update_release.py generate --apk … --version … --version-code … --asset-url https://github.com/Lopution/Pixiv-func/releases/download/v${version}/pixiv-func-v${version}-github.apk --signing-cert-sha256 $FINGERPRINT --key-file … --out-dir ./assets`（80-96）。
- `gh release create v${version} --draft` 上传 **单个 APK** + `update-manifest.json` + `update-manifest.sig`（98-109）。
- 泄漏扫描 `grep -rIl "PRIVATE KEY|BEGIN RSA|BEGIN EC PRIVATE"`（111-117）。

两个工作流都**没有**：`--split-per-abi`、`--obfuscate`、`--split-debug-info`、`--analyze-size`、`--target-platform`；没有安装 Rust target（`rustup target add aarch64-linux-android …`）的步骤——cargokit 的 `rustup.dart` 在构建时自行安装缺失 target（`cargokit/build_tool/lib/src/rustup.dart`），这依赖 runner 上可用的 rustup。

### C.2 更新清单格式与版本比较（与 per-ABI 拆分直接相关）

`tool/update_release.py:153-167` 生成的清单（`json.dumps(separators=(",",":"))`，无换行，签名的就是这些字节）：

```json
{"schema":1,"repository":"Lopution/Pixiv-func","tag":"v<version>","channel":"stable|beta",
 "version":"<semver>","versionCode":<int>,
 "asset":{"url":"<https://github.com/Lopution/Pixiv-func/releases/download/…/*.apk>",
          "size":<bytes>,"sha256":"<hex64>",
          "packageName":"io.github.lopution.pixivfunc",
          "signingCertificateSha256":"<hex64>"}}
```

- **单资产结构**：`asset` 是一个对象，不是数组；没有 ABI 字段。`update_release.py:139-149` 要求 URL host 为 `github.com`、path 以 `/Lopution/Pixiv-func/releases/download/` 开头、以 `.apk` 结尾。
- Dart 端解析 `lib/core/updater/update_manifest.dart`：必需键 `schema/…/versionCode/asset`（136-142）、`schema != 1` 拒绝（144-145）、`versionCode` 正整数（163）、`asset` 必需键 `url/size/sha256`（192-194）、`isStrictUpdateManifestAssetUrl(url)`（199；定义在 `lib/core/download/download_request.dart:182-186`：host `github.com` + 前缀 `/Lopution/Pixiv-func/releases/download/`）、`size ≤ updateAssetMaxBytes = 200 MiB`（`update_manifest.dart:10`, 203）、`packageName == io.github.lopution.pixivfunc`（208）。
- 版本判定 `lib/core/updater/update_service.dart:391-404`：先 semver 比较 `manifest.version.compareTo(currentVersion)`；`versionResult < 0 || (versionResult == 0 && manifest.versionCode <= info.versionCode)` → `up_to_date`。`info.versionCode` 来自 Kotlin `platformInfo` 的 `PackageInfo.longVersionCode`（github `DistributionUpdaterChannel.kt:101-115`）。
- 与 split 的交互：Flutter split 模式下 `output.versionCodeOverride = abiVersionCode * 1000 + versionCode`，`ABI_VERSION = {armeabi-v7a:1, arm64-v8a:2, x86_64:4}`（`FlutterPlugin.kt:657-676`、`FlutterPluginConstants.kt:39-46`），可用 `-Pforce-version-code-ignoring-abi=true` 关闭；APK 文件名变为 `app-<flavor>-<abi>-release.apk`（`/opt/flutter-3.47.0/packages/flutter_tools/lib/src/android/gradle.dart:147-158`）。因此一旦拆分：① 已安装设备的 `versionCode` 变成 `2001`（arm64）之类，而清单里的 `versionCode` 是人工输入的整数；② 清单只能指向一个 `url/size/sha256`；③ `release.yml:88-96` 的 `cp`/`--asset-url` 只处理一个文件。这些是**必须同步修改的契约**，不是可选项。

### C.3 flavor 差异与 F-Droid 约束

| 差异点 | github | fdroid |
|---|---|---|
| `UPDATE_SELF_UPDATER_ENABLED` | true | false |
| `UPDATE_PUBLIC_KEY_DER_B64` | 来自 Gradle 属性 | `""` |
| `RELEASE_SIGNED_OFFICIALLY` | 有钥匙时 true | false |
| signingConfig | 项目 keystore（或显式 debug） | 未设置 → 产出 `app-fdroid-release-unsigned.apk`（`build/app/outputs/apk/fdroid/release/`） |
| Manifest | `REQUEST_INSTALL_PACKAGES` | 无 |
| Kotlin | 233 行 updater | 74 行占位 |
| Dart | 通过 `getCapability.flavor` 区分（`update_platform.dart:42-46`） | 同 |

仓库内**没有** F-Droid 元数据（无 `metadata/`、`fastlane/`、`fdroid*.yml`；`rg -il "f-droid|fdroid"` 只命中 Gradle/Kotlin/Dart/测试和 `tool/RELEASE.md`）。`tool/RELEASE.md:47-48`："`fdroid` flavor stays store-managed (D-5): no local release key; F-Droid's build server signs its own builds." 没有可复现构建（reproducible build）说明、没有 `--split-debug-info`/`--obfuscate` 相关记录。F-Droid 构建 Rust 插件需要的工具链声明（rustup/NDK 版本）在仓库中不存在。

### C.4 流水线当前未做、且影响体积或可复现性的事项

| 项 | 现状 | 证据 |
|---|---|---|
| `--split-per-abi` / ABI 过滤 | 未做；单 universal APK 含 3 ABI | `ci.yml:55`、`release.yml:55`、`build.gradle.kts` 无 splits/abiFilters；`output-metadata.json` `"filters": []` |
| `--obfuscate --split-debug-info` | 未做 | 工作流 `rg obfuscate|split-debug-info` 零命中 |
| `--analyze-size` / 体积产物归档 | 未做 | 零命中；本地仅主会话手动跑过一次 |
| APK 体积门禁 | 无 | 工作流只断言 `test -s "$APK"` |
| Kotlin JVM 测试 | 未跑 | 无 `gradlew`/`test*UnitTest` 调用 |
| Rust 测试/审计 | 未跑 | 无 `cargo test`/`cargo audit`/`cargo deny` |
| 依赖锁验证 | 未做 | `flutter pub get` 无 `--enforce-lockfile`；`Cargo.lock` 未用 `--locked` |
| cargokit 陈旧 ABI 清理 | 无机制 | §0.2/§B.5 |
| `pubspec.lock` 与 `Cargo.lock` 的 FRB 三元组校验 | 无 | §B.6 |
| Rust 工具链钉版 | 无 `rust-toolchain.toml` | §B.5 |

---

## D. 候选改动（按优先级；体积一律"已测量/待测量"）

| # | 候选 | 证据 | 预期效果 | 影响的契约 | 风险 | 目标 |
|---|---|---|---|---|---|---|
| 1 | 发布改为 `--split-per-abi`，并把 x86_64 从发布产物中移除（保留 arm64-v8a + armeabi-v7a 或仅 arm64） | §0.1：三 ABI native 85.9 MB / 88.6 MB；x86_64 单独 31.8 MB；`FlutterPlugin.kt:540-561` split 逻辑；`gradle.dart:147-158` 命名 | **已测量**：arm64 APK 的 native 部分为 29.0 MB（fdroid arm64 构建中 `lib/arm64-v8a/` 精确为 29,020,544 B），去掉 x86_64/v7a 后每个 split APK 不再携带另外两套 ≈56.9 MB | updater：`versionCode` 变成 `2000+n`（或加 `-Pforce-version-code-ignoring-abi=true`）；`update-manifest.json` 单 `asset` 结构、`release.yml:88-109` 单文件上传、`update_service.dart:402-404` 比较逻辑、`update_release.py` 参数；F-Droid 需按自己的方式构建 ABI；API 29 设备无 x86 需求 | 中：契约面广，需一次性联动；`force-version-code` 与多 APK 同 versionCode 在 GitHub 分发下可接受但需明确决策 | 简洁性 / 升级 |
| 2 | 统一 SQLite 栈：Android 只走 `sqflite`（`sqflite_android`，0 native 字节），移除 `sqflite_common_ffi` + `sqlite3` native asset | 主会话：`libsqlite3.so` 1.73 MB/ABI 来自 `sqlite3` 3.5.2；`lib/core/history/history_database.dart:28-30` Android/iOS 已返回 `sqflite.databaseFactory`，ffi 仅桌面/测试分支；`pubspec.yaml:31-32` 两者都是直接依赖 | **已测量**：每 ABI 减少 ~1.7 MB（arm64 1,732,360 B） | 测试/桌面路径需要替代（`sqflite_common_ffi` 常用于单元测试）；`history_repository.dart` 也引用 sqflite | 低-中：需确认 `test/` 中是否依赖 ffi 工厂 | 简洁性 / 可维护性 |
| 3 | rhttp reqwest feature 裁剪：至少去掉 `multipart`、`form`、`socks`、`cookies`、`query`、`charset`（app 零使用）；`brotli/zstd/deflate/gzip` 与 `tokio full` 单独评估 | §B.2 对照表；§B.3 `-i` 树：cookie_store/publicsuffix、mime_guess、serde_urlencoded、encoding_rs 仅由这些 feature 引入；brotli/zstd-sys 由压缩 feature 引入 | **待测量**（需重编 `librhttp.so` 对比 5.44 MB arm64 基线）；`icu_*` 不会因 cookies 移除而消失（url 仍需） | 与上游的 fork diff 增大（UPSTREAM.md 声称只改 ECH）；去掉压缩会改变 `Accept-Encoding`，Pixiv 传输量可能上升 | 中：每去一个 feature 需确认插件 Dart 层不再引用（`HttpBody.form/multipart` 仍存在于生成代码中，可能编译失败或运行时报错） | 简洁性 / 可迭代性 |
| 4 | `[profile.release] opt-level = "s"`（或 `"z"`）实验 | `Cargo.toml:56-61` 当前 `opt-level = 3`，其余体积选项已开 | **待测量** | 无契约影响 | 低-中：TLS/解压吞吐可能下降，需要实际网络场景验证 | 简洁性 |
| 5 | `flutter build … --obfuscate --split-debug-info=<dir>`，并归档符号 | `ci.yml`/`release.yml` 零命中；`libapp.so` arm64 9,962,376 B | **待测量**（作用于 `libapp.so` 的符号/名字段） | 崩溃栈需符号文件还原；F-Droid 可复现构建需同参数 | 低：需要保存 symbols 目录 | 简洁性 / 升级 |
| 6 | 删除未使用依赖的资产：`cupertino_icons`（`pubspec.yaml:15`） | `lib/` 零引用；APK 内 `CupertinoIcons.ttf` 257,628 B | **已测量**：-257,628 B（约占 assets 的 40%） | 无 | 极低 | 简洁性 |
| 7 | androidx 工件整理：`work-runtime-ktx` → `work-runtime`（纯声明）；记录 `androidx.core:core:1.15.0` 仅为 `FileProvider`；评估是否需要 1.15.0 以上 | §A.6：ktx AAR 6,103 B / classes.jar 199 B（空壳）；`FileProvider` 两处引用 | **已测量**：体积 0 变化；只是消除误导性依赖 | 无 | 极低 | 可维护性 |
| 8 | FRB 版本三元组对齐：`Cargo.toml =2.12.0`、codegen 二进制、`pubspec.lock` 2.13.0 三者取齐，并在 `rhttp.dart:19` 恢复 `forceSameCodegenVersion` 默认或加 CI 校验 | §B.6 | 无体积影响 | 需要重跑 `flutter_rust_bridge_codegen generate` 并提交 `frb_generated.*` | 低：升级到 2.13 需同时升 Cargo 依赖并重编 | 升级 / 可迭代性 |
| 9 | CI 体积门禁：在 `android-release` job 产出后 `unzip -l` 统计 `lib/<abi>` 与总大小，超阈值失败；可选归档 `--analyze-size` JSON | §C.4：现在只 `test -s` | 无直接减量，防回归 | 阈值需在 #1/#2 落地后重定 | 低 | 可维护性 |
| 10 | 为 LLM 维护者撰写原生/Rust 契约文档：10 个 channel 的方法/载荷/错误码/线程模型、`widget_snapshot` 文件契约、cargokit ABI/陈旧产物行为、FRB 再生成步骤、updater 清单字段 | §A.2/§A.3 三种错误风格并存；`login_webview_intercept` 单侧；`.trellis/spec/` 仅 `frontend/state-management.md:1017,1140` 两节涉及 widget/updater 契约，无 Android/Rust 专章 | 无体积影响 | 无 | 极低 | 可维护性 |
| 11 | cargokit 输出目录清理：构建前删除 `build/rhttp/jniLibs/<buildType>/` 中非目标 ABI，或在 `plugin.gradle` 任务里 `delete(outputDir)` | §0.2：实测 10,353,372 B 陈旧 `librhttp.so` 混入 arm64 构建 | **已测量**：非 clean 树上避免 ~10.4 MB 污染；CI fresh checkout 无影响 | 修改 vendored cargokit（与上游 diff 增大） | 低 | 可维护性 / 升级 |
| 12 | 清理 minSdk 29 下的死分支与模板残留：12 处 `SDK_INT` 判断、`drawable-v21` 重复、debug/profile 冗余 `INTERNET`、`http://` 深链 filter | §A.3 / §A.4 / §A.5 | 体积影响可忽略（**已测量**：res 总计 10.9 KB） | `http` 深链移除会改变外部链接接管范围 | 低 | 简洁性 / 可维护性 |
| 13 | 抽出两份 `DistributionUpdaterChannel.kt` 的公共 `platformInfo/packageInfo/signerSha256` 到 main source set | §A.1：逐字重复三函数 | 无体积影响 | 无 | 低 | 可维护性 |
| 14 | 把 Kotlin JVM 测试（`android/app/src/test`，148 行）与 `plugins/rhttp/rhttp` 的 `flutter test`/`cargo test --offline` 纳入 CI | §A.1 / §B.7 / §C.4 | 无体积影响 | CI 时长增加 | 低 | 可维护性 / 可迭代性 |
| 15 | 评估 `rustls-platform-verifier` 是否保留：app 固定 webpki 根，Verifier 仅初始化不使用 | §B.2 最后两行；`init.rs:70`；`build.gradle.kts:120-151` 为它做 AAR 解包 | **待测量**（涉及 Rust crate + Android AAR classes） | 增大 fork diff；去掉后 `RootCertSource.platform` 不可用 | 中 | 简洁性 |
| 16 | Rust 工具链与依赖锁固化：`rust-toolchain.toml`、CI `cargo --locked`、`flutter pub get --enforce-lockfile`、可选 `cargo audit` | §B.5 / §C.4 | 无体积影响；提升可复现性（对 F-Droid 有价值） | F-Droid 构建脚本需声明同一工具链 | 低 | 升级 / 可维护性 |
| 17 | `path_provider_android 2.3.1` → `jni`/`jni_flutter` 引入 `libdartjni.so`（arm64 131,248 B）；仅记录，不建议单独处理 | `flutter pub deps` 行 100：`path_provider_android [flutter jni jni_flutter …]` | **已测量**：每 ABI 0.08–0.13 MB | 替换 path_provider 成本高 | — | （信息） |

---

## Related Specs

- `.trellis/spec/frontend/state-management.md:1017-1139`「Android Home Widget Snapshot and Background Contract」——widget 快照文件与 `WidgetUpdateCoordinator` 签名。
- `.trellis/spec/frontend/state-management.md:1140-`「Signed Updater and Distribution Flavor Contract」——updater/flavor 契约。
- `.trellis/spec/frontend/quality-guidelines.md:27`——Android SDK 路径约束（勿让 Gradle 解析到 `/usr/lib/android-sdk`）。
- `tool/RELEASE.md`——签名与更新清单发布流程。
- `plugins/rhttp/UPSTREAM.md`——fork 策略与同步指引。
- 同目录 `research/refactor-inventory.md`——全项目结构盘点（本文件是其 Android/Rust/CI 子集的深挖）。

## Caveats / Not Found

- 未运行任何新构建；所有体积数字来自现有 APK 的 `unzip -l` 与文件系统 mtime。`opt-level`、feature 裁剪、obfuscate 的体积收益都标注为待测量。
- `login_webview_intercept` 的原生端 `LoginWebViewPlatformView.kt` 在 HEAD 存在、在工作树被（未提交地）删除；`lib/features/login/login_intercept_controller.dart` 仍保留 Dart 端。这属于进行中的未提交工作，本文不判断其去向。
- 本次审计的 `android/` 与 `lib/core/platform/android_platform_interfaces.dart` 有未提交修改（+181/−460），所有行号以工作树为准。
- `pubspec.lock` 中 `flutter_rust_bridge 2.13.0` 与生成代码 2.12.0 的行为差异未做功能验证，只确认了 sanity check 被 `forceSameCodegenVersion: false` 绕过。
- 压缩 feature（brotli/zstd/gzip/deflate）对 Pixiv 实际响应体传输量的影响未测。
- F-Droid 侧的构建脚本/元数据不在本仓库，无法审计其对 Rust 工具链与 ABI 的要求。
- 插件 `android/build.gradle.kts` 中 AGP 8.11.2 classpath 与 app 的 AGP 9.1.0 并存，本次只记录版本差异，未验证是否产生解析冲突（现有构建成功）。
- `hs_err_pid*.log` 等文件被 git 忽略（`!!`），来源忽略规则未逐一确认是全局还是仓库级。
