# 实施清单：09-07-release-size-per-abi

对应 `prd.md`、`design.md`；步骤编号与 parent `implement.md` Child B 对齐，每步一个提交（B3+B4 两个提交同一 PR）。

## 执行前状态

- 工作树：`/root/Pixiv-func-B`（`main` 的 git worktree）
- 分支：`task/09-07-release-size-per-abi`（从 `main` 新切）
- HEAD：`cf7296e`（`git rev-parse --short HEAD`；完整 `cf7296ea1de67e9f61c3a82491a824cd3763531b`）
- HEAD 说明：`Merge pull request #2 from Lopution/task/09-07-dependency-toolchain-health`（child A 已合入）
- child A 收尾体积（待 B0 在干净 `jniLibs` 上复核）：fdroid fat `88,181,654` B；`librhttp.so` `5,412,048` / `3,636,460` / `6,738,312`；`libapp.so` `9,962,376` / `10,994,248` / `10,224,520`

## 工具链前置

按 `.trellis/spec/frontend/quality-guidelines.md`「Build Toolchain」：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
export ANDROID_HOME=/opt/android-sdk
# 不要手改 android/local.properties
flutter pub get --enforce-lockfile
```

github flavor 本机 release 必须加 `-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`。fdroid 不需要。正式 keystore 只存在于 CI secrets / 用户本机 `~/.pixivfunc-release/`；本工作树没有，也不得创建进仓库。

每步若改了 Dart/Rust/Gradle/workflow，收尾至少跑：

```bash
flutter analyze
flutter test
(cd plugins/rhttp/rhttp && flutter test)
(cd plugins/rhttp/rhttp/rust && cargo test --locked)
git diff --check
```

全量 `flutter test` 若出现无关文件的 `TimeoutException after 0:00:30`，先按 frontend spec 的 loopback 噪声判断，单文件重跑对照。

## 并行与顺序

- **B0–B2 彼此独立**（B0 建议最先，给后面提供干净基线；B1 与 B2 不互相依赖，可换序）。
- **B3+B4 必须一起落地**（流水线产出与 manifest/消费端同一 PR；两个 checkbox、两条 commit message）。
- **B5 严格串行**，一个 feature 一个提交，顺序 `multipart → form → socks → cookies → query → charset`，最后收窄 `tokio`。
- **B6 在 B5 全部完成之后**（对照才是「裁完 features 的 `.so`」）。
- **B7 在 B6 之后**（阈值 = B1–B6 实测 + 1 MB）。
- **B8 最后**（把各步已写入 §6 的数字收束，并抄 CacheManager Config）。

## 清单

- [x] **B0 基线**
  - 做：`rm -rf build/rhttp/jniLibs`（消除 `native-and-build-audit.md` §0.2 的陈旧 ABI 污染，上次泄漏 `10,353,372` B）。然后：
    1. 三 ABI fat：`flutter build apk --release --flavor fdroid`（实测：3.47.2 不允许多 ABI 带 `--analyze-size`；JSON 由单独一次 `--target-platform android-arm64 --analyze-size --code-size-directory=/tmp/pixiv-size-b0-fat` 补齐，不作基线）
    2. 干净 split（**不加** obfuscate，作为「只拆 ABI」对照）：`flutter build apk --release --flavor fdroid --split-per-abi --target-platform android-arm64,android-arm`
  - 触及：仅 `../09-02-performance-size-maintainability-refactor/research/apk-size-breakdown.md` §6（新建「B0」小节）。
  - 测量写入 §6：fat 文件字节（对照 `88,181,654`）；两份 split 的文件字节与 python 桶（`lib/<abi>`、`classes.dex`、`assets`）；三 ABI `librhttp.so` / `libapp.so` / `libsqlite3.so` raw 字节；`--analyze-size` 目录路径。禁止用单独 `--target-platform` 当「单 ABI 基线」。
  - 验证：parent 验证命令里的 python `zipfile` 片段；`unzip -l …/app-fdroid-release.apk | grep cupertino_icons` 仍为空。
  - 提交：`docs(size): record B0 clean ABI baseline`
  - 回滚：revert 该文档提交。
  - 真机：无。

- [x] **B1 `libsqlite3.so`**
  - 做：`android/app/build.gradle.kts` 的 `android { }` 在 `buildTypes` 后加 `packaging { jniLibs { excludes += "**/libsqlite3.so" } }`。抽出 `@visibleForTesting` 工厂选择（见 `design.md`），`history_database.dart:27-31` 生产路径仍是 `Platform.isAndroid || Platform.isIOS → sqflite.databaseFactory`。新测试断言 Android/mobile 工厂是 `sqflite.databaseFactory` 不是 `databaseFactoryFfi`。
  - 触及：`android/app/build.gradle.kts`、`lib/core/history/history_database.dart`、`test/` 下新测试（建议 `test/history_database_factory_test.dart`）。不删 `sqflite_common_ffi`。
  - 验证：
    ```bash
    flutter test test/history_database_factory_test.dart test/history_persistence_test.dart
    flutter build apk --release --flavor fdroid --split-per-abi --target-platform android-arm64,android-arm
    unzip -l build/app/outputs/flutter-apk/app-arm64-v8a-fdroid-release.apk | grep libsqlite3.so   # 必须无输出
    unzip -l build/app/outputs/flutter-apk/app-armeabi-v7a-fdroid-release.apk | grep libsqlite3.so
    flutter build apk --debug --flavor fdroid
    unzip -l build/app/outputs/flutter-apk/app-fdroid-debug.apk | grep libsqlite3.so
    ```
    若 APK 仍含 `libsqlite3.so`（native asset 绕过 jniLibs）：§6 记录 `NativeAssetsManifest.json`（或实际清单路径）+ 命令，**放弃本项**，收敛目标改为 ≤ 30 MB，提交改为记录结论的文档（仍用下面这条 message 若排除成功；失败则 `docs(size): record sqlite3 exclude infeasible`）。
  - 测量写入 §6：排除前后 `lib/<abi>` 桶与 APK 文件字节；`libsqlite3.so` 是否仍存在。
  - 提交：`size(android): exclude unused sqlite3 native asset`
  - 回滚：revert 该提交即恢复打包。
  - 真机（待用户真机）：历史记录读写（debug + release）。

- [x] **B2 cargokit 清理**
  - 做：`plugins/rhttp/rhttp/cargokit/gradle/plugin.gradle` 的 `CargoKitBuildTask.build()` 在 `execOperations.exec`（约 `:71`）之前 `project.delete(outputDir)`。`plugins/rhttp/UPSTREAM.md` D-5 列表加一条「cargo 任务前删除 `jniLibs/<buildType>`」。
  - 触及：上述两个文件。
  - 验证（复现 §0.2 后再证明已修好）：
    ```bash
    # 若树是干净的，先做一次三 ABI 污染输出目录
    flutter build apk --release --flavor fdroid
    flutter build apk --release --flavor fdroid --target-platform android-arm64
    python3 - <<'EOF'
    import zipfile
    z=zipfile.ZipFile('build/app/outputs/flutter-apk/app-fdroid-release.apk')
    rhttp=[n for n in z.namelist() if n.endswith('librhttp.so')]
    print(rhttp)
    assert rhttp == ['lib/arm64-v8a/librhttp.so'], rhttp
    EOF
    ```
    只主张 `librhttp.so`；`libdartjni.so` 在非 split 下仍可能多 ABI（`design.md`）。
  - 测量写入 §6：清理前后该 APK 是否仍含非目标 ABI 的 `librhttp.so`；污染字节若可算则记下。
  - 提交：`build(rhttp): clean stale ABI outputs before cargo build`
  - 回滚：revert 该提交。
  - 真机：无。

- [x] **B3 per-ABI 流水线**（与 B4 同一 PR）
  - 做：
    - `.github/workflows/ci.yml` `android-release`：`flutter build apk` 改为 `--split-per-abi --target-platform android-arm64,android-arm --obfuscate --split-debug-info=build/symbols/github`；对 `app-arm64-v8a-github-release.apk` 与 `app-armeabi-v7a-github-release.apk` 循环 `apksigner verify`，拒绝 `Android Debug`；跑体积汇总脚本；arm64 `> 32_000_000` 即失败；`--analyze-size` JSON 上传 artifact（不阻塞）；`build/symbols/github` 上传 artifact。
    - `.github/workflows/ci.yml` 新增 **`android-size`** job（不依赖 secrets，见 `design.md`「体积汇总脚本」）：`checkout@v6`、`setup-java@v5`（temurin 17）、`subosito/flutter-action@v2`、`flutter pub get --enforce-lockfile`、`rustup toolchain install`（根 `rust-toolchain.toml`，含 Android targets）、fdroid split/obfuscate 构建、同一份体积汇总 + 阈值脚本（`app-fdroid-*`，arm64 硬顶 `32_000_000`）、上传 `--analyze-size` JSON（`continue-on-error`）与两个 fdroid split APK 为 artifact（`retention-days: 14`，供用户侧载验证）。体积脚本抽成 `tool/apk_size_report.py`（参数：APK 路径列表 + 可选阈值环境变量），两个 job 与本机调用同一文件，避免 workflow 内两份 python 漂移。首次运行必须核对 cargokit 在 runner 上的交叉编译（rustc/NDK 下载）耗时并写入 §6。
    - `.github/workflows/release.yml`：同样的 build 旗标；验签循环；两 APK 指纹必须一致；`generate` 改多 `--apk <abi>=<path>`；draft Release 上传两个命名 APK + manifest + sig（不再上传 fat `pixiv-func-v<ver>-github.apk`）；符号目录只做 Actions artifact。
    - `android/app/build.gradle.kts:33-36` 注释改为写明 `2000+n` / `1000+n`，禁止 `-Pforce-version-code-ignoring-abi`。
  - 触及：`ci.yml`（`android-release` 改造 + 新 `android-size`）、`release.yml`、`tool/apk_size_report.py`（新）、`build.gradle.kts`（仅注释）。
  - 验证（本机 fdroid 可完整跑；github 加 debug-signing 旗标）：
    ```bash
    flutter build apk --release --flavor fdroid \
      --split-per-abi --target-platform android-arm64,android-arm \
      --obfuscate --split-debug-info=build/symbols/fdroid
    ls -l build/app/outputs/flutter-apk/app-*-fdroid-release.apk
    # 必须恰好两份：arm64-v8a、armeabi-v7a；无 x86_64、无 universal
    flutter build apk --release --flavor github \
      --split-per-abi --target-platform android-arm64,android-arm \
      --obfuscate --split-debug-info=build/symbols/github \
      -PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true
    # github 本机是 debug 签名：apksigner 会看到 Android Debug——这是预期，
    # 用来确认产物路径存在；CI 在正式 keystore 下必须拒绝 Debug。
    python3 - <<'EOF'
    import zipfile,collections,sys,glob
    for apk in glob.glob('build/app/outputs/flutter-apk/app-*-release.apk'):
        z=zipfile.ZipFile(apk); agg=collections.defaultdict(int)
        for i in z.infolist():
            k=i.filename.split('/')[1] if i.filename.startswith('lib/') else i.filename.split('/')[0]
            agg[k]+=i.compress_size
        print(apk, f"{sum(agg.values())/1e6:.1f} MB", {k:f"{v/1e6:.1f}" for k,v in sorted(agg.items(),key=lambda kv:-kv[1])[:6]})
    EOF
    ```
    断言：每个 split APK 的 `lib/` 只有一个 ABI 目录；arm64 APK `≤ 32_000_000`。
  - 测量写入 §6：B3 后两 flavor × 两 ABI 的文件字节与桶；`libapp.so` 相对 B0（obfuscate 效应）。
  - 提交：`release: ship per-ABI APKs`
  - 回滚：与 B4 一起 revert（见「回滚点」）。
  - 真机（待用户真机）：`--obfuscate` 后 widget 后台入口（`widgetBackgroundMain`）——加 widget，确认 `WidgetBackground: entered` 与出图。失败则去掉 obfuscate 旗标并记录（停止条件）。
  - 注意：`android-release` 在用户配置 keystore secrets 之前会红；体积门禁与 fdroid 产物由 `android-size` 在每个 PR 上产出，PR 合并前必须看到 `android-size` 绿并把它的汇总数字抄进 §6（与本机 fdroid 对照）。

- [x] **B4 updater 多资产**（与 B3 同一 PR）
  - 做：按 R5 与 `design.md` contracts：
    - `tool/update_release.py`：可重复 `--apk <abi>=<path>`；顶层 `schema: 2`、`versionCode` 为基数、`packageName` / `signingCertificateSha256` 在顶层；`assets[]` 含 `abi/url/size/sha256/versionCode`；URL 固定 `pixiv-func-v<version>-github-<abi>.apk`；`self-test` 改两 APK。
    - `lib/core/updater/update_manifest.dart`：只认 `schema == 2`；精确键集见 design；schema 1 → `UpdateManifestFormatException('schema')`。
    - `update_models.dart` / `update_platform.dart`：`UpdatePlatformInfo.supportedAbis`。
    - `update_service.dart`：按 `supportedAbis` 选第一个匹配 `assets[].abi`；无匹配 → `UpdateCheckStatus.invalid` + `errorCode: 'abi_unsupported'`（**不要** `up_to_date`）；semver 之后比较 `manifest.versionCode` 与 `info.versionCode % 1000`。下载/验签/安装不改语义。
    - 两份 `DistributionUpdaterChannel.kt` 的 `platformInfo` 增加 `supportedAbis`。
    - `tool/RELEASE.md` 改为两 APK 流程。
    - 测试：`test/updater_manifest_test.dart`、`updater_download_test.dart`、`updater_flavor_contract_test.dart` fixture 从 schema 1 迁到 2；新增 ABI 命中 / `abi_unsupported` / schema 1 拒绝 / `% 1000` 比较（`versionCode: 2001` 的安装值对基数 `1` 为 up_to_date，对基数 `2` 为 available）。
  - 触及：`tool/update_release.py`、`tool/RELEASE.md`、`lib/core/updater/update_manifest.dart`、`update_models.dart`、`update_service.dart`、`update_platform.dart`、两份 `DistributionUpdaterChannel.kt`、`test/updater_*.dart`。
  - **计划中、此刻不做**：在 `../09-01-release-blockers/prd.md` R5 末加一行交叉引用（`design.md`「目标与边界」原文）。
  - 验证：
    ```bash
    python3 tool/update_release.py self-test
    flutter test test/updater_manifest_test.dart test/updater_download_test.dart test/updater_flavor_contract_test.dart test/updater_about_test.dart
    ```
  - 测量写入 §6：无体积项；记「schema 2 生成器自测通过」。
  - 提交：`updater: multi-asset manifest (schema 2)`
  - 回滚：与 B3 一起。
  - 真机（待用户真机）：API 29 与高版本各一次**按 ABI 选择资产**的真实自更新；manifest 缺少本机 ABI 时 UI 可见 `abi_unsupported`，不得显示「已是最新」。依赖正式签名包与 draft Release，须 secrets 已配置。

- [ ] **B5 rhttp feature 裁剪**（七个提交，顺序固定）
  - 每步：改 `plugins/rhttp/rhttp/rust/Cargo.toml` 去掉该 feature → 若 rustc 失败，在 `http.rs` / `client.rs` 把对应分支改为返回 `RhttpError`（不跑 `flutter_rust_bridge_codegen`）→ 清理 `build/rhttp/jniLibs` → fdroid split 重编 → 测 `.so` / APK → 跑测试 → `UPSTREAM.md` 追加该 feature。
  - 每步验证：
    ```bash
    (cd plugins/rhttp/rhttp && flutter test)
    (cd plugins/rhttp/rhttp/rust && cargo test --locked)
    flutter test test/rhttp_client_factory_test.dart test/restricted_compat_network_test.dart
    flutter build apk --release --flavor fdroid \
      --split-per-abi --target-platform android-arm64,android-arm
    # 从 APK 取出 librhttp.so 字节写入 §6
    ```
    github 本机对照需要时加 `-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`。
  - 测量写入 §6：`design.md` 测量表全部列（两 ABI `.so`、arm64 APK、`lib/arm64-v8a` 桶、编译修补、测试结果）。对照 B0/B3 的 `5,412,048` / `3,636,460`。
  - 真机（待用户真机，**每步**）：登录 + 图片列表 + 大图下载。TLS 握手 / 解压 / ECH 任一回归 → 立即 `git revert` **该**提交（停止条件原文）。
  - 提交（message 固定）：
    - B5a `size(rhttp): drop reqwest multipart feature`
    - B5b `size(rhttp): drop reqwest form feature`
    - B5c `size(rhttp): drop reqwest socks feature`
    - B5d `size(rhttp): drop reqwest cookies feature`
    - B5e `size(rhttp): drop reqwest query feature`
    - B5f `size(rhttp): drop reqwest charset feature`
    - B5g `size(rhttp): narrow tokio features`  
      （`full` → `rt-multi-thread, net, time, sync, io-util, macros`，按 rustc 补齐，不预先扩大）
  - 不做：去掉 `gzip/deflate/brotli/zstd`；不换 aws-lc-rs；不删 `rustls-platform-verifier`。
  - 回滚：单步 revert。

- [ ] **B6 `opt-level = "s"` 实验**
  - 做：严格按 `design.md` 协议。只改 `Cargo.toml:57`。测量 `.so` + APK；**待用户真机**一次图片列表 / 大图下载墙钟（同一作品、同一网络档，对照 B5g）。
  - 采用 → 提交 `size(rhttp): set release opt-level to s`，UPSTREAM.md 记一笔。
  - 不采用 → **不提交** profile 改动（`git restore` `Cargo.toml`），数字写入 §6「不采用」。不要空提交。
  - 验证：同 B5 的插件/cargo/应用测试 + fdroid split 构建。
  - 回滚：若已提交则 revert；若未提交则无代码边界。
  - 停止条件：吞吐不可接受视为「不采用」，不是缺陷。

- [ ] **B7 阈值**
  - 做：把 B1–B6 之后的 fdroid arm64 / armeabi-v7a split APK 实测文件字节 `+ 1_000_000` 写入 `ci.yml`（`android-size` 与 `android-release` 两处）/ `release.yml` 的默认值，并在注释写 **「F 迁 material_ui 后需重测」**。读取 `vars.PIXIV_APK_MAX_BYTES_ARM64_V8A` / `vars.PIXIV_APK_MAX_BYTES_ARMEABI_V7A`（env 可覆盖）。超出失败。arm64 另保留 B3 起的硬顶 `32_000_000`（取更严者）。
  - 触及：`ci.yml`、`release.yml`、`tool/apk_size_report.py`（脚本已在 B3 落地，本步只把「打印」改成「比较」并填数字）。
  - 验证：对本机 fdroid split 跑同一脚本，确认当前产物低于阈值；把阈值临时改成 `1` 确认会失败（再改回）。CI：`android-size` 在 PR 上绿即门禁生效；github 路径等 secrets。
  - 测量写入 §6：两 ABI 的「实测 / 阈值 / 余量」。
  - 提交：`ci: enforce per-ABI APK size budget`
  - 回滚：revert 该提交后恢复只打印。
  - 真机：无。

- [ ] **B8 记录**
  - 做：收束 `research/apk-size-breakdown.md` §6（B0–B7 全部数字、sqlite 可行/放弃、obfuscate 保留/放弃、rhttp 逐步表、B6 结论、`rustls-platform-verifier` 评估「保留」）。抄录图片 CacheManager：`Config('pixiv_func_images')`，默认 `stalePeriod = Duration(days: 30)`、`maxNrOfCacheObjects = 200`（`network_policy.dart:1162-1170` + flutter_cache_manager 3.4.2 `_config_io.dart:14-15`），**不改配置**。阈值注释已含「F 迁 material_ui 后需重测」。
  - 触及：仅研究文件（及若 B7 漏写那句注释则补 workflow 注释）。
  - 验证：§6 无「待填 / TODO / 约」；每个 B5 步都有前后字节。
  - 提交：`docs(size): record per-ABI trim results and cache config`
  - 真机：无。

- [ ] **退出条件**
  - release 产物为 2 个 split APK（arm64-v8a、armeabi-v7a），无 universal、无 x86_64；两者在 **正式签名** 下 `apksigner verify` 通过（CI，待 secrets）。
  - arm64-v8a release APK ≤ 32 MB（B3 后）；B1–B6 后的实际值在 §6。
  - 单 ABI 构建后 APK 只含目标 ABI 的 `librhttp.so`（B2）。
  - 每项 rhttp 裁剪有前后测量与双侧测试记录。
  - CI 体积门禁生效（B7）；`UPSTREAM.md` 记录全部构建配置差异（清理 + 每个被删 feature + 可选 `opt-level`）。
  - `flutter analyze`、`flutter test`、插件测试、`cargo test --locked`、双 flavor 构建通过。
  - **待用户真机（退出前必须勾，agent 不能代做）**：API 29 与高版本按 ABI 自更新；`--obfuscate` 后 `widgetBackgroundMain`；B5/B6 的登录/图片/下载与 ECH/TLS；B1 的历史读写。

## 停止条件（parent 原文）

任一 feature 裁剪导致 TLS 握手/解压/ECH 回归立即回退该提交；`--obfuscate` 导致 widget 后台入口失效（`@pragma('vm:entry-point')`）则去掉 obfuscate 并记录。

补充（本 child PRD）：裁剪导致的回归只回退**当前**提交；sqlite 排除若对 native asset 无效则记录并放弃，不升级为删依赖。

## 回滚点

1. B1 排除规则；B2 `delete(outputDir)`。
2. B3+B4 一对提交（流水线 ↔ schema 2）；revert 必须成对，避免 HEAD 停在「一边 fat、一边 schema 2」。
3. B5a–B5g 各一；B6 仅在采用时存在；B7 阈值。
4. 不使用 `git reset --hard` / `git clean`。

## 高风险文件（执行时对照 parent 原文）

- `plugins/rhttp/rhttp/rust/Cargo.toml`、cargokit：每一步重编并跑双侧测试；工具链不一致不提交生成物。
- `tool/update_release.py`、`lib/core/updater/*`：manifest 字节即签名内容；字段名必须与 release-blockers R5 一致。
- `lib/core/network/compat/*`：B5 测试会碰到，但不改路由阶梯/ECH/证书语义。
- `pubspec.yaml` / lock / `build.gradle.kts` / workflows：删除前先搜消费者。
