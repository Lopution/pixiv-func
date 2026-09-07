# 全项目重构范围盘点

日期：2026-09-02

> **已被取代（2026-09-07）**：本文件是最初的静态盘点，其"不做体积基线"的前提已被推翻。
> 现行事实来源为同目录下五份研究文件：`apk-size-breakdown.md`（体积基线与归因）、
> `dependency-upgrade-audit.md`（依赖与工具链）、`dart-architecture-audit.md`（Dart 分层/重复/
> 可替换性/测试）、`native-and-build-audit.md`（Android/Gradle/CI/rhttp）、`modernization-gaps.md`
> （现代化缺口与死代码）。本文件仅保留作历史记录，不再更新。

这份记录是静态结构盘点，不是性能或体积基线。它用于决定后续 design 的边界，不能单独证明
某项改动带来运行时加速或固定的包体积下降。

## 仓库形态

| 区域 | 当前观测 |
|---|---|
| app Dart 源码 | `lib/` 约 185 个受版本控制文件 |
| app 测试 | `test/` 约 65 个受版本控制文件 |
| Android | 原生 Kotlin/资源/Gradle/flavor 与 JVM widget tests |
| Rust/FFI | `plugins/rhttp/rhttp` vendored 插件，含 Rust crate、cargokit 和 Flutter Rust Bridge 生成物 |
| Flutter 依赖 | `pubspec.yaml` 同时包含 Riverpod、网络/缓存、数据库、图片/Ugoira、WebView、rhttp 等依赖 |
| 资源 | `assets/emojis/`、`assets/stamps/`、`assets/icon.ttf`；当前 assets 目录约 384 KB |
| 发布 flavor | `github` 与 `fdroid`，发布签名/updater 语义由 09-01 release-blockers 约束 |

仓库内存在被 `.gitignore` 排除的构建目录、Gradle 缓存和 Rust `target/`。它们不能作为源码
体积或可维护性问题直接提交、删除或计入产品产物分析。

## 跨层边界

```text
Flutter 页面
  -> Riverpod controller/provider
  -> core repository/service/store
  -> http/rhttp、sqflite、缓存、下载与 Ugoira
  -> Dart platform channel / Rust FFI
  -> Android 原生组件、MediaStore、WebView、App Widget

assets + pubspec + Rust features + Gradle/flavor
  -> Flutter/Gradle/Cargokit 构建
  -> APK/AAB 与 native libraries
```

以下边界在重构时必须保持可追踪：

1. provider/controller 与 core service/repository 的状态所有权和取消语义；
2. Dart 与 Rust FFI 的 API、错误类型、生成版本；
3. Dart 与 Android MethodChannel 的方法名、payload、线程和 API 29 降级；
4. Flutter assets/dependencies 与 flavor/ABI 产物的映射；
5. Rust/FFI 源定义与生成 Dart/Kotlin/native 输出的一致性。

## 候选维护面（仅为调查线索）

以下文件较大或处于共享路径，值得在 design 阶段逐项判断，但不能仅凭行数直接重写：

- `lib/core/network/compat/network_policy.dart`
- `lib/core/download/download_manager.dart`
- `lib/core/ugoira/ugoira_zip.dart`、`ugoira_export.dart`
- `lib/features/settings/settings_page.dart`
- `lib/features/illust/detail/illust_detail_page.dart`
- `lib/features/profile/user_page.dart`、`profile_header_delegate.dart`
- `lib/core/paging/paged_feed_controller.dart`
- `lib/core/widget/widget_coordinator.dart` 与 Android widget 实现
- `android/app/src/main/kotlin/io/github/lopution/pixivfunc/` 下的 platform channels
- `plugins/rhttp/rhttp/rust/src/api/`、`rust/Cargo.toml`、cargokit 配置和其生成输出

每个候选必须先回答：它消除了哪一种重复工作或维护负担、有哪些公开消费者、如何保持错误/
取消/生命周期契约、以及如何独立回滚。

## 体积与构建候选面

- Flutter assets 与字体：先查所有 `AssetImage`、字体注册和测试 fixture 的消费者。
- Dart 依赖：以 `pubspec.yaml` 为入口，结合全仓库 import 和平台插件注册确认，不能只按包名
  搜索字符串判断未使用。
- Android dependencies/resources：核对 Gradle 依赖、Manifest、flavor source sets 与 API 29
  行为；不要为了体积移除 WorkManager、MediaStore 或 updater 所需能力。
- Rust：核对 `Cargo.toml` feature、`[profile.release]`、目标 ABI 和 cargokit 产物；保留
  TLS、压缩、multipart、stream 等真实消费者需要的能力。
- FFI 生成物：只能通过源 API/配置重新生成，必须保留上游版本与 `plugins/rhttp/UPSTREAM.md`
  的同步记录。

## 已知重叠与硬边界

- `09-01-settings-productization` 先决定质量、下载 destination、过滤消费者；本 task 不能按旧
  destination 或旧设置模型提前重构。
- `09-01-behavior-correctness-cleanup` 负责 `credentialRevision`、download recovery、tab
  eager 构建、深链和 `fetchPage` 契约；本 task 只能消费其完成后的稳定结果。
- `09-01-network-perf-ab` 负责网络测量及有数据支撑的路线调整；本 task 不借重构名义改变网络
  协议、SNI/ECH、证书校验或 fallback 顺序。
- `09-01-release-blockers` 负责正式 APK 签名、updater 验签和 GitHub CDN；本 task 若触及
  构建配置，必须保持其密钥与发布契约，并在最后重建验证。
- 已归档 `08-29-cn-direct-transport` 的 Rust/rhttp/ECH 方案是当前传输语义来源；可以治理其
  源码、生成流程和构建成本，但不能静默换成另一种协议方案。

## 计划结论

1. 这是一个跨层复杂 task，必须有 `prd.md`、`design.md` 和 `implement.md`。
2. 采用“契约盘点 → 可维护性边界 → 运行效率 → 产物/构建 → 全链路验证”的阶段顺序，每阶段
   单独提交和验证。
3. 不安排前置性能/体积基线；任何数值化收益以后另开测量或由用户明确增加验收指标。
