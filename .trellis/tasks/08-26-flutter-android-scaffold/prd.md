# 建立现代 Flutter Android API 36 工程基线

## Goal

把现有 Pixiv Func Dart UI shell 变成可真实执行 Flutter 质量门禁和 Android debug 构建的工程基线，为后续 Replica 业务链路提供稳定的现代 Flutter/Android 宿主。

## Confirmed repository facts

- 基线分支为 `main`，当前 HEAD 为 `8a20fb0c7b2c843e497151742d2ccc89ef42f542`。
- 仓库已有 `lib/`、`pubspec.yaml`、`README.md` 和 `LICENSE`，现有 Dart shell 已包含启动引导、登录 shell、Home shell、Riverpod settings 和 `AppIcons` 映射。
- 当前仓库缺少 `android/`、`test/`、`integration_test/`、根目录 `.gitignore` 和 `analysis_options.yaml`。
- `pubspec.yaml` 已声明项目名 `pixiv_func` 和 Flutter `>=3.47.0`，但当前执行环境找不到 `flutter` 命令。
- Flutter 官方 `3.47.0` 源码模板对应 Android compile/target API 36、AGP 9.1.0 和 Gradle 9.3.1，并支持开启 built-in Kotlin。

## Requirements

### R1. 生成 Android 工程

- 使用 Flutter `3.47.0` stable 官方 app template 生成 Android 平台目录。
- 保留现有 `lib/` 代码和 `pubspec.yaml` 的现有应用依赖，不用模板示例覆盖已有 Dart shell。
- 采用 Kotlin、Kotlin DSL 和 Flutter Plugin DSL；不得引入 `apply from: flutter.gradle`、旧版 KGP 配置或 ABI split 配置。

### R2. 固定 Android 身份和 API 基线

- Android `namespace` 与 `applicationId` 必须统一为 `io.github.lopution.pixivfunc`。
- 应用显示名称保持为 `Pixiv Func`。
- compile SDK 与 target SDK 必须为 API 36；Java/Kotlin 编译目标保持现代模板要求。
- 使用 AGP 9 与 built-in Kotlin；不得为兼容旧实现重新启用应用级 Kotlin Gradle Plugin。

### R3. 补齐仓库质量基线

- 添加与 Flutter stable 模板兼容的根目录 `.gitignore` 和 `analysis_options.yaml`，并保留 Trellis 忽略规则。
- 添加至少一个真实的 Flutter widget/startup smoke test，测试应针对现有 shell，而不是空断言或模板占位测试。
- 在可用工具链上运行 `flutter pub get`、`flutter analyze`、`flutter test` 和 `flutter build apk --debug`。

### R4. 保持任务边界

- 本任务只处理工程基线和构建门禁，不实现 OAuth、secure storage、网络层、推荐流、详情页或收藏链路。
- `assets/icon.ttf` 迁入和 `iconFont` 注册作为下一项独立工作；本任务不得用 Material fallback 替换现有 `AppIcons` 映射。
- 不修改与本任务无关的用户改动，不提交或推送远程分支。

## Acceptance Criteria

- [ ] 仓库包含由 Flutter `3.47.0` stable 官方模板生成且可审查的 `android/` 工程。
- [ ] `android/` 使用 `settings.gradle.kts`、现代 Flutter plugin DSL、AGP 9、built-in Kotlin，并且搜索不到 `apply from:.*flutter.gradle`、应用级 KGP plugin 或 ABI split 配置。
- [ ] `namespace`、`applicationId` 均为 `io.github.lopution.pixivfunc`，Android label 为 `Pixiv Func`。
- [ ] Android compile SDK/target SDK 均解析为 API 36。
- [ ] 根目录存在经审查的 `.gitignore`、`analysis_options.yaml` 和非占位 Flutter 测试。
- [ ] `flutter pub get` 成功。
- [ ] `flutter analyze` 成功且无新增 analyzer error/warning。
- [ ] `flutter test` 成功。
- [ ] `flutter build apk --debug` 成功；若环境工具链仍不完整，必须报告实际失败命令和错误，不得标记为通过。
- [ ] 现有 `lib/` UI shell 的可见行为未因 scaffold 生成而被无关改写，`git diff` 仅包含本任务范围内文件。

## Risks and deferred items

- 当前没有 Flutter SDK，也没有可发现的 `sdkmanager`；实现阶段必须先准备或定位 Flutter 3.47.0、Java 17 和 Android API 36 工具链。
- 当前 `LICENSE` 文件头实际显示 GPL v3，而项目说明要求 AGPL-3.0-only；许可证修正需要单独审查，不在本 scaffold 任务中默默处理。
- Flutter 模板的具体插件版本和 built-in Kotlin 属性以 `3.47.0` 生成结果及官方迁移规则为准，不手写旧版 Gradle。

## Open questions

无阻塞性的产品决策；任务启动仍须经过本规划摘要的独立确认。
