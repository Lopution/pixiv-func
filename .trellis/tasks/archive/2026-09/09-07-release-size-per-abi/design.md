# 设计：per-ABI 发布、schema 2 消费与 native 裁剪

对应 child PRD `prd.md`（R1–R5）、parent `design.md` §4 / §5.2、parent `implement.md` Child B。
schema 字段名的唯一 owner 是 `../09-01-release-blockers/prd.md` R5（仍为 `planning`）；本 child 按 R5 原文落地生成器、流水线与消费端，不发明、不改名。

## 目标与边界

B 做三件事：把 release 从单一 fat APK 改成两个 split APK；按 R5 生成并消费多资产 manifest；在单 ABI 内做可回退的 native 裁剪并设体积门禁。

B **不是** schema owner。R5 已经写死字段；B 只实现。release-blockers 归档前，在其 `prd.md` R5 段末增加一行交叉引用（**计划中的文档编辑，现在不动那个文件**）：

> 生成器、`release.yml` 多 APK 上传、消费端选资产与 `versionCode % 1000` 比较由 sibling `09-07-release-size-per-abi` 落地；本段仍是字段合同的唯一出处。

不做（PRD R5 / parent §15）：不替换 aws-lc-rs、不删压缩四项、不删 API 29、不删 flavor、不删测试、不改路由阶梯/ECH/SNI/证书语义、不负责 F-Droid 元数据与构建服务器。抽出两份 `DistributionUpdaterChannel.kt` 的公共 `platformInfo` 是 parent child D（D5），B 只在两份副本上各加 `supportedAbis`。

开工基线是 child A 收尾（`main` @ `cf7296e`）：fdroid fat `88,181,654` B，github fat（本机 debug 签名）`88,206,254` B；三 ABI `librhttp.so` `5,412,048` / `3,636,460` / `6,738,312`，`libapp.so` `9,962,376` / `10,994,248` / `10,224,520`。数字以 B0 清理 `build/rhttp/jniLibs` 后重测为准，写回 `../09-02-performance-size-maintainability-refactor/research/apk-size-breakdown.md` §6。

## 数据流

```
release.yml
  flutter build apk --release --flavor github
    --split-per-abi --target-platform android-arm64,android-arm
    --obfuscate --split-debug-info=build/symbols/github
    + 正式 keystore Gradle 属性
        │
        ├─ app-arm64-v8a-github-release.apk      (versionCode = 2000+n)
        ├─ app-armeabi-v7a-github-release.apk    (versionCode = 1000+n)
        └─ build/symbols/github/                 (Actions artifact，不上公开 Release)
        │
        ▼
  逐个 apksigner verify；拒绝 cert 含 "Android Debug"
        │
        ▼
  tool/update_release.py generate
    --apk arm64-v8a=<path> --apk armeabi-v7a=<path>
    --version <semver> --version-code <n>
    --signing-cert-sha256 <hex64> --key-file <pem> --out-dir ./assets
        │
        ▼
  update-manifest.json (schema 2) + update-manifest.sig
  （签名对象仍是 manifest 文件的整段字节，UTF-8、无尾换行、separators=(",",":")）
        │
        ▼
  gh release 上传：
    pixiv-func-v<ver>-github-arm64-v8a.apk
    pixiv-func-v<ver>-github-armeabi-v7a.apk
    update-manifest.json
    update-manifest.sig
        │
        ▼
  Dart UpdateManifest.parse          （只认 schema == 2）
        │
        ▼
  UpdateService._checkOnce
    platformInfo.supportedAbis 中第一个能在 assets[].abi 命中的资产
    先比 semver version，相等再比 manifest.versionCode 与 installed.versionCode % 1000
        │
        ▼
  下载 / 整包 sha256 / 平台 verifyApk / 安装   ← 现有路径，不改语义
```

`ci.yml` 的 `android-release` 走同一套 `flutter build apk` 旗标，但不跑 `update_release.py`、不建 GitHub Release；它负责验签循环、体积汇总、阈值（B7 后）和 `--analyze-size` 归档（不阻塞）。

本机可独立验证的是 **fdroid** flavor（无 keystore）。github flavor 本机构建必须加 `-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`，产物不可发布。`android-release` job 在用户补齐 keystore secrets 之前必然失败（现有 `Decode release keystore` 步骤在 secret 为空时 `exit 1`，见 `ci.yml:36-44`）。

## contracts

### Manifest schema 2（R5 原文，字段名冻结）

R5 点名的字段，B 必须原样使用：

| 位置 | 字段 | 含义 |
|---|---|---|
| 顶层 | `schema` | 必须为整数 `2` |
| 顶层 | `version` | semver 字符串 |
| 顶层 | `versionCode` | **不含** ABI 偏移的基数 `n`（`pubspec.yaml` 的 `+n`，今天是 `0.1.0+1` → `1`） |
| 顶层 | `packageName` | `io.github.lopution.pixivfunc` |
| 顶层 | `signingCertificateSha256` | 64 位小写 hex |
| `assets[]` | `abi` | `arm64-v8a` 或 `armeabi-v7a` |
| `assets[]` | `url` | `https://github.com/Lopution/Pixiv-func/releases/download/v<version>/pixiv-func-v<version>-github-<abi>.apk` |
| `assets[]` | `size` | APK 字节，`≤ updateAssetMaxBytes`（200 MiB，`update_manifest.dart:10`） |
| `assets[]` | `sha256` | 64 位小写 hex |
| `assets[]` | `versionCode` | **含**偏移的实际值：arm64 = `2000+n`，armeabi-v7a = `1000+n` |

R5 没有重写、也没有删除现有 schema 1 的身份字段。当前解析器（`update_manifest.dart:135-143`）用 `_exactKeys` 要求 `repository` / `tag` / `channel`；parent §4.2 示例 JSON 也保留它们。B **不发明新名**，也不单方面删掉它们，schema 2 的顶层精确键集是：

`schema`, `repository`, `tag`, `channel`, `version`, `versionCode`, `packageName`, `signingCertificateSha256`, `assets`

约束沿用现有代码：`repository == Lopution/Pixiv-func`，`tag == v$version`，`channel ∈ {stable, beta}`。`packageName` 与 `signingCertificateSha256` 从 schema 1 的 `asset` 对象升到顶层（R5 原文）；选中的 `UpdateReleaseAsset` 在解析后把这两项抄回去，这样 `update_download.dart:117`、`update_platform.dart:105-109` 的 `verifyApk` 调用不用改形。

`isStrictUpdateManifestAssetUrl`（`download_request.dart:182-186`：https + `github.com` + `/Lopution/Pixiv-func/releases/download/` + `.apk`）与 200 MiB 上限不变。签名仍覆盖整个 manifest 文件字节。

### `platformInfo`

两份 Kotlin `platformInfo`（github `DistributionUpdaterChannel.kt:101-115`，fdroid `:37-51`）现返回 `packageName` / `version` / `versionCode` / `signingCertificateSha256`。各加一项：

```kotlin
"supportedAbis" to Build.SUPPORTED_ABIS.toList()
```

类型是 `List<String>`（`Build.SUPPORTED_ABIS` 为 `Array<String>`）。Dart `UpdatePlatformInfo`（`update_models.dart:55-67`）增加 `supportedAbis`；`MethodChannelUpdatePlatform.info`（`update_platform.dart:53-70`）按现有 `platform_info_malformed` 校验——缺键、非 `List`、元素非 `String` 一律抛 `UpdatePlatformException('platform_info_malformed')`。fdroid 的 updater 仍 `enabled: false`，但 `getPlatformInfo` 契约与 github 对齐，避免 flavor 分叉。

### 无匹配 ABI 的错误表面

不引入新异常类型。选资产发生在 `UpdateManifest.parse` 之后、版本比较之前，走与 `asset_identity_mismatch`（`update_service.dart:380-388`）相同的服务层表面：

```dart
UpdateCheckResult(
  status: UpdateCheckStatus.invalid,
  errorCode: 'abi_unsupported',
)
```

禁止返回 `UpdateCheckStatus.noUpdate` / `errorCode: 'up_to_date'`（R5：「不静默 `up_to_date`」）。`UpdateCheckResult.release` 为 `null`。`updater_manifest_test.dart` 增加用例：`supportedAbis = ['x86_64']` 且 assets 只有两个 ARM ABI → `invalid` + `abi_unsupported`。

`UpdateManifestLike.asset`（`update_models.dart:103`）保持「已选中的单一资产」语义，供 `apply` / 下载复用；具体类另持 `assets` 全量。

版本判定（替换 `update_service.dart:402-404` 的直接 `manifest.versionCode <= info.versionCode`）：

1. `manifest.version.compareTo(currentVersion)`（现有 semver）
2. 仅当 semver 相等：`manifest.versionCode <= info.versionCode % 1000`
3. 不使用 `-Pforce-version-code-ignoring-abi`

今天 `info.versionCode` 来自 `PackageInfo.longVersionCode`（github `:103-108`）。split 之后 arm64 安装包是 `2000+n`，若不取模，基数 `n` 会永远小于安装值，新版本在 semver 相同时被误判 `up_to_date`。

### schema 1：拒绝，不双读

`gh release list` 在本仓库为空；`09-01-release-blockers` 仍是 `planning`；`tool/update_release.py:154` 与 `update_manifest.dart:144-146` 的 schema 1 从未对用户发布过。保留只读双读只会让测试和生成器分叉，且 parent §4.2 / PRD R2 的双读条件（「若已按 schema 1 发布过任何版本」）不成立。

决策：`schema != 2` 抛已有的 `UpdateManifestFormatException('schema')`（`update_service.dart:355-360` 已把它映成 `UpdateCheckStatus.invalid` + `errorCode: 'schema'`）。现有用例 `rejects unknown schema before release fields are trusted` 继续有效；再加一条 schema `1` 同样拒绝。生成器不再写 schema 1。

若后续发现已发布的 schema 1 资产，再按 parent 开一个周期的双读——那是新事实，不是本设计的默认路径。

## Gradle/CI 改动细节

### 构建旗标与产物名

两端统一为（`<f>` = `github` | `fdroid`）：

```bash
flutter build apk --release --flavor <f> \
  --split-per-abi \
  --target-platform android-arm64,android-arm \
  --obfuscate \
  --split-debug-info=build/symbols/<f>
```

产出（Flutter 3.47.2 的 split 命名，B0 实测：`app-<abi>-<flavor>-release.apk`，ABI 在 flavor 之前）：

- `build/app/outputs/flutter-apk/app-arm64-v8a-<f>-release.apk`
- `build/app/outputs/flutter-apk/app-armeabi-v7a-<f>-release.apk`

不产出 `*-x86_64-*`，不产出 `app-<f>-release.apk` universal。禁止用单独的 `--target-platform` 当发布手段（研究 `apk-size-breakdown.md` §2、`native-and-build-audit.md` §0.2：cargokit / 插件 jniLibs 不受该旗标约束，陈旧 ABI 曾泄漏 `10,353,372` B）。

github CI / `release.yml` 继续附加正式签名属性（`ci.yml:55-59`、`release.yml:55-60`）。本机 github 验证加 `-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`。fdroid 不需要任何签名属性，是本机与「secrets 落地前」的可复现路径。

### versionCode 注释

`android/app/build.gradle.kts:33-36` 现只写「`1000 * ABI_VERSION` 由 Flutter 自动加」。改为点名偏移，并写明禁止抹平：

- `armeabi-v7a`：Flutter `ABI_VERSION=1`（`FlutterPluginConstants.kt:41`）→ `1000+n`
- `arm64-v8a`：`ABI_VERSION=2`（同文件 `:42`）→ `2000+n`
- `x86_64` 的 `ABI_VERSION=4` 本流水线不构建
- 不要传 `-Pforce-version-code-ignoring-abi=true`（R5 / 研究 §7.2：F-Droid 多 APK 同版本要求 versionCode 互异）

`versionCode = flutter.versionCode` 赋值本身不变。

### 验签循环

把 `ci.yml:60-68`、`release.yml:64-77` 的单文件 `APK=.../app-github-release.apk` 改成对两个 split 路径循环：

```bash
for APK in \
  build/app/outputs/flutter-apk/app-arm64-v8a-github-release.apk \
  build/app/outputs/flutter-apk/app-armeabi-v7a-github-release.apk
do
  test -s "$APK" || { echo "::error::missing $APK"; exit 1; }
  "$APKSIGNER" verify --verbose --print-certs "$APK"
  if "$APKSIGNER" verify --print-certs "$APK" 2>/dev/null | grep -q 'Android Debug'; then
    echo "::error::debug-signed APK cannot pass: $APK"
    exit 1
  fi
done
```

`release.yml` 的指纹取 **两个 APK 必须相同** 的那一个 SHA-256（同一 keystore）；不一致则失败。该指纹写入 manifest 顶层 `signingCertificateSha256`。

### 体积汇总脚本

复用 parent「验证命令」的 python `zipfile` 片段，对每个 split APK 打印总 `compress_size` 与 `lib/<abi>`、`classes.dex`、`assets` 等桶。B3 先输出不失败；B7 拿 B1–B6 后的实测 `+ 1_000_000` 字节作为上限。

阈值通过 GitHub Actions `vars`（可被 job `env` 覆盖）注入，避免把数字硬编码进两次 workflow：

- `vars.PIXIV_APK_MAX_BYTES_ARM64_V8A`
- `vars.PIXIV_APK_MAX_BYTES_ARMEABI_V7A`

未设置时：B3–B6 不因阈值失败（只打印）；B7 把默认值写进 workflow，并要求仓库 vars 与之对齐。口径与研究文件一致，用 **1e6 的 MB**（32 MB = `32_000_000`）。B3 后 arm64 必须 `≤ 32_000_000`（验收「仅靠拆分」）；这是硬上限，即使 vars 未设也要在 B3 起对 arm64 生效，否则拆分回退无法被 CI 看见。armeabi-v7a 在 B7 前只打印。

**门禁必须在 secrets 缺失时也能生效。** `android-release` 在 keystore secrets 落地前必然红（`ci.yml:36-44`），若体积门禁只放在那里，B3–B7 期间 CI 对体积回退是盲的。因此 B3 在 `ci.yml` 新增不依赖 secrets 的 **`android-size`** job：fdroid flavor、同一套 split/obfuscate 旗标、`rustup toolchain install`（cargokit 交叉编译需要根 `rust-toolchain.toml` 的 Android targets；NDK 缺失时由 AGP 经 sdkmanager 下载）、体积汇总 + 阈值脚本（与 `android-release` 共用同一段脚本，只是文件名前缀 `app-fdroid-`）、`--analyze-size` JSON 与两个 fdroid split APK 作为 artifact 上传（后者供用户无签名侧载验证 widget 入口 / 历史读写 / rhttp 裁剪）。`android-release` 保留验签循环并复用同一阈值脚本；两处阈值来自同一对 `vars`。fdroid 与 github 的 `lib/*` 逐字节一致（child A 收尾：两 flavor 仅差 24,600 B 的 dex/资源），所以 fdroid 上的门禁对 github 产物同样有代表性。B7 改「打印」为「比较」时两处同时改。

`--analyze-size --code-size-directory=build/analyze-size/<f>` 的 JSON 作为 artifact 上传，`continue-on-error: true`，不阻塞。 Flutter 3.47.2 拒绝在多 ABI 构建上做 code-size analysis（B0 实测 `Cannot perform code size analysis when building for multiple ABIs`），所以该步是**单独一次** `--target-platform android-arm64 --analyze-size` 构建，只产 JSON，不当发布产物，也不当体积基线。

### `release.yml` 上传什么

公开 draft Release（现 `release.yml:98-109`）改为四个资产，不再上传 `pixiv-func-v<ver>-github.apk`：

1. `pixiv-func-v<ver>-github-arm64-v8a.apk`
2. `pixiv-func-v<ver>-github-armeabi-v7a.apk`
3. `assets/update-manifest.json`
4. `assets/update-manifest.sig`

`build/symbols/github/` 只上传为 Actions artifact（私有附件，parent §4.1「私有或附件」），不挂到公开 Release，避免把混淆符号表公开。

`generate` 从单 `--apk` + `--asset-url` 改为可重复的 `--apk <abi>=<path>`；URL 按 R5 规则由 version+abi 推导，不再接受任意 `--asset-url`（减少与命名合同漂移）。`self-test` 改为写两个假 APK 并断言 schema 2、两笔 `assets`、顶层 `versionCode` 为基数。

### secrets 与本机

`android-release` / `release.yml` 在 `PIXIV_RELEASE_KEYSTORE_B64` 等 secrets 未配置时保持失败——这是现状，B 不绕过。体积数字、cargokit 清理、sqlite 排除、rhttp 裁剪全部用 fdroid 验证：本机跑 fdroid split，CI 由 `android-size` job（上节）在每个 PR 上跑同一份 python 与阈值。github 路径的验签循环等 secrets。

## cargokit 清理

`plugins/rhttp/rhttp/cargokit/gradle/plugin.gradle`：

- `cargoOutputDir` 在 `:134`：`"${project.buildDir}/jniLibs/${buildType}"`（应用侧即 `build/rhttp/jniLibs/release`）
- `CargoKitBuildTask.build()` `@TaskAction` 从 `:49` 开始，在 `:71` 的 `execOperations.exec` 调 `build-gradle` **之前**插入：

```groovy
project.delete(outputDir)
```

`outputDir` 就是任务的 `cargoOutputDir`（`:162`）。`build_gradle.dart` 只 `copySync` 本次 target、从不删未请求的 ABI 子目录（研究 §0.2）；任务前 `delete` 是 D-5 允许的 vendored 差异，写入 `plugins/rhttp/UPSTREAM.md`「Build configuration diffs」列表（已有 compileSdk `tokenize('.')[0]`、FRB pin、`rust-toolchain.toml`、`frb_generated` rustfmt skip）。

证明「单 ABI 构建只有目标 ABI 的 `librhttp.so`」：

1. 故意先做一次三 ABI（或不 clean 的旧树）污染 `build/rhttp/jniLibs/release/{armeabi-v7a,x86_64}`
2. `flutter build apk --release --flavor fdroid --target-platform android-arm64`（**不用** `--split-per-abi`，这是 B2 的针对性复现，不是发布手段）
3. `unzip -l` 该 APK：`lib/arm64-v8a/librhttp.so` 恰好一份；`lib/armeabi-v7a/librhttp.so` 与 `lib/x86_64/librhttp.so` 不存在

`libdartjni.so` 在非 split 的 `--target-platform` 下仍可能三 ABI 全出（研究 §0.2 第 1 点，插件 AAR + `PLATFORM_ABI_LIST`）。B2 只主张 `librhttp.so`。B3 的 `--split-per-abi` 会设 `abiFilters`，jni 与 datastore 的 `.so` 随过滤器走。

## `libsqlite3.so` 排除

`android/app/build.gradle.kts` 的 `android { }` 今日没有 `packaging` 块（文件在 `:121` 结束）。在 `buildTypes` 之后加入：

```kotlin
packaging {
    jniLibs {
        excludes += "**/libsqlite3.so"
    }
}
```

D-4 保留桌面分支与 `sqflite_common_ffi` 主依赖；`sqlite3` 的 `hook/build.dart` 对每个 `targetOS` 编译，没有按 OS 跳过的开关（研究 §5）。Android 运行时已经走平台 SQLite：

```27:31:lib/core/history/history_database.dart
  static DatabaseFactory _platformDatabaseFactory() {
    if (Platform.isAndroid || Platform.isIOS) return sqflite.databaseFactory;
    sqfliteFfiInit();
    return databaseFactoryFfi;
  }
```

`flutter test` 跑在 Linux VM 上，`Platform.isAndroid` 为 false，不能假装安卓。抽出 `@visibleForTesting` 的纯函数（例如 `historyDatabaseFactory({required bool useMobileSqflite})`），生产 `_platformDatabaseFactory` 仍按 `Platform.isAndroid || Platform.isIOS` 调用；测试断言 `useMobileSqflite: true` 的返回值 `identical` 于 `sqflite.databaseFactory`，且不是 `databaseFactoryFfi`。现有 `test/history_persistence_test.dart` 继续注入 `databaseFactoryFfi`，不改。

验证：

- release / debug APK `unzip -l | grep libsqlite3.so` 为空
- 若 `assets/NativeAssetsManifest.json`（或等价 native-assets 清单）仍列出该库但 `lib/<abi>/` 没有 `.so`，视为排除成功（运行时不加载）
- 真机历史记录读写（待用户）

若 Flutter native asset 打包绕过 jniLibs 合并、APK 里仍然有 `libsqlite3.so`：在 §6 记录命令与清单路径，**放弃本项**，收敛目标从 ≤ 28 MB 改为 ≤ 30 MB（PRD R3 / parent B1）。不改为从 `pubspec` 删除 `sqflite_common_ffi`。

预期收益（若生效）：每 ABI −`1,732,360` B（child A 收尾 arm64 值；研究 §1 写 1.73 MB）。

## rhttp 裁剪顺序与测量表

`plugins/rhttp/rhttp/rust/Cargo.toml:32-51` 当前 reqwest features：`charset, cookies, form, http2, query, rustls, stream, multipart, socks, brotli, deflate, gzip, zstd`；`:21` `tokio = { version = "1.52", features = ["full"] }`；`:56-61` `[profile.release]` 已是 `opt-level = 3, strip, lto, codegen-units = 1, panic = abort`。

应用零使用（`native-and-build-audit.md` B.2）：`multipart` / `form` / `socks` / `cookies` / `query` / `charset`。压缩四项保留（Pixiv 依赖 gzip；zstd/brotli 不在本 child 实验）。`http2` / `rustls` / `stream` 保留。

顺序（每项一个提交，先改 `Cargo.toml`，编不过再做最小 Rust 修补——把对应分支改成返回 `RhttpError`，**不**跑 FRB codegen、不删 Dart 枚举）：

| 步 | 去掉 | 预期编译后果 | 证明无回归的测试 | 测量列 |
|---|---|---|---|---|
| B5a | `multipart` | `http.rs:414-443` 的 `reqwest::multipart::Form` / `Part` 编不过；`MultipartValue::File` 还用了 `tokio::fs`（`:421`） | 插件 `flutter test`；`cargo test --locked`（`ech_config_test` / `ech_live_handshake`）；`test/rhttp_client_factory_test.dart`（SNI/ECH/证书/ALPN）；`test/restricted_compat_network_test.dart` | 见下 |
| B5b | `form` | `http.rs:413` `request.form(&form)` 编不过 | 同上 | |
| B5c | `socks` | `client.rs:146-160` 的 `reqwest::Proxy::*` 在 `socks://` URL 上需要该 feature；纯 http/https proxy 可能仍能编过。app 零 `ProxySettings`（B.2） | 同上 | |
| B5d | `cookies` | `client.rs:165-166` `cookie_store(...)` 编不过；`cookie_store` / `publicsuffix` 应离图；`icu_*` 仍在（`url` 需要，B.3） | 同上 | |
| B5e | `query` | `http.rs:375-376` `request.query(&query)` 编不过。app 自行拼 URL | 同上 | |
| B5f | `charset` | 源码无直接 API，多半能直接编过；`encoding_rs` 离图（B.3） | 同上 | |
| B5g | `tokio` 收窄 | `full` → 先试 `rt-multi-thread, net, time, sync, io-util, macros`（parent §4.3）；缺 feature 按 rustc 报错补，不预先猜。B5a 去掉 multipart 后 `tokio::fs` 应不再被引用 | 同上；缺 `fs`/`process`/`signal` 是预期 | |

每步测量表（写入 §6，列固定，禁止只写「变小了」）：

| 列 | 内容 |
|---|---|
| 步 | B5a…B5g / B6 |
| 去掉的 feature 或 profile 项 | 原文 |
| `librhttp.so` arm64 字节 | stored = raw |
| `librhttp.so` armeabi-v7a 字节 | 同上 |
| fdroid arm64 split APK 文件字节 | `app-arm64-v8a-fdroid-release.apk` |
| `lib/arm64-v8a` 桶 | python 汇总 |
| 编译修补（若有） | 文件:行，或「无」 |
| 双侧测试 | 通过 / 失败摘要 |
| 真机 | 待用户：登录 + 图片列表 + 大图下载；TLS/ECH/解压任一回归 → `git revert` 该提交 |

基线对照（B0 重测后替换）：arm64 `librhttp.so` `5,412,048`，armeabi-v7a `3,636,460`。

### `opt-level = "s"` 实验协议（B6）

1. 以 B5g 后的 `.so` / APK 为对照，记入 §6。
2. 只改 `Cargo.toml:57` `opt-level = 3` → `opt-level = "s"`，其它 profile 键不动。
3. 清理 `build/rhttp/jniLibs` 后 fdroid `--split-per-abi` 重编（可暂不加 obfuscate，避免与 B3 符号段纠缠；若 B3 已合入则带相同旗标，使对比只剩 profile）。
4. 记录两 ABI `.so` 与 arm64 APK 字节。
5. **待用户真机**一次：打开推荐/关注图片列表，再下一张大图，记下墙钟（秒，同一账号、同一作品、同一网络档）。对照是 B5g 的同操作。
6. 采用门：`.so` 下降且列表+下载墙钟没有用户可感知的回退（由用户判定）。不满足则 `git restore` `Cargo.toml`，不提交 profile 改动，只在 §6 写「不采用」+ 数字。
7. 采用则提交 `size(rhttp): set release opt-level to s`，UPSTREAM.md 记一笔。

### `rustls-platform-verifier`

默认保留。B.2：`init.rs:70` 每进程 `init_with_env`，TLS 路径走 webpki（`rhttp_client_factory.dart` 注释 + Dart 默认 `RootCertSource.webpki`）。评估只写 §6：去掉会增大 fork diff，且 `RootCertSource.platform` 不可用；本 child 不改依赖。`android/build.gradle.kts` 为它解 AAR 的逻辑不动。

## `--obfuscate` 风险

唯一的应用入口 pragma 在 `lib/main.dart:33-34`：

```33:34:lib/main.dart
@pragma('vm:entry-point')
Future<void> widgetBackgroundMain() => runWidgetBackground();
```

实现是 `lib/core/widget/widget_background.dart:29` 的 `runWidgetBackground()`（独立 engine，自己 `Rhttp.init()`，写 `filesDir/widget_snapshot/active.json`）。parent §9 已写：`--obfuscate` 不影响 MethodChannel 字符串名，但必须回归这条入口。

B3 起 CI/本机 release 带 `--obfuscate --split-debug-info=build/symbols/<f>`。符号目录随 Actions artifact 走。

**待用户真机**（无法在本工作树代替）：

1. 安装 B3 的 github 或 fdroid obfuscated APK（github 若是 debug 签名，只能覆盖同签名的旧包）
2. 添加桌面 widget，等 WorkManager 后台跑完
3. 确认 widget 出图/出占位，而不是永久空白；`adb logcat` 能看到 `WidgetBackground: entered`（`widget_background.dart:36`）

失败：从 `ci.yml` / `release.yml` / 本机命令去掉 `--obfuscate --split-debug-info`，在 §6 与 `UPSTREAM.md` 记录「obfuscate 破坏 widget 入口，已放弃」。这是 parent 停止条件，整项回退，不尝试改名或加第二个 pragma 碰运气。

## 兼容与回滚

| 步 | 可否单独 revert | 说明 |
|---|---|---|
| B0 | 文档提交 | 只改研究文件 |
| B1 | 是 | 去掉 `packaging.jniLibs.excludes` + 测试即回到打包 `libsqlite3.so` |
| B2 | 是 | 去掉 `project.delete(outputDir)` + UPSTREAM 一行 |
| B3+B4 | **必须一起** | 流水线已上传两个 APK / schema 2，消费端必须能选 ABI；只合 B3 会让现网（尚无正式 release）生成器与解析器不一致；只合 B4 会让 `release.yml` 仍指不存在的 fat 路径。一次 PR 内两提交，revert 用两次 `git revert` 且同一 PR 完成 |
| B5a–B5g | 是，逐个 | 任一 TLS 握手 / 解压 / ECH 回归 → 立即 `git revert` **该**提交（parent 停止条件原文） |
| B6 | 是 | 不采用则根本不提交 |
| B7 | 是 | 去掉阈值比较，保留打印 |
| B8 | 文档提交 | |

失败只回滚当前提交。不使用 `git reset --hard` / `git clean`（parent §10、`implement.md` 推荐顺序）。`pubspec.lock` / `Cargo.lock` 冲突只允许 `flutter pub get` / `cargo update` 再生，不手改。

B3+B4 的 revert 边界：两个 commit 的 message 固定为 `release: ship per-ABI APKs` 与 `updater: multi-asset manifest (schema 2)`。先 revert 消费端再 revert 流水线，或相反，只要二者不同时停留在「一边 schema 2、一边 fat」的 HEAD。

## 与 F 的衔接

体积门禁阈值（B7 写入 workflow / vars 的注释）必须带原句：

> F 迁 material_ui 后需重测

parent D-11 / spec 升级策略：`cached_network_image` 4 与 `go_router` 18 随 F 的 app 迁移才动；那次升级可能增大 `libapp.so`（研究 §3：快照由 Flutter 框架主导，但 material 组件集变化会动 AOT）。B 不预抬阈值「给 F 留空」，F 负责重测后改 vars。

图片 `CacheManager` 的 `Config` **只记录、不改**。唯一构造在 `lib/core/network/compat/network_policy.dart:1162-1170`：

```1162:1170:lib/core/network/compat/network_policy.dart
  CacheManager get imageCacheManager {
    return _imageCacheManager ??= CacheManager(
      Config(
        'pixiv_func_images',
        fileService: HttpFileService(
          httpClient: client(PixivDestinationPurpose.image),
        ),
      ),
    );
  }
```

应用没有传 `stalePeriod` / `maxNrOfCacheObjects`。`flutter_cache_manager` 3.4.2 的 IO 默认（`.pub-cache/.../flutter_cache_manager-3.4.2/lib/src/config/_config_io.dart:14-15`）是 `stalePeriod = Duration(days: 30)`、`maxNrOfCacheObjects = 200`。B8 把这三行（cache key、30 天、200）抄进 §6，标明「F 不得在未重测体积的情况下顺手改 Config」。
