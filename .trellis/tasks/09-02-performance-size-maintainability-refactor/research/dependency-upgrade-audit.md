# 依赖与工具链升级审计

日期：2026-09-07　数据来源：`flutter pub outdated`（主项目与 `plugins/rhttp/rhttp`）、
`cargo update --dry-run`、`cargo info`、pub.dev / crates.io API、Codeberg 提交记录、GitHub
Releases API、Flutter releases feed、AndroidX/AGP/Kotlin/Gradle 发布页。

本文件是 09-02 "可迭代性"目标的输入：哪些组件现在就该动、哪些必须一起动、哪些不该动。
所有版本号为审计日当天的事实，实现时以当时的 `pub outdated` 为准重新核对。

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
