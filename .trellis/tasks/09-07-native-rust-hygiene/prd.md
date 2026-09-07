# 原生层与 Rust 插件整理（09-02 child D）

## Goal

让 Android 原生层的 MethodChannel 有一套被文档与测试约束的参数/错误/线程约定，清理 minSdk 29 下的死分支与
模板残留，去掉两份 updater channel 的逐字重复，并把 vendored rhttp 的 fork 差异、再生成流程与工具链写成可复现文档。
不改 channel 名、方法名与 Dart 侧语义。对应 parent R3（清理部分）、R4（原生契约）、R6；事实来源
`research/native-and-build-audit.md` A.1–A.7、B.1、B.6–B.7；目标形态见 parent `design.md` §5。已确认决策 D-5。

## 开工 gate

- `09-01-settings-productization`（`android/` 有其未提交改动：`MainActivity`、`MediaStoreChannel`、`WidgetForegroundChannel`、
  已删除的 `LoginWebViewPlatformView.kt` 等）与 `09-01-release-blockers`（updater Kotlin）归档。
- 开工第一步：对当时 HEAD 重跑 A.1–A.3 清单（channel 表、`SDK_INT`、`!!`、错误风格）。

## Requirements

### R1. 契约文档先行

- `.trellis/spec/backend/android-channels.md`：10 个 channel 的方法、参数、返回、错误码、线程模型；`widget_snapshot/active.json`
  文件契约；`pixivfunc/widget_background` 方向反转说明。先写现状，再改代码，改完更新为目标状态。

### R2. 参数与错误（design §5.1）

- `call.argument<T>()!!` 全部改为显式 `result.error("invalid_argument", "<name> missing", null)`。
- 三种错误风格统一为细分错误码 `<channel>_<reason>` + `result.error`；updater 的 `{valid:false, errorCode}` Map 形态保留并在文档标注为例外。
- Kotlin JVM 单测覆盖错误码路径（缺参、非法值、平台异常）。

### R3. 线程

- MediaStore 写入、SAF 写入、反查图片复制、updater APK 解析四处的同步 IO 移出主线程（`BinaryMessenger.TaskQueue` 或 IO 协程），
  `result` 回主线程。真机验证下载、SAF 写入、反查、自更新。

### R4. 死分支与残留

- 删除 12 处 `SDK_INT < Q/P/O` 恒真分支，只保留 3 处 `TIRAMISU`。
- 删除 `drawable-v21/launch_background.xml` 重复；删除 debug/profile manifest 冗余 `INTERNET`。
- `http://pixiv.net` 深链 filter 保留（改变接管范围属产品决策，不在本 child）。
- `login_webview_intercept` 的 Dart 单侧残留在 09-01 确认删除原生端后一并清理。

### R5. updater 去重

- 两份 `DistributionUpdaterChannel.kt` 的 `platformInfo/packageInfo/signerSha256` 抽到 main source set
  `updater/UpdaterPlatformInfo.kt`；flavor 文件只保留差异逻辑。

### R6. rhttp fork 文档（D-5）

- `plugins/rhttp/UPSTREAM.md` 增加"构建配置差异"一节：`Cargo.toml` features/profile（随 child B 变更）、`android/build.gradle.kts`
  AGP 9 适配（已存在未记录）、cargokit 清理、`rust-toolchain.toml`、`rust/tests/ech_*`、`rust/examples/ech_reqwest_probe.rs`。
- 同步指引扩为七步（diff 上游 → 重打 ECH → 重打构建配置差异 → `frb_check.sh` → 必要时 codegen → 双侧测试 → 更新记录）。
- `.trellis/spec/backend/rust-plugin.md`：fork 策略、FRB 三元组、再生成命令、cargokit ABI/输出目录行为、插件 AGP/Kotlin 版本与 app 的差异。

### R7. 不做

- 不改 channel 名、方法名、payload 字段；不改 updater 验签算法与 manifest 字节语义；不改 Rust 源码逻辑；不引入新 channel 框架。

## Acceptance Criteria

- [ ] `rg '!!' android/app/src/main/kotlin` 中 `call.argument` 相关为 0；缺参路径有单测。
- [ ] 所有 `result.error` 码符合 `<channel>_<reason>`；文档与代码一致。
- [ ] 四处 IO 不在主线程执行（代码 + 真机验证下载/SAF/反查/自更新）。
- [ ] `rg "SDK_INT" android/app/src` 只剩 `TIRAMISU` 3 处；`drawable-v21` 与冗余 `INTERNET` 已删。
- [ ] 两份 `DistributionUpdaterChannel.kt` 无重复函数；Kotlin 测试通过。
- [ ] `UPSTREAM.md` 覆盖全部 fork 差异与七步同步；`rust-plugin.md`、`android-channels.md` 落地。
- [ ] `flutter analyze`、`flutter test`、Kotlin 测试、双 flavor 构建通过；API 29 真机四项功能各一次。

## Notes

- 中等复杂度：`task.py start` 前从 parent design §5 与 implement "Child D" 派生本目录 `implement.md`（D0–D6）；`design.md` 可省略，
  以 parent design §5 为设计依据。
- 停止条件：channel 超时、线程阻塞或权限行为变化 → 回滚该提交。
