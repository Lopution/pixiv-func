# 依赖与工具链升级审计

日期：2026-09-07　数据来源：`flutter pub outdated`（主项目与 `plugins/rhttp/rhttp`）、
`cargo update --dry-run`、`cargo info`、pub.dev / crates.io API、Codeberg 提交记录、GitHub
Releases API、Flutter releases feed、AndroidX/AGP/Kotlin/Gradle 发布页。

本文件是 09-02 "可迭代性"目标的输入：哪些组件现在就该动、哪些必须一起动、哪些不该动。
所有版本号为审计日当天的事实，实现时以当时的 `pub outdated` 为准重新核对。

> 2026-09-07 晚更新（D-11/D-12）：用户决定整个 app 迁到 `material_ui`/`cupertino_ui` 并采用 `go_router` 架构。
> 因此 §1 的 `go_router` **不再删除**（child F 升到 18 并接线），§5 的 `cached_network_image` 4 与 `go_router` 18
> 在 child F 内随 app 迁移一起升级；child A 只删 `cupertino_icons`。§1/§5 的事实描述保留不改。

## 1. 声明了但未使用的直接依赖（直接删除）

| 依赖 | 证据 |
|---|---|
| `go_router: ^17.0.0` | `lib/` 与 `test/` 无任何 `package:go_router/` import，`GoRoute(` 0 命中；导航使用 `lib/app/replica_page_route.dart` 与 `lib/app/navigation/replica_route.dart`（`PageRouteBuilder`）。自初始提交 `09160b9` 起从未使用。 |
| `cupertino_icons: ^1.0.8` | `CupertinoIcons.` 0 处使用；该包只是字体资产，当前被打进 APK（`assets/flutter_assets/packages/cupertino_icons/…` 0.26 MB）。 |

顺带效果：删除 `go_router` 后不再面对其 18.0 迁移到 `material_ui` 的问题（见 §5）。

## 2. 低风险、可直接做

| 组件 | 当前 → 目标 | 说明 |
|---|---|---|
| Flutter SDK | 3.47.0 → 3.47.2（2026-08-27，Dart 3.13.2） | 同 minor hotfix。3.47.2 更新 libpng 修复安全漏洞、修 Linux 触摸事件泄漏等；3.47.1 加入插件类名/包名校验防止 `GeneratedPluginRegistrant` 代码注入。官方 Android 验证矩阵（JDK 17 / KGP 2.4.0 / AGP 9.1.0 / Gradle 9.3.1）不变。本地 SDK 在 `/opt/flutter-3.47.0`（目录名含版本，`android/local.properties` 的 `flutter.sdk` 指向它）；CI 在 `ci.yml` 两处、`release.yml` 一处写死 `"3.47.0"`。 |
| `androidx.core:core` | 1.15.0（2024-11）→ 1.19.0（2026-06-03） | 落后 4 个 minor，仅版本号变更。 |
| `androidx.work` | `work-runtime-ktx:2.11.2` → `work-runtime:2.11.2` | 2.11 起 ktx artifact 为空壳，`CoroutineWorker` 等已并入主包；2.12.0-rc01 已发布但未 stable。 |
| pub 锁内 patch | `flutter pub upgrade`（不改 pubspec） | 21 项：`riverpod`/`flutter_riverpod` 3.4.2→3.4.3、`webview_flutter_android` 4.14.0→4.14.1、`webview_flutter_wkwebview` 3.26.0→3.26.1、`shared_preferences_android` 2.4.27→2.4.28、`shared_preferences_foundation` 2.5.6→2.5.7、`jni_flutter` 1.0.2→1.0.3、`analyzer` 13.3→14.3、`package_config` 2.2→3.0 等。 |
| Cargo.lock（`plugins/rhttp/rhttp/rust`，已入 git） | `cargo update` | 116 个 crate 语义兼容更新。TLS/HTTP 栈：`rustls` 0.23.40→0.23.43、`aws-lc-rs` 1.17.0→1.18.1、`aws-lc-sys` 0.41.0→0.45.0、`rustls-webpki` 0.103.13→0.103.15、`webpki-root-certs` 1.0.7→1.0.9（根证书库）、`hyper` 1.10.1→1.11.1、`h2` 0.4.14→0.4.19、`tokio` 1.52.3→1.53.1、`quinn` 0.11.9→0.11.11。`flutter_rust_bridge` 为 `=2.12.0` 精确 pin，不会被动。改动后需 `cargo test` 并重编 `.so` 跑插件 Dart 测试。 |
| `actions/checkout` | `@v4` → `@v6`（最新 v7.0.1，2026-07-20） | v6/v7 运行于 node24，ubuntu-latest 支持。 |
| `actions/setup-java` | `@v4` → `@v5` | 官方 README 已将 v1–v4 标为 deprecated；v6.0.0（2026-08-24）刚发布，README 仍推荐 v5 用于生产。 |
| Kotlin Gradle Plugin | 2.4.0 → 2.4.10（2026-07-14） | 同 release line bug-fix。Flutter 3.47 验证矩阵写的是 2.4.0；2.4.20 计划 9 月发布（RC3）。可选。 |

## 3. 值得做、但要跑测试的大版本

### `archive` 3.6.1 → 4.2.0 与 `image` 4.3.0 → 4.9.2（耦合）

- `image` ≥ 4.4.0 依赖 `archive ^4`（pub.dev 元数据：4.4.0/4.5.0 → `^4.0.1`，4.6.0 →
  `^4.0.2`，4.7.0/4.8.0 → `^4.0.7`，4.9.x → `^4.0.9`）。当前 `archive ^3.6.1` 把 `image` 卡在
  2024-10 的 4.3.0（`pub outdated`：Upgradable 4.3.0 / Resolvable 4.9.2）。
- 项目对 `archive` 的使用面：仅 `lib/core/ugoira/ugoira_zip.dart:5` 的
  `import 'package:archive/archive.dart' show Inflate, getCrc32;`，ZIP 目录解析为自研实现。
  两个符号在 archive 4.2.0 仍导出（`lib/archive.dart` 导出 `src/codecs/zlib/inflate.dart`
  与 `src/util/crc32.dart`；`Inflate(List<int> bytes, {OutputStream? output, int? uncompressedSize})`）。
  4.0.0 的 breaking（`InputStream`→`InputMemoryStream`、`decodeBuffer`→`decodeStream`、
  4.0.8 移除 `Crc32` 类）均不涉及。
- `image` 在 `lib/core/ugoira/ugoira_export.dart` 使用 `GifEncoder`、`QuantizerType.neural`、
  `Image.fromBytes`，4.x 内稳定。
- 副作用：解锁传递依赖 `xml` 6.6.1 → 7.x（`image` 4.9.2 已不依赖 `xml`）。
- 验证：`test/ugoira_test.dart` + 手测 Ugoira 播放与 GIF 导出。

### `flutter_secure_storage` 10.3.1 → 11.0.0（2026-08-06）

- 11.0 删除 v10 已弃用项：`KeyCipherAlgorithm.RSA_ECB_PKCS1Padding`、
  `StorageCipherAlgorithm.AES_CBC_PKCS7Padding`、`AndroidOptions.encryptedSharedPreferences`、
  `sharedPreferencesName`。项目两处调用（`lib/core/auth/credential_store.dart:33`、
  `lib/core/comments/translation_credentials.dart:87`）均为 `const FlutterSecureStorage()`，
  无自定义 options，代码零改动。
- 官方要求已有数据先在 v10 完成迁移；项目尚未发布（README："暂未提供可用发行版"），
  无存量用户数据。
- Android 侧 minSdk 24 / compileSdk 37：项目 minSdk 29 满足；compileSdk 37 高于 Flutter 默认
  36（`FlutterExtension.kt` `compileSdkVersion = 36`）。AGP 9.1.1 发布说明明确"supports
  Android API level 37.0 and below"，建议同时 AGP 9.1.0 → 9.1.1（同系列 patch，2026-04-13），
  并确认本机与 CI 安装 `platforms;android-37`。
- 验证：`flutter build apk --debug` 走通 Gradle；手测登录凭据与翻译凭据读写。

## 4. 跟上游、暂不单独动

| 组件 | 现状 | 原因 |
|---|---|---|
| `flutter_rust_bridge` | 主项目 `pubspec.lock` 解析到 Dart 运行时 **2.13.0**；rhttp 生成代码 `lib/src/rust/frb_generated.dart:74` `codegenVersion => '2.12.0'`；`rust/Cargo.toml:11` `=2.12.0`；本机 `flutter_rust_bridge_codegen` 2.12.0 | 三方不一致，但 `plugins/rhttp/rhttp/lib/src/rhttp.dart:19` 传了 `forceSameCodegenVersion: false`，运行时只告警不抛错。上游 rhttp 仍在 2.12.0，`UPSTREAM.md` 要求最小 diff；上游 bump 后再四方同步（Dart `^`、Cargo `=`、重跑 `flutter_rust_bridge_codegen generate`）。 |
| rhttp 上游 | vendored `9871f1c`（2026-07-08）= Codeberg `main` HEAD；0.18.0 为最新发布 | 无需同步。 |
| rhttp dev 工具链 | `ffigen` 11.0 / `freezed` 3.2.5 / `build_runner` 2.15.1（可用 21.0 / 4.0.1 / 2.16.1） | 只影响重新生成绑定时的工具链，不进 APK；跟上游。 |
| AGP / Gradle / JDK | 9.1.0 / 9.3.1 / 17（可用 AGP 9.3.1 / Gradle 9.7.1） | 当前组合 = Flutter 3.47 官方验证矩阵。AGP 9.3 要求 Gradle ≥ 9.5.0，Kotlin 2.4.0 官方兼容上限 Gradle 9.5.0。随 Flutter 3.48 stable（beta 3.48.0-0.4.pre 已于 2026-09-03 发布）一起升。 |
| Rust 直接依赖主版本 | `reqwest` 0.13.4、`rustls` 0.23、`rustls-platform-verifier` 0.7.0、`jni` 0.22.4、`ndk-context` 0.1.1、`chrono` 0.4.45 | 均为 crates.io 最新主版本，只有锁内 patch（§2）。 |
| `subosito/flutter-action` | `@v2`（最新 v2.23.0） | 仍在 v2 线。 |
| `flutter_lints` | 6.0.0 | 最新。 |

## 5. 不建议现在动：`cached_network_image` 3.4.1 → 4.0.0

- 4.0.0（2026-08-25）唯一的 breaking 是把 `flutter/material.dart` 换成 `material_ui`
  （Flutter 3.47 拆出的独立包），并要求 Flutter ≥ 3.44。
- 应用有 56 个文件 import SDK 内置 `package:flutter/material.dart`；3.47 SDK 的
  `packages/flutter/lib/material.dart` 仍是完整实现而非 `material_ui` 的重导出。升级会把第二份
  Material 实现编进快照，且 `material_ui.Theme.of` 与 SDK `Theme.of` 使用不同的
  InheritedWidget 类型，互不可见。
- `material_ui` 官方只提供"应用已迁 `material_ui`、旧插件仍用 `flutter/material`"方向的
  `MaterialUiCompatibilityBridge`，没有反向桥。
- 结论：`cached_network_image` 4.x 应与"整个应用迁移到 `material_ui`"绑定为同一个决策，
  不在本 task 内单独升级；但本 task 应把 `material_ui` 迁移列为**升级项目**目标下的一个
  显式候选（Flutter 官方路线是 Material/Cupertino 脱离 SDK，独立发版）。

## 6. 与 09-02 各目标的对应

- 可迭代性：§1 删依赖、§2 的 CI/Kotlin/androidx、§3 的耦合升级，都应作为"依赖健康"阶段
  一次性做完并写入 CI 的常规检查（`flutter pub outdated`、`cargo update --dry-run`）。
- 简洁性：§1 的 `cupertino_icons` 与 §5 的 material_ui 决策直接影响产物；
  `sqflite`/`sqflite_common_ffi` 双栈见 `apk-size-breakdown.md` §5。
- 升级项目：Flutter hotfix、`material_ui` 路线、FRB 版本对齐流程。
- 可维护性：FRB 三方版本必须有一处可执行的核对脚本，否则每次升级都会重新踩坑。

## 实施时复核（child A，f24c9a1，2026-09-07）

测量环境：工作树 `/root/Pixiv-func-A`，`task/09-07-dependency-toolchain-health` @ `f24c9a1`。
Flutter 3.47.0 / Dart 3.13.0 / cargo 1.98.0。命令：`flutter pub get --enforce-lockfile` 后
`flutter pub outdated`；`cd plugins/rhttp/rhttp/rust && cargo update --dry-run`（未写 lock）。

### `flutter pub outdated` 直接依赖

| Package | Current | Upgradable | Resolvable | Latest |
|---|---|---|---|---|
| `archive` | 3.6.1 | 3.6.1 | 4.2.0 | 4.2.0 |
| `cached_network_image` | 3.4.1 | 3.4.1 | 4.0.0 | 4.0.0 |
| `flutter_riverpod` | 3.4.2 | 3.4.3 | 3.4.3 | 3.4.3 |
| `flutter_secure_storage` | 10.3.1 | 10.3.1 | 11.0.0 | 11.0.0 |
| `go_router` | 17.5.0 | 17.5.0 | 18.0.1 | 18.0.1 |
| `image` | 4.3.0 | 4.3.0 | 4.9.2 | 4.9.2 |

dev_dependencies 全部 up-to-date。工具提示 22 个 lock 内可 `flutter pub upgrade` 的传递依赖、
8 个受约束挡住的 resolvable 大版本（含上表的 `archive` / `image` / `flutter_secure_storage` /
`go_router` / `cached_network_image`）。与审计日 §2/§3/§5 的直接依赖表一致，无新增落后项。

### `cargo update --dry-run`

`Locking 122 packages to latest compatible versions`（审计日 §2 写的是 116）。lock 未写入。
相对 §2 的差异：`rustls` 目标从审计日的 0.23.43 变为 **0.23.44**；`getrandom` 0.3.4→0.4.3、
`rand` 0.9.4→0.10.2 是比 §2 列举的 TLS/HTTP patch 更大的兼容线跳跃。新增 crate 包括
`chacha20`、`core_detect`、`cpufeatures`、`miniz_oxide`、`multiversion`/`multiversion-macros`/
`multiversion_no_op`、`rand_pcg`、`syn 3.0.5`、`target-features`、`zlib-rs`。

会改动的 crate：

```
aho-corasick 1.1.4 → 1.1.5
alloc-stdlib 0.2.2 → 0.2.4
android_system_properties 0.1.5 → 0.1.6
anyhow 1.0.102 → 1.0.104
async-compression 0.4.42 → 0.4.44
aws-lc-rs 1.17.0 → 1.18.1
aws-lc-sys 0.41.0 → 0.45.0
bitflags 2.13.0 → 2.13.1
brotli 8.0.3 → 8.0.4
brotli-decompressor 5.0.1 → 5.0.3
bytemuck 1.25.0 → 1.25.2
bytes 1.11.1 → 1.12.1
cc 1.2.63 → 1.4.5
cfg_aliases 0.2.1 → 0.2.2
+ chacha20 0.10.2
combine 4.6.7 → 4.6.8
compression-codecs 0.4.38 → 0.4.39
compression-core 0.4.32 → 0.4.33
cookie 0.18.1 → 0.18.2
+ core_detect 1.0.0
+ cpufeatures 0.3.1
crc32fast 1.5.0 → 1.5.1
delegate-attr 0.3.0 → 0.3.1
displaydoc 0.2.6 → 0.2.7
encoding_rs 0.8.35 → 0.8.40
find-msvc-tools 0.1.9 → 0.1.12
flate2 1.1.9 → 1.1.10
futures{,-channel,-core,-executor,-io,-macro,-sink,-task,-util} 0.3.32 → 0.3.34
getrandom 0.3.4 → 0.4.3
h2 0.4.14 → 0.4.19
hermit-abi 0.5.2 → 0.5.3
http 1.4.1 → 1.5.0
http-body 1.0.1 → 1.1.0
http-body-util 0.1.3 → 0.1.5
hyper 1.10.1 → 1.11.1
icu_{collections,locale_core,normalizer,normalizer_data,properties,properties_data} 2.2.0 → 2.3.0
icu_provider 2.2.0 → 2.3.1
indexmap 2.14.0 → 2.14.2
ipnet 2.12.0 → 2.12.2
jobserver 0.1.34 → 0.1.35
js-sys 0.3.99 → 0.3.105
libc 0.2.186 → 0.2.189
litemap 0.8.2 → 0.8.3
log 0.4.32 → 0.4.34
memchr 2.8.1 → 2.8.3
+ miniz_oxide 0.9.1
mio 1.2.1 → 1.2.3
+ multiversion 0.8.0 / multiversion-macros 0.8.0 / multiversion_no_op 1.0.0
pkg-config 0.3.33 → 0.3.34
portable-atomic 1.13.1 → 1.15.0
potential_utf 0.1.5 → 0.1.6
− ppv-lite86 0.2.21
proc-macro2 1.0.106 → 1.0.107
quinn 0.11.9 → 0.11.11
quinn-proto 0.11.14 → 0.11.17
quinn-udp 0.5.14 → 0.5.15
quote 1.0.45 → 1.0.47
r-efi 5.3.0 → 6.0.0
rand 0.9.4 → 0.10.2
− rand_chacha 0.9.0
rand_core 0.9.5 → 0.10.1
+ rand_pcg 0.10.2
regex 1.12.3 → 1.13.1
regex-automata 0.4.14 → 0.4.18
regex-syntax 0.8.10 → 0.8.11
rustc-demangle 0.1.27 → 0.1.28
rustc-hash 2.1.2 → 2.1.3
rustls 0.23.40 → 0.23.44
rustls-pki-types 1.14.1 → 1.15.1
rustls-webpki 0.103.13 → 0.103.15
rustversion 1.0.22 → 1.0.23
serde / serde_core / serde_derive 1.0.228 → 1.0.229
serde_json 1.0.150 → 1.0.151
simd-adler32 0.3.9 → 0.3.10
simd_cesu8 1.1.1 → 1.2.0
smallvec 1.15.1 → 1.16.0
socket2 0.6.4 → 0.6.5
syn 2.0.117 → 2.0.119；+ syn 3.0.5
+ target-features 0.1.6
thiserror / thiserror-impl 2.0.18 → 2.0.20
time 0.3.47 → 0.3.55
time-core 0.1.8 → 0.1.9
time-macros 0.2.27 → 0.2.32
tinystr 0.8.3 → 0.8.4
tinyvec 1.11.0 → 1.13.2
tokio 1.52.3 → 1.53.1
tokio-macros 2.7.0 → 2.7.2
tokio-rustls 0.26.4 → 0.26.5
tokio-util 0.7.18 → 0.7.19
− wasip2 1.0.3+wasi-0.2.9
wasm-bindgen{,-macro,-macro-support,-shared} 0.2.122 → 0.2.128
wasm-bindgen-futures 0.4.72 → 0.4.78
web-sys 0.3.99 → 0.3.105
webpki-root-certs 1.0.7 → 1.0.9
− windows-sys 0.60.2 及 windows-targets / windows_* 0.53.x 一组
− wit-bindgen 0.57.1
writeable 0.6.3 → 0.6.4
− zerocopy / zerocopy-derive 0.8.50
zeroize 1.8.2 → 1.9.0
zerotrie 0.2.4 → 0.2.5
zerovec 0.11.6 → 0.11.8
zerovec-derive 0.11.3 → 0.11.6
+ zlib-rs 0.6.7
zmij 1.0.21 → 1.0.23
zstd-safe 7.2.4 → 7.3.0
zstd-sys 2.0.16+zstd.1.5.7 → 2.1.0+zstd.1.5.7
```

`flutter_rust_bridge` 仍为 Cargo `=2.12.0` pin，dry-run 未改。verbose 提示另有 3 个 unchanged
dependencies behind latest（未展开）。

### A3 实际 cargo update

`cd plugins/rhttp/rhttp/rust && cargo update` 写入 lock：`Locking 122 packages`，
与 A0 dry-run 一致。`flutter_rust_bridge` 仍为 Cargo `=2.12.0`。落地版本：
`rustls` 0.23.40→**0.23.44**、`tokio` 1.52.3→**1.53.1**、`hyper` 1.10.1→**1.11.1**、
`aws-lc-rs` 1.17.0→**1.18.1**、`aws-lc-sys` 0.41.0→**0.45.0**、
`webpki-root-certs` 1.0.7→**1.0.9**、`h2` 0.4.14→0.4.19、`quinn` 0.11.9→0.11.11。

新 crate（`cargo metadata` 相对更新前）：`chacha20` 0.10.2（MIT OR Apache-2.0）、
`core_detect` 1.0.0（MIT/Apache-2.0）、`cpufeatures` 0.3.1（MIT OR Apache-2.0）、
`multiversion` / `multiversion-macros` 0.8.0（MIT OR Apache-2.0）、
`multiversion_no_op` 1.0.0（Apache-2.0 OR MIT）、`rand_pcg` 0.10.2（MIT OR Apache-2.0）、
`target-features` 0.1.6（MIT OR Apache-2.0）、`zlib-rs` 0.6.7（Zlib）；另增同名第二版本
`miniz_oxide` 0.9.1、`syn` 3.0.5。许可证字符串变化：`zstd-safe` / `zstd-sys` 从
MIT OR Apache-2.0 改为 **BSD-3-Clause**；`android_system_properties` 仅 SPDX 写法
`MIT/Apache-2.0` → `MIT OR Apache-2.0`。仓库无按 crate 列名的 NOTICE/THIRD 清单
（根 `NOTICE` 只指向历史 license-audit），未新建。
