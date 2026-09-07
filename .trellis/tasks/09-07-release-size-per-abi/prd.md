# 发布体积：per-ABI 拆分与 native 裁剪（09-02 child B）

## Goal

把用户下载的 release 包从 88.6 MB（三 ABI fat APK）收敛到 per-ABI arm64-v8a ≤ 32 MB（仅靠拆分，构成已实测），
再通过 native 裁剪向 ≤ 28 MB 收敛（推算值；若 `libsqlite3.so` 排除不可行则 ≤ 30 MB）。所有数字以
`unzip -l` 前后对比为准，写回 `../09-02-performance-size-maintainability-refactor/research/apk-size-breakdown.md` §6。
对应 parent R3（体积部分）、design §4；已确认决策 D-1、D-2、D-4、D-5。

## 开工 gate

- child A 完成（lock 刷新、`cupertino_icons` 删除是基线的一部分）。
- `09-01-release-blockers` 的 R5（多资产 manifest schema 2）已落地，或与其并行且以其为 schema owner；
  本 child 只做消费端与流水线，不单方面定义 manifest 字段。
- 开工第一步：清理 `build/rhttp/jniLibs/*`（陈旧 ABI 污染）后重跑三 ABI 与单 ABI 基线。

## Requirements

### R1. per-ABI 发布流水线（D-1、D-2）

- `ci.yml`、`release.yml` 改为 `flutter build apk --release --flavor <f> --split-per-abi --target-platform android-arm64,android-arm --obfuscate --split-debug-info=build/symbols/<f>`；
  产出 `app-<f>-arm64-v8a-release.apk`、`app-<f>-armeabi-v7a-release.apk`；不产出 x86_64、不产出 universal。
- versionCode 保留 Flutter 偏移（arm64 `2000+n`、armeabi-v7a `1000+n`）；`build.gradle.kts` 注释更新。
- 两个 APK 逐个 `apksigner verify`、拒绝 debug 签名；符号目录作为 release artifact 上传；`release.yml` 上传两个 APK。
- 不使用 `--target-platform` 单独作为单 ABI 手段（cargokit 与插件 jniLibs 不受其约束，研究 §2）。

### R2. updater 消费端（D-3 的消费侧）

- `update_service.dart` 按 `supportedAbis` 选资产、semver + `versionCode % 1000` 比较；无匹配 ABI 时给出可诊断失败。
- Kotlin `platformInfo` 返回 `supportedAbis`；`tool/RELEASE.md` 更新为两 APK 流程。
- 若 release-blockers 已按 schema 1 发布过任何版本，实现 schema 1/2 双读一个周期。

### R3. 单 ABI 内裁剪（每项一个提交，附前后 `.so`/APK 大小）

- cargokit：`plugins/rhttp/rhttp/cargokit/gradle/plugin.gradle` cargo 任务前 `delete(cargoOutputDir)`，消除陈旧 ABI 混入（实测 10.4 MB）。
- `libsqlite3.so`（D-4 保留桌面分支与主依赖）：`android/app/build.gradle.kts` `packaging { jniLibs { excludes += "**/libsqlite3.so" } }`；
  新增测试断言 Android 平台工厂为 `sqflite.databaseFactory`。**待验证**排除对 Flutter native asset 是否生效；不可行则记录并放弃。
- rhttp reqwest feature：按 `native-and-build-audit.md` B.2 去掉应用零使用的 `multipart`、`form`、`socks`、`cookies`、`query`、`charset`；
  `tokio = full` 收窄。每去一项：cargokit 重编 → 插件 `flutter test` → `cargo test` → `test/rhttp_client_factory_test.dart` +
  `restricted_compat_network_test.dart` → 真机登录/图片/下载。压缩四项保留（Pixiv 依赖 gzip；zstd/brotli 单独实验）。
- rhttp `[profile.release] opt-level = "s"` 实验：记录 `.so` 大小与一次图片列表/大图下载耗时；不可接受则回退。
- `rustls-platform-verifier` 默认保留，只记录评估结论。
- 以上 Rust/cargokit 差异按 D-5 记入 `UPSTREAM.md`。

### R4. 体积门禁与记录

- CI 输出每个 split APK 的总大小与 `lib/<abi>`、`classes.dex`、assets 明细；阈值 = R3 完成后实测 + 1 MB，超出失败。
- `--analyze-size` JSON 作为 artifact 归档（不阻塞）。
- 每步更新 `research/apk-size-breakdown.md` §6。

### R5. 不做

- 不替换 aws-lc-rs（ECH 依赖 HPKE）、不删压缩支持、不删 API 29、不删 flavor、不删测试；不改路由阶梯/ECH/SNI/证书语义。
- 不负责 F-Droid 元数据与其构建服务器配置。

## Acceptance Criteria

- [ ] release 产物为 2 个 split APK（arm64-v8a、armeabi-v7a），无 universal、无 x86_64；两者验签通过。
- [ ] arm64-v8a release APK ≤ 32 MB；R3 各项落地后的实测值写入研究文件 §6。
- [ ] API 29 与高版本 Android 真机各完成一次按 ABI 选择资产的真实自更新。
- [ ] `--obfuscate` 后 widget 后台入口（`@pragma('vm:entry-point')` `widgetBackgroundMain`）真机验证正常；符号目录随 release 归档。
- [ ] 单 ABI 构建后 APK 只含目标 ABI 的 `librhttp.so`（cargokit 清理生效）。
- [ ] 每项 rhttp 裁剪有前后测量与双侧测试记录；ECH/TLS 真机无回归。
- [ ] CI 体积门禁生效；`UPSTREAM.md` 记录全部构建配置差异。
- [ ] `flutter analyze`、`flutter test`、插件测试、`cargo test`、双 flavor 构建通过。

## Notes

- 复杂 child：`task.py start` 前需从 parent design §4 与 implement "Child B" 派生本目录的 `design.md`（manifest 消费、
  Gradle/CI 改动细节、裁剪顺序）与 `implement.md`（B0–B8 清单）。
- 停止条件：任一裁剪导致 TLS 握手/解压/ECH 回归立即回退；`--obfuscate` 破坏 widget 入口则去掉并记录。
