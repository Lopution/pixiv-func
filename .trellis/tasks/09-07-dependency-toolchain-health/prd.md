# 依赖与工具链健康（09-02 child A）

## Goal

让项目的依赖、SDK、构建工具链和 CI 回到"当前稳定且可持续升级"的状态，并建立日后升级不再踩坑的
校验：删除零引用依赖、完成低风险与耦合升级、固化 Rust/Dart 锁文件与工具链、把 Kotlin/插件/Rust
测试纳入 CI。对应 parent `prd.md` R2，事实来源 `../09-02-performance-size-maintainability-refactor/research/dependency-upgrade-audit.md`。

## 开工 gate

- 用户批准本 PRD。与 09-01 进行中改动只在 `pubspec.yaml` 行级可能冲突；开工时 `git status --short` 存档。
- 无其它前置。本 child 是 B 的前置（`cupertino_icons` 删除与 lock 刷新是体积基线的一部分）。

## Requirements

### R1. 删除零引用依赖

- `cupertino_icons ^1.0.8`：`lib/`、`test/` 均 0 import（研究 §1）。删除后 `flutter pub get`，确认 APK 中
  `assets/flutter_assets/packages/cupertino_icons/` 消失（−0.26 MB）。
- `go_router ^17.0.0` 当前同样 0 import，但按 parent D-12 **保留**，由 child F 升到 18 并接线；本 child 不动它。

### R2. 低风险升级（每项一个提交，每项跑全量检查）

- Flutter 3.47.0 → 3.47.2：本地 SDK；`.github/workflows/ci.yml` 两处与 `release.yml` 一处
  `flutter-version`。3.47.2 含 libpng 安全修复，3.47.1 含插件注册器注入校验。
- `androidx.work:work-runtime-ktx` → `work-runtime` 2.11.2（ktx 已空壳）。`androidx.core:core` 1.19.0
  要求 compileSdk 37，并入 R3 的 AGP 9.1.1 项。
- `flutter pub upgrade`（不改约束）；`plugins/rhttp/rhttp/rust` 下 `cargo update`（含 rustls 0.23.43、
  aws-lc-rs 1.18.1、webpki-root-certs 1.0.9、hyper 1.11.1、tokio 1.53.1 等）；核对新 crate 许可证；
  重编 `.so` 后跑插件 `flutter test` 与 `cargo test`。
- `actions/checkout@v4` → `@v6`；`actions/setup-java@v4` → `@v5`。
- Kotlin 2.4.0 → 2.4.10 可选（Flutter 3.47 验证矩阵为 2.4.0；若不动需在 spec 注明理由）。

### R3. 耦合升级（需测试）

- `archive ^3.6.1` → `^4.2.0`，解锁 `image` 4.3.0 → 4.9.x。项目只用 `Inflate`/`getCrc32`（4.2.0 仍导出）
  与 `GifEncoder`/`QuantizerType.neural`/`Image.fromBytes`。验证 `test/ugoira_test.dart` + 真机 Ugoira 播放与 GIF 导出。
- `flutter_secure_storage ^10.0.0` → `^11.0.0`：两处调用均为 `const FlutterSecureStorage()`，代码零改动；
  需 AGP 9.1.0 → 9.1.1（明确支持 API 37）、`compileSdk 37`、`androidx.core:core` 1.15.0 → 1.19.0
  （1.19.0 的 AAR 要求 compileSdk 37），并确认本机与 CI 有 `platforms;android-37`。验证 `flutter build apk --debug`
  与真机登录凭据、翻译凭据读写。项目未发布，无 v10 数据迁移问题。

### R4. 版本三元组与工具链固化

- `tool/frb_check.sh`：比对 `pubspec.lock` 的 `flutter_rust_bridge`、`plugins/rhttp/rhttp/lib/src/rust/frb_generated.dart`
  的 `codegenVersion`、`rust/Cargo.toml` pin、`flutter_rust_bridge_codegen --version`；不一致即失败；进 CI。
- 插件 `pubspec.yaml` 的 `flutter_rust_bridge: ^2.12.0` 收紧为与 Cargo 相同的精确版本，`flutter pub get` 后
  lock 回到 2.12.0，消除当前 Dart 2.13.0 / 生成代码 2.12.0 的漂移。`rhttp.dart:19` 的 `forceSameCodegenVersion: false` 保留（上游行为）。
- `plugins/rhttp/rhttp/rust/rust-toolchain.toml`（当前 stable 1.98）；CI `flutter pub get --enforce-lockfile`、`cargo test --locked`。
- 以上差异记入 `plugins/rhttp/UPSTREAM.md`（parent D-5 已允许构建配置类差异）。

### R5. CI 覆盖

- 新增 job：`plugin`（`plugins/rhttp/rhttp` `flutter test`；`cargo fmt --check` + `cargo test --locked`；`frb_check.sh`）、
  `android-unit`（`./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest`）、
  `deps-report`（`flutter pub outdated`、`cargo update --dry-run` 归档 artifact，不阻塞）。
- `dart format --set-exit-if-changed` **不在本 child 加入**（E 负责一次性格式化后再加，避免与 09-01 未提交文件冲突）。

### R6. 不做

- `cached_network_image` 4.x、`go_router` 18 等 `material_ui` 系升级（parent D-11：在 child F 内随 app 迁移一起做）；
  AGP 9.3 / Gradle 9.7（超出 Flutter 验证矩阵）；rhttp dev 工具链（ffigen/freezed/build_runner）与上游走。原因记入 spec 的升级策略。

## Acceptance Criteria

- [ ] `flutter pub outdated` 直接依赖中无可解析的落后项（`go_router`/`cached_network_image` 的 material_ui 系大版本除外，留给 F）。
- [ ] APK 中无 `cupertino_icons` 字体。
- [ ] 本地与 CI 的 Flutter 为 3.47.2；`ci.yml`/`release.yml` 使用 `checkout@v6`、`setup-java@v5`。
- [ ] `tool/frb_check.sh` 通过并在 CI 执行；`pubspec.lock` 的 `flutter_rust_bridge` 与 Cargo pin 一致。
- [ ] `rust-toolchain.toml` 存在；CI 使用 `--enforce-lockfile` 与 `--locked`。
- [ ] CI 跑 Kotlin JVM 测试、插件 Dart 测试、`cargo test`，全部通过。
- [ ] `flutter analyze`、`flutter test`、双 flavor release 构建通过；Ugoira 与凭据读写真机验证通过。
- [ ] `UPSTREAM.md` 记录 toolchain/lock 差异；spec 增加升级策略段（跟随 Flutter 验证矩阵）。

## Notes

- 执行顺序与提交粒度见 parent `implement.md` "Child A"。
- 本 child 为轻量 task：PRD-only；不需要单独 `design.md`/`implement.md`。
