# APK 体积基线与归因

日期：2026-09-07　测量环境：Flutter 3.47.0 / Dart 3.13.0 / cargo 1.98.0 / AGP 9.1.0 / Gradle 9.3.1

这份记录取代 09-02 原计划"不做体积基线"的决定。用户在 2026-09-07 明确提出"现在 80 多 MB
偏大"，体积成为本 task 的可验收目标之一，因此必须先有实测构成，再谈裁剪。

## 1. 现状：通用 fat APK 88.6 MB

`build/app/outputs/flutter-apk/app-github-release.apk`（2026-09-07 16:11 构建，release，
未 `--split-per-abi`，未 `--obfuscate`/`--split-debug-info`）：

| 类别 | 大小 | 说明 |
|---|---|---|
| `lib/x86_64/*` | 31.81 MB | 模拟器 ABI，真机基本不存在 |
| `lib/arm64-v8a/*` | 29.02 MB | 现代真机唯一需要的 ABI |
| `lib/armeabi-v7a/*` | 25.06 MB | 老 32 位设备 |
| `classes.dex` | 1.56 MB | Kotlin + androidx（R8 已默认启用，`build/app/outputs/mapping/*/mapping.txt` 存在） |
| `assets/flutter_assets/**` | ~0.62 MB | `stamps/` 0.35 MB(40 文件)、`emojis/` 0.03 MB(38 文件)、`NOTICES.Z` 0.12 MB、`packages/cupertino_icons` 字体 0.26 MB(压缩后 0.12 MB，来自未使用的依赖)、字体/shader 极小 |
| `res/*` + `resources.arsc` | 0.22 MB | |
| 其它（META-INF、kotlin builtins、manifest） | <0.1 MB | |

**结论：88.6 MB 中 85.9 MB 是三套 ABI 的 native 库，Dart/Kotlin/资源合计不到 3 MB。**
所有 `.so` 均以 stored（不压缩）方式打包，这是 Android 对齐要求，不是可优化项。

### 单 ABI（arm64-v8a）构成

| 文件 | 大小 | 来源 | 可动性 |
|---|---|---|---|
| `libflutter.so` | 11.75 MB | Flutter 引擎 | 不可动（随 Flutter 版本走） |
| `libapp.so` | 9.96 MB | Dart AOT 快照 | 小幅可动（见 §3） |
| `librhttp.so` | 5.44 MB | vendored Rust 插件（reqwest + rustls + aws-lc-rs + tokio + 压缩编解码 + quinn） | 可动（feature/profile 裁剪，见 §4） |
| `libsqlite3.so` | 1.73 MB | `sqlite3` Dart 包的 native asset，由直接依赖 `sqflite_common_ffi` 引入 | 可整体移除（见 §5） |
| `libdartjni.so` | 0.13 MB | `path_provider_android` → `jni` | 不动 |
| `libdatastore_shared_counter.so` | 0.01 MB | androidx.datastore（shared_preferences_android） | 不动 |

## 2. 实测：单 ABI 构建

命令：`flutter build apk --release --flavor fdroid --target-platform android-arm64 --analyze-size --code-size-directory=/tmp/pixiv-size`
（Rust 有缓存，Gradle 93.7 s）

结果 `app-fdroid-release.apk` = **42.21 MB**，而不是预期的 ~31.5 MB。拆包发现：

- `libapp.so`、`libflutter.so`、`libsqlite3.so` 只有 arm64-v8a 一份（`--target-platform` 对
  Flutter 自身产物和 Dart native asset 生效）；
- `librhttp.so`（cargokit）与 `libdartjni.so`（jni）仍然三 ABI 全出：`lib/x86_64` 6.82 MB、
  `lib/armeabi-v7a` 3.74 MB。

**结论：`--target-platform` 不是单 ABI 打包的正确手段。**真正的 per-ABI 产物必须由
`flutter build apk --split-per-abi`（为每个 split 设置 abiFilters）或 Gradle
`splits.abi` / `ndk.abiFilters` 决定，否则 cargokit 会继续为全部 ABI 编译并打包。

按实测数据推算：一个只含 arm64-v8a 的 APK ≈ 29.02 + 2.44 ≈ **31.5 MB**；armeabi-v7a ≈ 27.5 MB；
x86_64 ≈ 34.3 MB。

## 3. Dart AOT 快照（libapp.so 9.96 MB）归因

`--analyze-size` 对 arm64 快照的符号归因（accounted 9 MB）：

| 归属 | 大小 |
|---|---|
| `package:flutter` | 4 MB |
| `package:pixiv_func`（应用代码） | 1 MB |
| `dart:mixin_deduplication` / `dart:core` / `dart:ui` / `dart:typed_data` / `dart:io` / `dart:async` / `dart:collection` / `dart:convert` | ~1.6 MB 合计 |
| `package:flutter_localizations` | 301 KB |
| `package:riverpod` | 121 KB |
| `package:webview_flutter_android` | 120 KB |
| `package:intl` | 108 KB |
| `package:rhttp` | 78 KB |
| `package:material_color_utilities` | 71 KB |
| `package:flutter_rust_bridge` | 51 KB |
| `package:easy_refresh` | 50 KB |
| `package:image` | 45 KB |
| `package:source_span` | 43 KB |

要点：

- 快照由 Flutter 框架本身与 Dart 运行时库主导，应用代码只占 1 MB；`image`/`archive` 这类
  重量级纯 Dart 包在 tree-shaking 后只剩几十 KB，**不是体积问题**。
- 剩余可动项只有：`--obfuscate --split-debug-info=<dir>`（去掉符号名与调试信息；效果需实测，
  通常是快照的个位数百分比到 10% 级别）。`--tree-shake-icons` 已默认生效
  （MaterialIcons 1.6 MB → 13 KB，`icon.ttf` 5.9 KB → 2.4 KB）。
- `flutter_localizations` 301 KB 是全部 locale 的 Material/Cupertino 字符串，无法按 locale 裁剪。

## 4. librhttp.so（5.44 MB arm64）

`plugins/rhttp/rhttp/rust/Cargo.toml` 当前 reqwest features：`charset, cookies, form, http2,
query, rustls, stream, multipart, socks, brotli, deflate, gzip, zstd`；`tokio = full`；
`rustls`（aws-lc-rs provider，ECH 必需）；`webpki-root-certs`；`rustls-platform-verifier`。
Cargo.lock 中还包含 `quinn`/`quinn-proto`/`quinn-udp`（HTTP/3，`UPSTREAM.md` 注明"上游
编入、本项目未使用"）。`[profile.release]` 已是 `opt-level=3, lto, strip, codegen-units=1,
panic=abort`。

可动方向（效果全部标注"待测量"，需要逐项构建对比 `.so` 大小）：

1. 去掉 Dart 侧未消费的 reqwest features（socks / cookies / multipart / form 等，以
   `research/native-and-build-audit.md` 的消费者表为准）；
2. 收窄 `tokio` features（`full` → 实际需要的 rt-multi-thread/net/time/sync/io-util 等）；
3. 移除 HTTP/3（quinn）编入路径；
4. `opt-level = "s"` 或 `"z"`（TLS 热路径在 aws-lc 的 C 代码里，受影响较小，但需实测吞吐）。
5. aws-lc-rs 不能换成 ring：ECH 依赖 HPKE，仅 aws-lc-rs provider 提供。

任何一条都必须按 `plugins/rhttp/UPSTREAM.md` 的 fork 策略处理：改 `Cargo.toml` 是允许的
"源配置"改动，但要记录与上游的差异；改动后重跑 `cargo test` 与插件 Dart 测试。

## 5. libsqlite3.so（1.73 MB/ABI）：仅因依赖声明位置而进入 APK

- `pubspec.yaml` 同时直接依赖 `sqflite`（Android 走 `sqflite_android`，使用系统 SQLite，
  0 native 字节）与 `sqflite_common_ffi`（→ `sqlite3` 3.5.2，通过 native asset/`hooks` 自行
  编译并打包 `libsqlite3.so`）。
- `lib/core/history/history_database.dart:27-31`：
  `if (Platform.isAndroid || Platform.isIOS) return sqflite.databaseFactory;` 之后才
  `sqfliteFfiInit(); return databaseFactoryFfi;`。**Android 运行时走的是平台 SQLite，FFI 分支
  只服务桌面与测试**；而仓库只有 `android/` 平台目录，桌面分支在生产代码中是死路径。
- 但 Dart native asset 的打包只看依赖图，不看运行时分支：只要 `sqflite_common_ffi` 在
  `dependencies` 里，`libsqlite3.so` 就会被编译并进入每个 ABI。
- 用户决策 D-4（2026-09-07）：**保留桌面分支与 `sqflite_common_ffi` 主依赖**。`sqlite3` 3.5.2 的
  `hook/build.dart` 对每个 `targetOS` 都编译（Android 分支只是额外链接 libm），没有按 OS 跳过的
  user-define。因此只能在 Android 打包侧排除：`packaging { jniLibs { excludes += "**/libsqlite3.so" } }`，
  并用测试保证 Android 上的工厂是 `sqflite.databaseFactory`。排除是否对 Flutter native asset 生效
  **待构建验证**；不可行则接受每 ABI 1.73 MB，收敛目标放宽到 ≤ 30 MB。
- 全仓库 DB 访问点只有 `history_repository.dart` 一处，import `sqflite_common_ffi` 的生产文件只有
  `history_database.dart`（`dart-architecture-audit.md` D 节）。

## 6. 可量化目标（本 task 新增，供 prd 引用）

| 指标 | 基线 | 目标 | 手段 |
|---|---|---|---|
| 用户下载的 release 包（arm64-v8a） | 88.6 MB（fat） | ≤ 32 MB 先落地（仅 per-ABI），再向 ≤ 28 MB 收敛 | `--split-per-abi`；随后 §4/§5 裁剪与 `--obfuscate` |
| release 产物是否含 x86_64 | 含 | 不含（模拟器用 debug 包） | split 输出中不发布 x86_64 |
| `libsqlite3.so` | 每 ABI 1.73 MB | 0（B1 已验证：split APK 内无 `libsqlite3.so`）| Android 侧 `packaging.jniLibs.excludes`；桌面分支与主依赖按 D-4 保留 |
| 未使用依赖带入的 assets | `cupertino_icons` 0.26 MB | 0 | 删除依赖 |
| CI 体积门禁 | 无 | 每次 release 构建输出 per-ABI 大小并对比上次（B3 输出、B7 门禁已落地） | `tool/apk_size_report.py` + `android-size` job |

"≤ 28 MB"是 §3–§5 全部落地后的推算值，不是承诺；实现阶段每完成一项就重新测量一次并
更新本文件。

### B0 干净 ABI 基线（child B，2026-09-07）

工作树 `/root/Pixiv-func-B`，`task/09-07-release-size-per-abi` @ `4c7742e`。
Flutter 3.47.2 / Dart 3.13.2 / cargo 1.98.0 / AGP 9.1.1 / Gradle 9.3.1。
先 `rm -rf build/rhttp/jniLibs`（目录本就不存在），再测。

命令与墙钟：

| 步 | 命令 | 开始 (UTC) | 墙钟 | Gradle |
|---|---|---|---|---|
| fat（三 ABI） | `flutter build apk --release --flavor fdroid` | 2026-09-07T15:50:06Z | 421.75 s | 383.6 s |
| split（仅拆 ABI，无 obfuscate） | `flutter build apk --release --flavor fdroid --split-per-abi --target-platform android-arm64,android-arm` | 2026-09-07T15:57:16Z | 109.87 s | 98.4 s |
| analyze-size（见下） | `flutter build apk --release --flavor fdroid --target-platform android-arm64 --analyze-size --code-size-directory=/tmp/pixiv-size-b0-fat` | 2026-09-07T15:59:12Z | 91.58 s | 77.5 s |

`implement.md` 写的 fat 命令带 `--analyze-size`。Flutter 3.47.2 拒绝在多 ABI
上做 code-size analysis（`Cannot perform code size analysis when building for
multiple ABIs`），所以 fat 本身不带该旗标。`--analyze-size` 目录在 fat + split
测完之后用单独的 `--target-platform android-arm64` 补齐，不作为单 ABI 基线。
该补齐命令把 `app-fdroid-release.apk` 覆写成带 §0.2 陈旧 ABI 泄漏的 41.8 MB
包；下表 fat 数字全部取自覆写前的 `88,181,654` B 文件。

产物路径：Flutter 3.47.2 的 split 命名是 `app-<abi>-<flavor>-release.apk`，
不是 design 初稿的 `app-<flavor>-<abi>-release.apk`。

| 产物 | 文件字节 | vs child A 收尾 `88,181,654` |
|---|---|---|
| `app-fdroid-release.apk`（fat，三 ABI） | 88,181,654 | 0 |
| `app-arm64-v8a-fdroid-release.apk` | 31,187,258 | —（≤ 32,000,000） |
| `app-armeabi-v7a-fdroid-release.apk` | 27,203,718 | — |

Python `zipfile` 按 `compress_size` 汇总（native `.so` 均为 stored，compress = raw）：

| 桶 | fat | split arm64-v8a | split armeabi-v7a |
|---|---|---|---|
| zip 条目合计 | 87,955,713 | 31,062,844 | 27,116,041 |
| `lib/x86_64` | 31,846,664 | — | — |
| `lib/arm64-v8a` | 28,993,008 | 28,993,008 | — |
| `lib/armeabi-v7a` | 25,046,204 | — | 25,046,204 |
| `classes.dex` | 1,309,768 | 1,309,768 | 1,309,768 |
| `assets` | 513,725 | 513,720 | 513,720 |

每个 split APK 的 `lib/` 只有一个 ABI 目录。fat 与两份 split 的对应 `lib/<abi>`
桶逐字节相同。

split APK 内 `.so` raw 字节（stored = raw）：

| 文件 | arm64-v8a | armeabi-v7a |
|---|---|---|
| `librhttp.so` | 5,412,048 | 3,636,460 |
| `libapp.so` | 9,962,376 | 10,994,248 |
| `libsqlite3.so` | 1,732,360 | 1,713,736 |
| `libflutter.so` | 11,747,528 | 8,615,900 |
| `libdartjni.so` | 131,248 | 81,444 |

fat 另含 x86_64：`librhttp.so` 6,738,312 / `libapp.so` 10,224,520 /
`libsqlite3.so` 1,709,544。`librhttp.so` 与 `libapp.so` 三 ABI 与 child A
收尾逐字节一致。

`unzip -l … \| grep cupertino_icons`：fat 与两份 split 均为空。

`--analyze-size` 目录：`/tmp/pixiv-size-b0-fat/`
（`snapshot.arm64-v8a.json` 22,859,365 B，`trace.arm64-v8a.json` 2,281,577 B）。
该次单 ABI 分析打印的 APK 仍带 `lib/armeabi-v7a` 4 MB + `lib/x86_64` 7 MB
（cargokit / 插件 jniLibs 陈旧 ABI，§0.2），再次确认 `--target-platform`
不能当发布手段。

### B1 `libsqlite3.so` 排除（child B，2026-09-07）

`android/app/build.gradle.kts` 在 `buildTypes` 后加入
`packaging { jniLibs { excludes += "**/libsqlite3.so" } }`。未删除
`sqflite_common_ffi`。`historyDatabaseFactory({required bool useMobileSqflite})`
抽出为 `@visibleForTesting` 纯函数；生产 `_platformDatabaseFactory` 仍是
`Platform.isAndroid || Platform.isIOS` → `sqflite.databaseFactory`。
`test/history_database_factory_test.dart`：mobile 分支 `identical` 于
`sqflite.databaseFactory` / `databaseFactorySqflitePlugin`，且不是
`databaseFactoryFfi`（Linux VM 需先注册 plugin factory，否则 getter 抛
uninitialized）。`test/history_persistence_test.dart` 未改，与新测试合计
9 passed。

验证（`unzip -l … | grep libsqlite3.so` 均为空）：

| APK | 文件字节 | `libsqlite3.so` |
|---|---|---|
| `app-arm64-v8a-fdroid-release.apk` | 29,443,769 | 无 |
| `app-armeabi-v7a-fdroid-release.apk` | 25,489,119 | 无 |
| `app-fdroid-debug.apk` | 202,265,375 | 无 |

`assets/flutter_assets/NativeAssetsManifest.json` 仍列出
`package:sqlite3/src/ffi/libsqlite3.g.dart` → `["absolute","libsqlite3.so"]`
（release 224 B，debug 313 B），但 `lib/<abi>/` 没有对应 `.so`。按
`design.md`「清单仍列、库不在包内」视为排除成功，运行时不加载。**可行。**

墙钟：split release 111.77 s（Gradle 100.0 s，2026-09-07T16:03:12Z）；
debug 353.18 s（Gradle 320.2 s，2026-09-07T16:05:11Z）。

前后桶（fdroid split，无 obfuscate；compress = raw）：

| | B0 arm64 | B1 arm64 | Δ | B0 armeabi-v7a | B1 armeabi-v7a | Δ |
|---|---|---|---|---|---|---|
| APK 文件 | 31,187,258 | 29,443,769 | −1,743,489 | 27,203,718 | 25,489,119 | −1,714,599 |
| `lib/<abi>` | 28,993,008 | 27,260,648 | −1,732,360 | 25,046,204 | 23,332,468 | −1,713,736 |
| `classes.dex` | 1,309,768 | 1,309,768 | 0 | 1,309,768 | 1,309,768 | 0 |
| `assets` | 513,720 | 513,720 | 0 | 513,720 | 513,720 | 0 |

`lib/<abi>` 桶减少值与 B0 的 `libsqlite3.so` raw 字节逐字节相等
（arm64 1,732,360 / armeabi-v7a 1,713,736）。`librhttp.so` / `libapp.so`
未变。真机历史读写待用户。

### B2 cargokit 清理（child B，2026-09-07）

`CargoKitBuildTask.build()` 在 `execOperations.exec` 之前
`project.delete(outputDir)`（`jniLibs/<buildType>`）。
`plugins/rhttp/UPSTREAM.md` D-5 增加对应一条。

**清理前（B0 的 analyze-size 补齐，无 delete）**：先有三 ABI `jniLibs`，再
`--target-platform android-arm64`（无 `--split-per-abi`）。Flutter 打印
`app-fdroid-release.apk (41.8MB)`，分析表含 `lib/armeabi-v7a` 4 MB +
`lib/x86_64` 7 MB。泄漏的是陈旧 `librhttp.so`（加上插件 AAR 的
`libdartjni.so` / `libdatastore_shared_counter.so`）。§0.2 当时测得
`librhttp.so` 跨 ABI 泄漏 10,353,372 B。

**复现污染（本步 fat，delete 已在仓）**：
`flutter build apk --release --flavor fdroid`（2026-09-07T16:11:34Z，
墙钟 94.04 s，Gradle 83.9 s）后磁盘：

```
build/rhttp/jniLibs/release/arm64-v8a/librhttp.so     5,412,064
build/rhttp/jniLibs/release/armeabi-v7a/librhttp.so   3,636,464
build/rhttp/jniLibs/release/x86_64/librhttp.so        6,738,328
```

非目标 ABI 合计 10,374,792 B（若不被 delete，会原样打进下一包）。

**清理后**：
`flutter build apk --release --flavor fdroid --target-platform android-arm64`
（2026-09-07T16:13:08Z，墙钟 130.01 s，Gradle 116.4 s）。磁盘
`jniLibs/release/` 只剩 `arm64-v8a/librhttp.so`。

```
rhttp ['lib/arm64-v8a/librhttp.so']
ASSERTION_OK
```

`app-fdroid-release.apk` 文件 29,690,737 B。相对 B0 同命令的 41.8 MB，
少掉的主体是那两份陈旧 `librhttp.so`。

`libdartjni.so`（及 `libdatastore_shared_counter.so`）在非 split 的
`--target-platform` 下仍三 ABI 全出：

```
libdartjni ['lib/arm64-v8a/libdartjni.so', 'lib/armeabi-v7a/libdartjni.so', 'lib/x86_64/libdartjni.so']
```

与 `design.md` 一致：B2 只主张 `librhttp.so`；B3 的 `--split-per-abi`
会设 `abiFilters`，jni / datastore 随过滤器走。

### B3 per-ABI 流水线

工作树 `/root/Pixiv-func-B`，`task/09-07-release-size-per-abi` 在 B0–B2 之后。
Flutter 3.47.2 / `tool/apk_size_report.py` 口径（文件字节 + `zipfile` `compress_size`）。
构建旗标：`--split-per-abi --target-platform android-arm64,android-arm --obfuscate --split-debug-info=build/symbols/<f>`。
本机 github 加 `-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`（debug 签名，预期）。

| 步 | 命令 | 开始 (UTC) | 墙钟 |
|---|---|---|---|
| fdroid split + obfuscate | `flutter build apk --release --flavor fdroid --split-per-abi --target-platform android-arm64,android-arm --obfuscate --split-debug-info=build/symbols/fdroid` | 2026-09-07T16:39:25Z | 17.65 s（增量；B1/B2 已暖机） |
| github split + obfuscate | 同上，`--flavor github` + debug-signing 旗标 | 2026-09-07T16:40:10Z | 100.49 s |

产物恰好两份 / flavor：`app-arm64-v8a-<f>-release.apk`、`app-armeabi-v7a-<f>-release.apk`。
无 `x86_64`、无 universal。每个 split 的 `lib/` 只有一个 ABI 目录。
符号：`build/symbols/fdroid/` 与 `build/symbols/github/`（`app.android-arm.symbols` 4,217,792 B，`app.android-arm64.symbols` 4,901,824 B）。

| 产物 | 文件字节 | `lib/<abi>` | `classes.dex` | `assets` |
|---|---|---|---|---|
| fdroid arm64-v8a | 27,805,369 | 25,622,248 | 1,309,820 | 513,721 |
| fdroid armeabi-v7a | 23,523,039 | 21,366,388 | 1,309,820 | 513,721 |
| github arm64-v8a（debug 签） | 27,829,969 | 25,622,248 | 1,313,584 | 513,707 |
| github armeabi-v7a（debug 签） | 23,547,639 | 21,366,388 | 1,313,584 | 513,707 |

arm64 文件字节 ≤ 32,000,000（硬顶）。两 flavor 的 `lib/<abi>` 逐字节一致；github 多出的 ~24.6 KB 在 dex/资源（与 child A 收尾的 flavor 差同量级）。

`libapp.so` vs B1（无 obfuscate；B0/B1 raw 9,962,376 / 10,994,248）：

| ABI | B1 `libapp.so` | B3 `libapp.so` | Δ |
|---|---|---|---|
| arm64-v8a | 9,962,376 | 8,323,976 | −1,638,400 |
| armeabi-v7a | 10,994,248 | 9,028,168 | −1,966,080 |

`librhttp.so` 未变：5,412,048 / 3,636,460。fdroid arm64 APK 相对 B1 的 29,443,769 为 −1,638,400，与 `libapp.so` 下降一致。

`tool/apk_size_report.py --self-test` 通过（单 ABI zip 通过；双 `lib/` 与 env 更严阈值失败并打 `::error::`）。

### B4 schema 2 生成器

`python3 tool/update_release.py self-test` 通过：schema 2、两笔 assets、顶层 `versionCode` 为基数、sign→verify→tamper、拒绝覆盖。

### B5 rhttp feature 裁剪

工作树 `/root/Pixiv-func-B`，`task/09-07-release-size-per-abi`。对照 B0/B3 `librhttp.so` 5,412,048 / 3,636,460。
每步 fdroid split，无 `--obfuscate`（与 B1 同旗标，避免和 B3 符号段纠缠）。
`jniLibs/release/*/librhttp.so` 在每步构建前删除，构建后 mtime 新于删除时刻。

| 步 | 去掉 | `librhttp.so` arm64 | `librhttp.so` armeabi-v7a | fdroid arm64 APK | `lib/arm64-v8a` 桶 | 编译修补 | 双侧测试 | 真机 |
|---|---|---|---|---|---|---|---|---|
| B5a | reqwest `multipart` | 5,376,968 | 3,612,364 | 29,408,689 | 27,225,568 | 有：`http.rs:414-418` | 通过 | 待用户 |
| B5b | reqwest `form` | 5,375,112 | 3,611,044 | 29,406,833 | 27,223,712 | 有：`http.rs:413-417` | 通过 | 待用户 |
| B5c | reqwest `socks` | 5,330,032 | 3,583,788 | 29,361,753 | 27,178,632 | 无 | 通过 | 待用户 |
| B5d | reqwest `cookies` | 5,243,360 | 3,519,188 | 29,275,081 | 27,091,960 | 有：`client.rs:165-169` | 通过 | 待用户 |
| B5e | reqwest `query`（**已回退**，见下） | 5,237,280 | 3,514,492 | 29,269,001 | 27,085,880 | 有：`http.rs:375-379` | 通过 | — |
| B5f | reqwest `charset` | 5,052,080 | 3,349,084 | 29,083,801 | 26,900,680 | 无 | 通过 | 待用户 |
| B5g | tokio `full` → listed | 5,027,744 | 3,335,776 | 29,059,465 | 26,876,344 | 无 | 通过 | 待用户 |
| B5e 回退 | 恢复 reqwest `query` | 5,042,000 | 3,340,216 | 29,073,721 | 26,890,600 | 恢复 `request.query(&query)` | 通过 | 待用户 |

#### B5a drop reqwest `multipart`

`Cargo.toml` 去掉 `multipart`；`cargo update -w` 从 lock 去掉 `mime_guess` 2.0.5、`unicase` 2.9.0。
host `cargo build --release --locked` 先在 `http.rs:414-443` 的 `reqwest::multipart::{Form,Part}` / `request.multipart` 失败（E0433/E0599）。
`HttpBody::Multipart(_)` 改为 `return Err(RhttpError::RhttpUnknownError("multipart body is not supported"))`（`:414-418`）。未跑 FRB codegen，Dart API 未改。

构建：`flutter build apk --release --flavor fdroid --split-per-abi --target-platform android-arm64,android-arm`
开始 2026-09-07T16:54:23Z，墙钟 111 s（Gradle 97.8 s）。
`jniLibs` mtime 2026-09-08 00:55:57 +0800，APK mtime 00:56:14。

| | B0/B3 | B5a | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,412,048 | 5,376,968 | −35,080 |
| `librhttp.so` armeabi-v7a | 3,636,460 | 3,612,364 | −24,096 |
| fdroid arm64 APK（无 obfuscate；对照 B1 29,443,769） | 29,443,769 | 29,408,689 | −35,080 |
| `lib/arm64-v8a` 桶（对照 B1 27,260,648） | 27,260,648 | 27,225,568 | −35,080 |

APK 文件与 `lib/arm64-v8a` 下降值与 arm64 `.so` 下降逐字节相等。

测试：
- `(cd plugins/rhttp/rhttp && flutter test)`：32 passed
- `(cd plugins/rhttp/rhttp/rust && cargo test --locked)`：2 passed, 1 ignored (`ech_live_handshake`)
- `flutter test -j 4 test/rhttp_client_factory_test.dart test/restricted_compat_network_test.dart`：41 passed
- clippy：CI `plugin` job 不跑 clippy（只 `cargo fmt --check` + `cargo test --locked`），未跑

features 之后：`charset, cookies, form, http2, query, rustls, stream, socks, brotli, deflate, gzip, zstd`；`tokio` 仍 `full`。
真机登录/图片/下载待用户。

#### B5b drop reqwest `form`

`Cargo.toml` 去掉 `form`；`cargo update -w` 未改 lock（`serde_urlencoded` 仍由 `query` 拉入）。
host `cargo build --release --locked` 在 `http.rs:413` `request.form(&form)` 失败（E0599）。
`HttpBody::Form(_)` 改为 `return Err(RhttpError::RhttpUnknownError("form body is not supported"))`（`:413-417`）。未跑 FRB codegen。

构建：同上 fdroid split。开始 2026-09-07T17:00:20Z，墙钟 74 s（Gradle 65.3 s）。
`jniLibs` mtime 2026-09-08 01:01:20 +0800，APK mtime 01:01:34。

| | B5a | B5b | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,376,968 | 5,375,112 | −1,856 |
| `librhttp.so` armeabi-v7a | 3,612,364 | 3,611,044 | −1,320 |
| fdroid arm64 APK | 29,408,689 | 29,406,833 | −1,856 |
| `lib/arm64-v8a` 桶 | 27,225,568 | 27,223,712 | −1,856 |

测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。clippy 未跑（CI 无）。
features 之后：`charset, cookies, http2, query, rustls, stream, socks, brotli, deflate, gzip, zstd`。
真机待用户。

#### B5c drop reqwest `socks`

`Cargo.toml` 去掉 `socks`；`cargo update -w` 未改 lock（reqwest 0.13 的 `socks` 不再拉 `tokio-socks`）。
host `cargo build --release --locked` 直接通过：`client.rs:146-160` 的 `reqwest::Proxy::{http,https,all}` 对 http/https 代理不需要该 feature。未改 `http.rs` / `client.rs`。`socks://` 代理若传入，走现有 `Error creating proxy` → `RhttpUnknownError`。app 零 `ProxySettings`。

构建：同上 fdroid split。开始 2026-09-07T17:03:50Z，墙钟 40 s（Gradle 34.0 s）。
`jniLibs` mtime 2026-09-08 01:04:11 +0800。

| | B5b | B5c | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,375,112 | 5,330,032 | −45,080 |
| `librhttp.so` armeabi-v7a | 3,611,044 | 3,583,788 | −27,256 |
| fdroid arm64 APK | 29,406,833 | 29,361,753 | −45,080 |
| `lib/arm64-v8a` 桶 | 27,223,712 | 27,178,632 | −45,080 |

测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。clippy 未跑（CI 无）。
features 之后：`charset, cookies, http2, query, rustls, stream, brotli, deflate, gzip, zstd`。
真机待用户。

#### B5d drop reqwest `cookies`

`Cargo.toml` 去掉 `cookies`；`cargo update -w` 从 lock 去掉 `cookie` 0.18.2、`cookie_store` 0.22.1、`publicsuffix` 2.3.0、`psl-types`、`time`/`time-core`/`time-macros`、`serde_json`、`deranged`、`num-conv`、`powerfmt`、`document-features`、`litrs`、`zmij`。`icu_*` 仍在（`url` 需要）。
host `cargo build --release --locked` 在 `client.rs:166` `cookie_store(...)` 失败（E0599）。
`settings.cookie_settings.is_some()` 改为 `return Err(RhttpError::RhttpUnknownError("cookie store is not supported"))`（`:165-169`）。未跑 FRB codegen。

构建：同上 fdroid split。开始 2026-09-07T17:06:11Z，墙钟 76 s（Gradle 62.8 s）。
`jniLibs` mtime 2026-09-08 01:07:12 +0800。

| | B5c | B5d | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,330,032 | 5,243,360 | −86,672 |
| `librhttp.so` armeabi-v7a | 3,583,788 | 3,519,188 | −64,600 |
| fdroid arm64 APK | 29,361,753 | 29,275,081 | −86,672 |
| `lib/arm64-v8a` 桶 | 27,178,632 | 27,091,960 | −86,672 |

测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。clippy 未跑（CI 无）。
features 之后：`charset, http2, query, rustls, stream, brotli, deflate, gzip, zstd`。
真机待用户。

#### B5e drop reqwest `query`

`Cargo.toml` 去掉 `query`；`cargo update -w` 从 lock 去掉 `serde_urlencoded` 0.7.1、`ryu` 1.0.23。`form_urlencoded` 仍由 `url` 拉入。
host `cargo build --release --locked` 在 `http.rs:376` `request.query(&query)` 失败（E0599）。
`query.is_some()` 改为 `return Err(RhttpError::RhttpUnknownError("query parameters are not supported"))`（`:375-379`）。app 自行拼 URL。未跑 FRB codegen。

构建：同上 fdroid split。开始 2026-09-07T17:09:09Z，墙钟 68 s（Gradle 60.0 s）。
`jniLibs` mtime 2026-09-08 01:10:02 +0800。

| | B5d | B5e | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,243,360 | 5,237,280 | −6,080 |
| `librhttp.so` armeabi-v7a | 3,519,188 | 3,514,492 | −4,696 |
| fdroid arm64 APK | 29,275,081 | 29,269,001 | −6,080 |
| `lib/arm64-v8a` 桶 | 27,091,960 | 27,085,880 | −6,080 |

测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。clippy 未跑（CI 无）。
features 之后：`charset, http2, rustls, stream, brotli, deflate, gzip, zstd`。
真机待用户。

#### B5f drop reqwest `charset`

`Cargo.toml` 去掉 `charset`；`cargo update -w` 从 lock 去掉 `encoding_rs` 0.8.40、`mime` 0.3.17、`multiversion`/`multiversion-macros`/`multiversion_no_op`、`core_detect`、`target-features`。
host `cargo build --release --locked` 直接通过（源码无 charset API）。插件 `.text()` 仍编过，无 charset 时按 UTF-8；app 兼容层只取字节、Dart `http.Response.fromStream` 解码。未改 `http.rs` / `client.rs`。

构建：同上 fdroid split。开始 2026-09-07T17:11:52Z，墙钟 69 s（Gradle 60.4 s）。
`jniLibs` mtime 2026-09-08 01:12:49 +0800。

| | B5e | B5f | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,237,280 | 5,052,080 | −185,200 |
| `librhttp.so` armeabi-v7a | 3,514,492 | 3,349,084 | −165,408 |
| fdroid arm64 APK | 29,269,001 | 29,083,801 | −185,200 |
| `lib/arm64-v8a` 桶 | 27,085,880 | 26,900,680 | −185,200 |

测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。clippy 未跑（CI 无）。
features 之后：`http2, rustls, stream, brotli, deflate, gzip, zstd`。
真机待用户。

#### B5g narrow tokio features

`tokio = { version = "1.52", features = ["full"] }` 改为
`["rt-multi-thread", "net", "time", "sync", "io-util", "macros"]`。
host `cargo build --release --locked` 直接通过，未再补 feature（B5a 已去掉 `tokio::fs`；未要 `fs`/`process`/`signal`）。
`cargo update -w` 从 lock 去掉 `parking_lot`、`signal-hook-registry`、`errno`。未改 `http.rs` / `client.rs`。`[profile.release]` 未动。

构建：同上 fdroid split。开始 2026-09-07T17:14:41Z，墙钟 76 s（Gradle 67.3 s）。
`jniLibs` mtime 2026-09-08 01:15:45 +0800。

| | B5f | B5g | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,052,080 | 5,027,744 | −24,336 |
| `librhttp.so` armeabi-v7a | 3,349,084 | 3,335,776 | −13,308 |
| fdroid arm64 APK | 29,083,801 | 29,059,465 | −24,336 |
| `lib/arm64-v8a` 桶 | 26,900,680 | 26,876,344 | −24,336 |

测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。clippy 未跑（CI 无）。
reqwest features 仍是：`http2, rustls, stream, brotli, deflate, gzip, zstd`。
tokio features：`rt-multi-thread, net, time, sync, io-util, macros`。

B5a–B5g 合计（回退前）vs B0/B3 `librhttp.so`：arm64 5,412,048 → 5,027,744（−384,304）；armeabi-v7a 3,636,460 → 3,335,776（−300,684）。
真机待用户。

#### B5e 回退（`git revert c463431`）

主会话复核发现 B5e 的前提「app 自行拼 URL」不成立：`plugins/rhttp/rhttp/lib/src/client/io/io_request.dart:46-53` 把 URL 重建为 `scheme://host:port/path`，查询串一律经 `query: uri.queryParameters` 传给 Rust（空 map 也是 `Some`）。app 全部请求走 `RhttpCompatibleClient`（`lib/core/network/compat/rhttp_client_factory.dart:68`），B5e 之后 `http.rs:375` 对每个请求返回 `RhttpUnknownError("query parameters are not supported")`——app 41 个测试在 mock 层通过，没有覆盖真实 Rust 路径。按 B5 停止条件（行为回归 → revert 该提交）回退：`Cargo.toml` 恢复 `query`，`cargo update -w` 后 lock 恢复 `serde_urlencoded` 0.7.1、`ryu` 1.0.23（无其他变化），`http.rs` 恢复 `request.query(&query)`。`UPSTREAM.md` 改记为「保留」。

构建：fdroid split，无 `--obfuscate`，删 `build/rhttp/jniLibs/release/*/librhttp.so` 后重建。开始 2026-09-07T17:20:30Z，墙钟 70 s（Gradle 60.4 s）。`jniLibs` mtime 2026-09-08 01:21:24 +0800。

| | B5g | B5e 回退 | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,027,744 | 5,042,000 | +14,256 |
| `librhttp.so` armeabi-v7a | 3,335,776 | 3,340,216 | +4,440 |
| fdroid arm64 APK | 29,059,465 | 29,073,721 | +14,256 |
| `lib/arm64-v8a` 桶 | 26,876,344 | 26,890,600 | +14,256 |

（回加 `query` 后多出的字节大于 B5e 当时省下的 6,080：`serde_urlencoded` 回来时 `charset`/`cookies` 已不在，链接布局不同。）
测试：plugin flutter 32 passed；cargo test 2 passed + 1 ignored；app 41 passed。

**B5 最终**（B5a–d、B5f–g 生效，B5e 回退）vs B0/B3 `librhttp.so`：arm64 5,412,048 → 5,042,000（−370,048）；armeabi-v7a 3,636,460 → 3,340,216（−296,244）。
reqwest features 最终：`http2, query, rustls, stream, brotli, deflate, gzip, zstd`；tokio：`rt-multi-thread, net, time, sync, io-util, macros`。
真机待用户（登录 + 图片列表 + 大图下载，一次覆盖 B5 全部）。

### B6 `opt-level = "s"` 实验（已测量，未采用，待用户真机）

按 `design.md` 协议：对照为 B5 最终（B5e 回退后，opt-level 3）的 fdroid split（无 `--obfuscate`）；只改 `plugins/rhttp/rhttp/rust/Cargo.toml` `[profile.release]` `opt-level = 3` → `"s"`，其余 profile 键不动；删 `build/rhttp/jniLibs/release/*/librhttp.so` 后重建。开始 2026-09-07T17:24:16Z，墙钟 97 s。对照组随后用同旗标重建，字节与「B5e 回退」逐字节一致（29,073,721 / 5,042,000），说明测量可复现。

| | opt-level 3（B5 最终） | opt-level "s" | Δ |
|---|---|---|---|
| `librhttp.so` arm64 | 5,042,000 | 3,503,000 | −1,539,000（−30.5%） |
| `librhttp.so` armeabi-v7a | 3,340,216 | 2,455,272 | −884,944（−26.5%） |
| fdroid arm64 APK | 29,073,721 | 27,534,721 | −1,539,000 |
| fdroid armeabi-v7a APK | 25,192,875 | 24,307,931 | −884,944 |
| `lib/arm64-v8a` 桶 | 26,890,600 | 25,351,600 | −1,539,000 |

`.so` 下降条件满足，且幅度是 B5 全部裁剪的 4.2 倍。采用门的另一半——列表 + 大图下载墙钟无可感知回退——只能由用户真机判定（`cc` crate 会把 `OPT_LEVEL=s` 映射为 `-Os`，brotli/zstd 的 C 解压也会随之改编译等级；rustls 的密码学在 aws-lc-rs 汇编里，不受影响）。**因此本轮不提交 profile 改动**（`Cargo.toml` 已 `git restore`），`UPSTREAM.md` 不记。两组 APK 已留在（gitignore 的）`build/b6-compare/opt-3/` 与 `build/b6-compare/opt-s/`（fdroid 两 ABI 各一），供用户装同一台机对比：同一账号、同一作品、同一网络档，先开推荐/关注图片列表，再下一张大图，记墙钟秒数。用户判定可采用时，单独一个提交 `size(rhttp): set release opt-level to s`（一行改动 + UPSTREAM.md 一笔）即可。

### B7 阈值

以 B1–B6（B6 未采用）之后、带 `--obfuscate` 的 fdroid split 实测为基线（2026-09-07，与 B3 口径相同；B3 值减去 B5 的 `.so` 节省恰好等于实测）：

| ABI | 实测文件字节 | 阈值（实测 + 1,000,000） | 余量 |
|---|---|---|---|
| arm64-v8a | 27,435,321 | 28,435,321 | 1,000,000 |
| armeabi-v7a | 23,226,795 | 24,226,795 | 1,000,000 |

写入 `ci.yml`（`android-release`、`android-size` 两处）与 `release.yml`：`${{ vars.PIXIV_APK_MAX_BYTES_ARM64_V8A || '28435321' }}` / `${{ vars.PIXIV_APK_MAX_BYTES_ARMEABI_V7A || '24226795' }}`，仓库 vars 可覆盖；arm64 另受 B3 的硬顶 32,000,000（脚本取更严者）。注释含「F 迁 material_ui 后需重测」。本机验证：同一脚本对上表产物通过（exit 0）；把 armeabi-v7a 阈值临时设为 23,000,000 时报 `::error::armeabi-v7a APK exceeds size cap: 23226795 > 23000000` 并 exit 1。github flavor 比 fdroid 大 24,600 B（B3 表），在余量之内。

### CI 复核（PR #4 `android-size`，干净 runner）

`https://github.com/Lopution/pixiv-func/actions/runs/34144929182`，job 12 m 36 s：`rustup toolchain install`（`rust-toolchain.toml`）+ `Swatinem/rust-cache` + fdroid split/obfuscate + 体积报告 + `--analyze-size` + 两份 artifact 全部成功。runner 产物 arm64 27,809,465 / armeabi-v7a 23,525,215（B3 阶段，B5 之前），与本机 27,805,369 / 23,523,039 相差 4,096 / 2,176 字节（zip 对齐页），`lib/<abi>` 桶 25,626,344 / 21,368,564。`android-release` 在 `Decode release keystore` 处失败（secrets 未配置，现状，B 不绕过）；`android-unit` 6 m 9 s 通过。

### 图片 CacheManager 配置（抄录，不改）

`lib/core/network/compat/network_policy.dart:1162-1170`：`CacheManager(Config('pixiv_func_images', fileService: HttpFileService(httpClient: client(PixivDestinationPurpose.image))))`，未传 `stalePeriod` / `maxNrOfCacheObjects`。flutter_cache_manager 3.4.2 `lib/src/config/_config_io.dart:14-15` 默认：`stalePeriod = Duration(days: 30)`、`maxNrOfCacheObjects = 200`。本 child 不改配置；仅记录以便 parent 讨论磁盘占用时有据可查。

### `rustls-platform-verifier` 评估：保留

`plugins/rhttp/rhttp/rust/Cargo.toml:25,30`（`rustls-platform-verifier = "0.7"`，android 目标同版）。`init.rs` 每进程 `init_with_env`；app 的 TLS 路径走 webpki（`rhttp_client_factory.dart` + Dart 默认 `RootCertSource.webpki`），但 `RootCertSource.platform` 是插件公开能力，去掉它会增大 fork diff、砍掉平台信任库这条路，且 `android/build.gradle.kts` 为它解 AAR 的逻辑要跟着改。B5 未触碰；保留。

### child B 收束（B0–B7）

| 项 | 结果 | 依据 |
|---|---|---|
| release 产物 | 恰好 2 个 split APK（arm64-v8a、armeabi-v7a），无 universal、无 x86_64 | B3 / CI `android-size` |
| arm64-v8a release APK | 27,435,321 B（≤ 32,000,000 硬顶；也低于 §6 表「≤ 28 MB」推算值） | B7 实测 |
| armeabi-v7a release APK | 23,226,795 B | B7 实测 |
| 相对 fat 88.6 MB | −69% | B0 → B7 |
| `libsqlite3.so` | 可行，已排除（每 ABI −1.7 MB） | B1 |
| cargokit 陈旧 ABI | 单 ABI 构建只含目标 ABI 的 `librhttp.so` | B2 |
| `--obfuscate` | 保留（`libapp.so` −1,638,400 / −1,966,080）；`widgetBackgroundMain` 待用户真机 | B3 |
| updater | schema 2 多资产，按 `supportedAbis` 选择 | B4 |
| rhttp features | multipart / form / socks / cookies / charset 去掉，tokio 收窄；`query` 保留（B5e 回退）；`librhttp.so` −370,048 / −296,244 | B5 |
| `opt-level = "s"` | 测得 −1,539,000 / −884,944，未采用，待用户真机墙钟 | B6 |
| CI 门禁 | 28,435,321 / 24,226,795，vars 可覆盖，arm64 硬顶 32,000,000 | B7 |

## 7. 对既有契约的影响（必须在 design 中处理）

1. **updater manifest**：`tool/update_release.py generate` 当前接受单一 `--asset-url` 与单一
   APK 的 sha256；per-ABI 发布需要 manifest 携带按 ABI 区分的资产（url + sha256 + size），
   Kotlin 侧按 `Build.SUPPORTED_ABIS` 选择。这属于 `09-01-release-blockers` 的契约，需要与
   其协调，不能由本 task 单方面改格式。
2. **versionCode**：`--split-per-abi` 时 Flutter 为每个 split 加 `1000 * abiIndex`
   （`android/app/build.gradle.kts:31-34` 注释已说明），updater 的版本比较必须按 versionName
   或按去掉 ABI 偏移后的 versionCode 进行；F-Droid 多 APK 同版本反而要求 versionCode 互异，
   所以偏移应保留而不是用 `-Pforce-version-code-ignoring-abi=true` 抹平。
3. **F-Droid**：per-ABI 多 APK 是 F-Droid 支持的发布形态，但需要 metadata 侧对应配置；本 task
   只负责让构建输出满足要求，不负责 F-Droid 上架流程。
4. **CI**：`ci.yml` 的 android-release 检查与 `release.yml` 都以单一 `app-github-release.apk`
   路径为前提，需要改为遍历 split 产物、逐个验签。

## 实施时复核（child A，f24c9a1，2026-09-07）

测量环境：工作树 `/root/Pixiv-func-A`，`task/09-07-dependency-toolchain-health` @ `f24c9a1`。
Flutter 3.47.0 / Dart 3.13.0 / cargo 1.98.0 / AGP 9.1.0 / Gradle 9.3.1。新鲜树（无
`.dart_tool/`、`build/`、cargo `target/`、`android/local.properties`）。命令：
`flutter build apk --release --flavor fdroid`（fat，三 ABI，未 `--split-per-abi` /
`--obfuscate`）。开始 `2026-09-07T11:56:26Z`，Gradle `assembleFdroidRelease` 306.4 s，
墙钟约 5 min 8 s（crate 源已在 `~/.cargo/registry`，首编比预计的 20–40 min 短）。
产物 `build/app/outputs/flutter-apk/app-fdroid-release.apk`，文件 88,582,315 B
（Flutter 打印 88.6MB）。

### A0 基线（删 `cupertino_icons` 前）

Python `zipfile` 按 `compress_size` 汇总（native `.so` 均为 stored，compress = raw）：

| 类别 | compress_size | MB (1e6) |
|---|---|---|
| APK 文件 | 88,582,315 | 88.582 |
| zip 条目合计 | 88,334,227 | 88.334 |
| `lib/x86_64` | 31,810,096 | 31.810 |
| `lib/arm64-v8a` | 29,020,544 | 29.021 |
| `lib/armeabi-v7a` | 25,060,556 | 25.061 |
| `classes.dex` | 1,561,052 | 1.561 |
| `assets` | 635,806 | 0.636 |

`unzip -l … \| grep cupertino_icons`：

```
   257628  1981-01-01 01:01   assets/flutter_assets/packages/cupertino_icons/assets/CupertinoIcons.ttf
```

该条目 raw 257,628 B（0.258 MB）、compress 115,944 B（0.116 MB）。

与 §1 的 `app-github-release.apk`（88.6 MB / 31.81 / 29.02 / 25.06 / 1.56 / ~0.62，
`cupertino_icons` 0.26 MB / 压缩 0.12 MB）在 0.01 MB 内一致。本次数的是 **fdroid**
fat APK，不是 github；flavor 差异未体现在这些桶上。`lib/arm64-v8a` 内
`librhttp.so` 5,439,920 B、`libsqlite3.so` 1,732,360 B、`libflutter.so` 11,747,528 B、
`libapp.so` 9,962,376 B，与 §1 单 ABI 表（5.44 / 1.73 / 11.75 / 9.96 MB）一致。
新鲜树，无 §0.2 陈旧 ABI 污染。

### A1 之后（已删 `cupertino_icons`）

同一命令 `flutter build apk --release --flavor fdroid`（增量，Gradle 25.1 s，
开始 `2026-09-07T12:06:47Z`）。`unzip -l … \| grep cupertino_icons` 无输出；
`assets/flutter_assets/packages/` 下已无任何包目录。

| 类别 | A0 | A1 | Δ |
|---|---|---|---|
| APK 文件 | 88,582,315 | 88,466,077 | −116,238（−0.116 MB） |
| zip 条目合计 | 88,334,227 | 88,218,210 | −116,017 |
| `lib/x86_64` | 31,810,096 | 31,810,096 | 0 |
| `lib/arm64-v8a` | 29,020,544 | 29,020,544 | 0 |
| `lib/armeabi-v7a` | 25,060,556 | 25,060,556 | 0 |
| `classes.dex` | 1,561,052 | 1,561,052 | 0 |
| `assets` | 635,806 | 519,789 | −116,017（−0.116 MB） |

Flutter 打印从 88.6MB → 88.5MB。APK 文件少 0.116 MB，对齐字体压缩后体积
（A0 的 115,944 B）加上 zip 目录开销；未压缩 0.258 MB 是条目 raw size，不是
下载体积。`lib/*` 与 `classes.dex` 字节未变。

### A3 之后（pub upgrade + cargo update）

`flutter build apk --release --flavor fdroid`（3.47.2，新 Cargo.lock 重编三 ABI
`.so`）。开始 `2026-09-07T12:42:01Z`，Gradle 230.5 s，墙钟约 4 min 6 s。
产物 88,443,521 B（Flutter 打印 88.4MB），相对 A2 的 88,466,077 B 为 **−22,556 B**。
`cupertino_icons` 仍不在包内。`assets` 519,789 → 513,620（`NOTICES.Z` 随 lock
去掉一批仅测试用传递依赖而缩小）。

`librhttp.so`（stored = raw）相对 A2：

| ABI | A2 | A3 | Δ |
|---|---|---|---|
| `arm64-v8a` | 5,439,920 | 5,412,048 | −27,872 |
| `armeabi-v7a` | 3,651,244 | 3,636,460 | −14,784 |
| `x86_64` | 6,702,128 | 6,738,312 | +36,184 |

### A6 之后（archive 4.2.0 / image 4.9.2）

`flutter build apk --release --flavor fdroid`（3.47.2）。开始 `2026-09-07T13:45:28Z`，
Gradle 50.9 s，墙钟约 57 s。产物 88,443,452 B，相对 A4 的 88,443,344 B 为 **+108 B**。
`assets` 513,620 → 513,726（+106；`NOTICES.Z` 随 lock：+`posix` 6.5.2，−`petitparser`/`xml`）。
`classes.dex` 1,561,052 未变。`lib/*` 未变（含三 ABI `libapp.so`；image 4.9 未增大 AOT 快照）。
`cupertino_icons` 仍不在包内。

### A7 之后（flutter_secure_storage 11 / AGP 9.1.1 / compileSdk 37 / androidx.core 1.19.0）

`flutter build apk --release --flavor fdroid`（3.47.2）。开始 `2026-09-07T14:05:24Z`，
Gradle 77.2 s，墙钟约 83 s。产物 88,181,654 B，相对 A6 的 88,443,452 B 为 **−261,798 B**。
`classes.dex` 1,561,052 → 1,309,768（−251,284；R8 对 core 1.19 / secure_storage 11 的收缩），
`assets` −1，res/其它 +180，`lib/*` 未变（三 ABI `libapp.so` 与 `librhttp.so` 逐字节一致）。
`aapt2 dump badging`：`compileSdkVersion='37'`、`minSdkVersion:'29'`、`targetSdkVersion:'36'`（target 未变）。

### child A 收尾（A10 之后，HEAD da55454，全量检查复测）

A8–A10 不改产物，fdroid release 复测为 88,181,654 B（与 A7 相同）。github flavor
（`-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`，本机无正式 keystore）为 88,206,254 B，
两 flavor 的 `lib/*` 逐字节一致，`cupertino_icons` 均为 0 项。

| ABI | `librhttp.so` | `libapp.so` |
|---|---|---|
| arm64-v8a | 5,412,048 | 9,962,376 |
| armeabi-v7a | 3,636,460 | 10,994,248 |
| x86_64 | 6,738,312 | 10,224,520 |

child A 全程：A0 基线 → 收尾，fdroid release 净变化见各小节；`cupertino_icons` 字体已从包内移除，
其余体积工作留给 child B（per-ABI 拆分、`libsqlite3.so`、`librhttp.so` 特性裁剪）。

### F8 最终复测（child F，2026-09-10）

测量环境：工作树 `/root/Pixiv-func-F`，`task/09-07-interaction-visual-modernization` @ `75b870b`；
Flutter 3.47.2，fdroid split，带 `--obfuscate --split-debug-info`。命令：

```bash
flutter build apk --release --flavor fdroid \
  --split-per-abi --target-platform android-arm64,android-arm \
  --obfuscate --split-debug-info=build/symbols/fdroid
```

`tool/apk_size_report.py` 的精确结果（`file_bytes` 是 APK 下载文件字节，桶为 ZIP
`compress_size`）：

| ABI | APK 文件 | `lib/<abi>` | `classes.dex` | `assets` | F8 默认阈值 |
|---|---:|---:|---:|---:|---:|
| arm64-v8a | 27,981,038 | 25,776,488 | 1,313,356 | 518,544 | 28,981,038 |
| armeabi-v7a | 23,805,284 | 21,627,200 | 1,313,356 | 518,544 | 24,805,284 |

相对 B7，arm64-v8a 增加 545,717 B，armeabi-v7a 增加 578,489 B；arm64 仍低于
32,000,000 B 硬上限。两个 split 均只有自己的 `lib/<abi>` 目录，未生成 universal 或
x86_64 release APK。使用 F8 默认阈值运行 size report 退出码为 0；默认值为各 ABI
实测值加 1,000,000 B。
