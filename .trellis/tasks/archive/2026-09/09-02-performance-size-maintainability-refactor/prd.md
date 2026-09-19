# 全项目重构：可维护性、可迭代性、简洁性与现代化收口

## Goal

在 Func 1.0 的功能 child 收口后，对整个项目做一次有边界、可回滚、可量化的工程重构。
用户在 2026-09-07 给出的四个目标，以及它们在本仓库中的具体含义：

| 目标 | 在本仓库中的含义 |
|---|---|
| **可维护性** | 架构便于人和 LLM 后续维护：分层与依赖方向有明文规则并被工具校验；每个跨层契约只有一个 owner；文件规模可被一次性读完；`.trellis/spec/` 描述的是真实约定而不是模板 |
| **可迭代性** | 组件（HTTP 传输、图片缓存、DB、安全存储、UI 依赖、Rust 插件、Android 工具链）能低成本升级或替换：依赖健康有常规检查；替换点单一；版本三元组（如 flutter_rust_bridge）有脚本校验 |
| **简洁性** | 发布包体积从 88.6 MB 收敛到单 ABI ≤ 32 MB 先落地、≤ 28 MB 为收敛目标；代码用最小、最优实现：删除零引用代码与依赖，合并重复实现，不新增框架 |
| **升级项目** | 完成从 beta56 原版移植后不彻底的现代化：i18n、状态范式、分页、日志、lint 严格度、格式化、CI 覆盖、工具链版本，以及依赖升级 |

## 2026-09-07 相对 09-02 原稿的决策变更

| 原稿（2026-09-02） | 现在 | 原因 |
|---|---|---|
| 不做前置性能/体积基线，不写 MB 目标 | **已测量体积基线并设定 MB 目标**（`research/apk-size-breakdown.md`） | 用户明确"80 多 MB 偏大"；88.6 MB 中 85.9 MB 是三套 ABI 的 native 库，问题成因清晰，可量化 |
| 单一 task、五个 Phase、一个全局 gate（等 09-01 全部归档） | **拆为 parent + 5 个 child，每个 child 独立 gate**（见任务地图） | 依赖健康、体积流水线与 09-01 的 Dart 功能改动几乎不重叠，没有理由等；Dart 架构收敛才需要等 09-01 落地 |
| 组件升级不在范围 | **组件升级并入**（`research/dependency-upgrade-audit.md`） | 用户要求 |
| 运行效率作为独立目标 | **不再作为独立目标**；仍禁止无证据的缓存/预取/并发 | 审计未发现可证明的重复运行工作热点；用户四个目标里没有性能 |

保留不变的原则：不新增 architecture-v2 类基础设施；重构不改变协议/安全/下载恢复语义；生成代码
只由源与配置再生；不用删测试、关安全校验、去 API 29、删 flavor 来换体积。

## Background And Evidence

五份研究文件是本 PRD 的事实来源，实现前必须对当时 HEAD 重新核对数字：

| 文件 | 结论 |
|---|---|
| `research/apk-size-breakdown.md` | fat APK 88.6 MB = x86_64 31.8 + arm64 29.0 + armeabi-v7a 25.1 + 其它 <3 MB。单 arm64 ≈ 31.5 MB（libflutter 11.75 不可动 / libapp 9.96 小幅可动 / librhttp 5.44 可裁 / libsqlite3 1.73 纯死重）。`--target-platform` 不能当单 ABI 打包用；cargokit 陈旧 ABI 产物会混入构建 |
| `research/dependency-upgrade-audit.md` | `go_router`、`cupertino_icons` 零引用；Flutter 3.47.2 hotfix 含安全修复；`archive`+`image` 耦合升级；`flutter_secure_storage` 11 需 AGP 9.1.1；FRB Dart 2.13.0 / 生成代码与 Cargo 2.12.0 三方漂移；`cached_network_image` 4 与 `material_ui` 绑定，暂不动 |
| `research/dart-architecture-audit.md` | 200 文件 / 51.8k 行；89 个手写 provider，分页范式统一但 3 个 ChangeNotifier 游离；6 个页面直接碰平台/HTTP；`core→features` 反向依赖 1 处；`IllustCard` 定义在推荐页导致 12 个页面文件成环；主机 allowlist 5 处重复声明；17 个等价 i18n 包装；32 个私有 feed 尾部/空态/错误 widget；49 个互不相关 Exception；三套 HTTP 栈；测试假实现在 17 个文件重复；spec 索引过期、5 份模板未填 |
| `research/native-and-build-audit.md` | 10 个 MethodChannel 三种错误风格、全部主线程同步 IO；12 处 minSdk 29 下恒真的 `SDK_INT` 分支；两份 `DistributionUpdaterChannel` 三函数逐字重复；rhttp 六个 reqwest feature 应用零使用；updater manifest 单 asset 结构；CI 不跑 Kotlin/插件/Rust 测试，无格式检查、无体积门禁 |
| `research/modernization-gaps.md` | 已废弃 Flutter API 0 命中、`flutter analyze` 0 问题；缺口在结构：i18n 字典 1924 行 + 27 个包装 + 缺 key 运行时抛错；history 页手写分页绕过 `PagedFeedController`；4 个零引用文件、13 个零引用声明、142 个可私有化声明；46 个文件 `dart format` 漂移；图标字体 14 个 glyph 仅用 6 个；`useMaterial3: false` 为有意冻结视觉 |

`research/refactor-inventory.md`（2026-09-02）为早期盘点，已被以上文件取代，只保留作历史。

## 已确认的决策（用户 2026-09-07 答复）

| # | 决策 | 答复 | 后果 |
|---|---|---|---|
| D-1 | 发布形态 | **只发布 per-ABI 拆分 APK：arm64-v8a + armeabi-v7a**；不发布 universal，不发布 x86_64（模拟器用 debug 包） | child B 流水线；armeabi-v7a 是否继续发布待有下载数据后再议 |
| D-2 | versionCode | **保留 Flutter 的 `1000 × ABI` 偏移**；updater 按 `versionName` semver + 去偏移 versionCode 比较 | 不使用 `-Pforce-version-code-ignoring-abi`；F-Droid 多 APK 友好 |
| D-3 | updater 多资产 schema 归属 | **现在写进 `09-01-release-blockers` 的 PRD**，由其按多资产实现 | release-blockers PRD 已追加 R5（2026-09-07）；child B 只做消费端与流水线 |
| D-4 | 目标平台 | **保留桌面分支**（`history_database.dart` 的 FFI 路径与 `sqflite_common_ffi` 主依赖不动） | `libsqlite3.so` 不能靠移依赖去掉；改为 Android 侧 Gradle `packaging.jniLibs.excludes` 排除（待构建验证，见 design §4.3），并以测试保证 Android 不触达 FFI 分支。若排除不可行则接受 1.73 MB/ABI，收敛目标放宽到 ≤ 30 MB |
| D-5 | rhttp fork 策略 | **允许第二类差异**：`Cargo.toml` feature/profile、Gradle AGP 适配、cargokit 清理、toolchain 固化；不改 Rust 逻辑；逐项记入 `UPSTREAM.md` | child B 的 rhttp 裁剪与 child D 的文档 |
| D-6 | i18n 形态 | **迁到 Flutter 内置 `gen-l10n` + ARB** | child C；design §3 方案 (a) |
| D-7 | lint 严格度 | **分两批开启** `strict-casts/strict-raw-types/strict-inference`、`unreachable_from_main`、`prefer_final_locals`、`unawaited_futures`、`avoid_dynamic_calls`、`sort_pub_dependencies`；不开 `public_member_api_docs`、`lines_longer_than_80_chars` | child E |
| D-8 | 任务结构 | **转为 parent，创建 5 个 child** | 已按任务地图创建（见 Notes） |

2026-09-07 GPT 两轮评审后新增，用户已答复：

| # | 决策 | 答复 | 后果 |
|---|---|---|---|
| D-9 | 运行时性能基线 | **不测** | 性能只作为静态可验证的设计约束（design §14），不做数值验收，不声称"更快/更省"；唯一数值门禁是 B 的 APK 体积 |
| D-10 | 现代交互与视觉项 | **不延后，在本 task 内推进** | 新建 child F 承接（D-13）；README "第一阶段冻结用户可感知体验" 原则在 F 阶段终止 |
| D-11 | Material 3 落地路径 | **整个 app 迁到 `material_ui` + `cupertino_ui`，`useMaterial3` 取默认** | `go_router` 18、`cached_network_image` 4 随之在 F 升级；legacy 插件经 `MaterialUiCompatibilityBridge`；F 结束重测体积并更新 B 阈值 |
| D-12 | 导航架构 | **`go_router` `StatefulShellRoute` + `restorationScopeId`** | A 不再删除 `go_router`（当前零引用，F 升到 18 并接线）；C 的 `routes.dart` 门面以 id 为形参，F 用 go_router 重实现 |
| D-13 | 交互/视觉现代化归属 | **新建 child F，排在 C 之后、E 之前** | C 保持行为中性；F 是唯一改变可见行为的 child |

明确不在本轮决策、留待后续独立 task：网络路线/ECH/SNI/证书策略。

## 两条横跨 child 的设计轴（2026-09-07 补入）

GPT 对推送后 HEAD 的两轮评审，经代码核实后吸收为两条设计轴。它们不是新 child，而是 A–E 的验收维度：

1. **UI 组件体系**：`lib/app/` 明确成为组件层（theme tokens / motion / navigation 门面 / feed 组件 /
   图片 / 反馈 / settings 原语），`features/` 只做组合。事实：8 处瀑布流写死 2 列；Hero+`PixivImage`
   组合 3 处；`Semantics` 0 处、14 处 `GestureDetector` 无语义；两份路由实现；32 个私有 feed 状态 widget。
   详见 design §13。
2. **性能约束**：不作为独立目标，也不做基线（D-9）；对图片 decode 尺寸、重建范围、启动、持久化增长、
   通道开销设**静态可验证**的约束。事实：`PixivImage` 无 decode 尺寸限制；启动已是"首帧前只等
   settings + 账号"；下载恢复表整表重写。详见 design §14。

## Requirements

### R1. 任务边界与 gate

- 每个 child 有自己的开工 gate（见任务地图），不再用"09-01 全部归档"一刀切。
- 任何 child 开工前必须 `git status` 记录并保留不属于本 task 的用户改动；不得借重构覆盖它们。
- 候选改动若会改变用户可见行为、API/OAuth/TLS 语义、下载恢复协议、数据迁移格式或 flavor 发布契约，
  必须另开 task 或回到用户确认。D-1～D-3 是本轮已知的唯一契约变更，且由 D-3 交给
  release-blockers 定稿。
- 属于 09-01 child 的进行中改动（例如 `login_intercept_controller.dart` 与其原生端的去留由
  `09-01-settings-productization` D3 决定）不由本 task 处理，只在其归档后清理残留。

### R2. 可迭代性：依赖与工具链健康

- 删除零引用依赖 `cupertino_icons`；`go_router`（当前零引用）保留，F 升到 18 并接线（D-12）。
- 按 `research/dependency-upgrade-audit.md` 执行：Flutter 3.47.2（本地 + CI 三处）、
  `androidx.core` 1.19.0、`work-runtime-ktx` → `work-runtime`、`flutter pub upgrade`、
  `cargo update`、`actions/checkout@v6`、`actions/setup-java@v5`、Kotlin 2.4.10（可选）；
  耦合升级 `archive` 4 + `image` 4.9；`flutter_secure_storage` 11 + AGP 9.1.1 + `platforms;android-37`。
- 建立 flutter_rust_bridge 三元组校验脚本（`pubspec.lock` Dart 运行时 / `frb_generated.dart`
  `codegenVersion` / `Cargo.toml` pin），CI 执行；rhttp 的 `pubspec.yaml` 把 `flutter_rust_bridge`
  约束收紧到与 Cargo 一致，消除"能解析到 2.13.0"的漂移来源。
- 固化工具链：`rust-toolchain.toml`；CI `flutter pub get --enforce-lockfile`、`cargo --locked`。
- CI 常规检查：`flutter pub outdated`、`cargo update --dry-run` 输出归档（不阻塞）；Kotlin JVM
  测试、`plugins/rhttp/rhttp` 的 `flutter test`、`cargo test` 纳入 CI。
- 升级策略写入 spec：跟随 Flutter stable 的官方 Android 验证矩阵（当前 JDK 17 / KGP 2.4.0 /
  AGP 9.1.x / Gradle 9.3.1），不单独追 AGP/Gradle 新版；`material_ui` 系依赖（`go_router` 18、`cached_network_image` 4）
  在 F 内随 app 迁移一起升级。

### R3. 简洁性：发布体积与最小实现

- 体积（实现阶段每完成一项重新测量并更新 `research/apk-size-breakdown.md` §6）：
  - 用户下载包：从 88.6 MB（fat）→ per-ABI arm64 ≤ 32 MB（仅靠拆分，已测量构成）→ ≤ 28 MB
    （rhttp feature/profile 裁剪、`--obfuscate --split-debug-info`、`cupertino_icons` 移除、
    Android 侧排除 `libsqlite3.so` 全部落地后的推算值，不是承诺；若 D-4 下的排除不可行则为 ≤ 30 MB）。
  - release 产物不含 x86_64；不发布 universal APK。
  - CI 输出每个 split APK 的总大小与 `lib/<abi>` 明细，超过阈值失败；阈值在拆分落地后设定。
  - cargokit 构建前清理非目标 ABI 的陈旧输出，避免本地构建污染。
- 最小实现（删除与合并，不新增抽象）：
  - 删除 4 个零引用文件、13 个零引用公共声明、8 个未使用的 `AppIcons` 常量（放宽
    `icon_font_test`）；142 个仅本文件使用的公共声明按需私有化。
  - 合并重复实现：两份右滑路由 → `ReplicaPageRoute`；27 个 i18n 包装 → 一个入口；44 处手写
    SnackBar → 一个 helper；32 个私有 feed 尾部/空态/错误 widget → 3 个共享组件；JSON 读取
    helper（`_map`/`_firstString`/`_positiveInt`/`_optionalString`/`_nextUrl` 各 2–5 份）→ 一处；
    两份 `DistributionUpdaterChannel` 的三个重复函数 → main source set；两个同名 `MePage` 只留一个。
  - `history_page` 改用 `PagedFeedController`，删除手写 `ScrollController` 分页。
  - 清理 minSdk 29 下的 12 处死 `SDK_INT` 分支、`drawable-v21` 重复、debug/profile manifest 冗余
    `INTERNET`。
- 删除任何资源、依赖、feature、ABI 前必须完成全仓库消费者搜索、许可证核对与双 flavor 构建。

### R4. 可维护性：分层、单一 owner、可发现性

- 分层规则写入 `.trellis/spec/frontend/directory-structure.md` 并由测试/脚本校验：
  `lib/core/**` 不得 import `lib/features/**`；`lib/features/**` 之间只经 `lib/app/navigation/`
  的路由门面互相到达；共享 UI（`IllustCard`、瀑布流 sliver、feed 尾部/空态/错误）放 `lib/app/`；
  repository / controller / entity 全部在 `lib/core/<domain>/`，`features/` 只放 widget。
  现有违反项（1 处 `core→features`、5 个 features 内的 repository、`app ↔ onboarding` 互引、
  12 文件页面环）在 child C 中修正。
- 单一 owner：Pixiv 主机与 header 只由 `PixivClientIdentity` 派生（`PixivDestinationRegistry`、
  预热列表、fast-route 引导、探测页、登录拦截从它枚举）；`SharedPreferencesAsync` 经一个
  provider 注入（消除 7 处直接构造）；设置键命名空间统一并集中声明；`http.Client`/`HttpClient`
  在生产代码中不得内联构造，一律经 provider 注入。
- 状态范式：3 个 `ChangeNotifier` 控制器并入 Riverpod；页面不再 `new MethodChannel*` 适配器。
- 文件规模：拆分 `settings_page.dart`（14 个页面类）、`illust_detail_page.dart`、`user_page.dart`、
  `network_policy.dart`（策略与 `PixivNetworkFactory`/图片缓存所有权分离）；>150 行的 5 个方法拆分。
- 异常体系只做文档与命名整理（`backend/error-handling.md` 写真实分类与传播规则），不改 `catch` 行为。
- LLM 可发现性：`.trellis/spec/` 两份 `index.md` 状态列修正；填写 `directory-structure`×2、
  `error-handling`、`database-guidelines`、`type-safety`、`logging-guidelines`；新增 Android channel
  契约表、Rust/FRB 再生成流程、cargokit ABI 行为、updater manifest 字段的专章；`lib/core/<domain>/`
  每个目录一段 library 级文档说明职责与 owner。
- 测试：共享假实现进 `test/helpers/`（凭据/元数据/账号 store、prefs 初始化）；去除以私有类名字符串
  定位的断言；删除自述"one-off"的诊断测试；`.gitignore` 加 `test/failures/`。

### R5. 升级项目：完成现代化

- i18n 按 D-6 落地；无论选项，都要有"四语言 key 集合 = 代码引用集合"的自动校验，修复
  `networkDohEndpointsInvalid` 未定义与 ru 缺 `detailQuality`。
- `dart format lib test` 一次性格式化并纳入 CI；lint 按 D-7 分两批开启并清零。
- 日志出口统一（极小 `log()` 或集中 `debugPrint`），补 `logging-guidelines.md`。
- 硬编码颜色收敛到 `FuncTokens`/主题（保留 replica 视觉决定的 `Colors.*`）。
- 图标：删除未用 glyph 常量；`icon.ttf` 子集化为可选项（需更新 `beta56-icon-font.md` 记录）。
- Material 3、`material_ui`/`cupertino_ui`、`go_router` 架构、Predictive Back、状态恢复、Hero 手势、M3 组件由 F 交付（R10）。
- 显式不做：json 代码生成、Result/Either 类型、`material_ui`/`go_router` 之外的 UI/路由库。

### R6. 兼容性、安全与许可

- 保持 Android API 29 基线、`github`/`fdroid` flavor 语义；平台能力差异显式降级。
- 保持 API/OAuth/TLS 真实服务器身份验证、凭据安全存储、下载安全校验、反向搜图/翻译隐私边界；
  不把调试开关或不安全 fallback 扩展成默认路径。ECH/aws-lc-rs 不因体积被替换。
- rhttp 的 `Cargo.toml`/Gradle 差异按 D-5 记录进 `UPSTREAM.md`；依赖替换或 vendored 修改同步更新
  NOTICE 与许可证清单；`cargo update` 后核对新引入 crate 的许可证。
- `--obfuscate` 后符号文件必须随 release 归档，否则崩溃栈不可读。

### R7. 验证与交付

- 每个 child：`flutter analyze`、`dart format --set-exit-if-changed`、`flutter test`；触及处追加
  Kotlin JVM 测试、插件 `flutter test`、`cargo fmt --check` + `cargo test`；`github` 与 `fdroid`
  release 构建（child B 起为 per-ABI）。
- 触及 FFI/生成链路时执行 codegen 并确认无未解释漂移；触及资源/依赖/native 时检查 APK 文件清单、
  签名与 manifest。
- 体积类改动必须附带"改动前/后"的 `unzip -l` 汇总，写入 `research/apk-size-breakdown.md`。
- 每个阶段提交前 `git diff --check`；最终由用户执行 parent 级真机矩阵（API 29 与高版本 Android、
  登录、浏览、下载、Ugoira、widget、updater 跨 ABI 自更新、大陆网络）。

### R8. UI 组件体系（设计轴一，主要落在 C，E 校验）

- `lib/app/` 按 design §13.1 组织：`theme/`、`motion/`、`navigation/`、`widgets/feed/`、`widgets/pixiv_image.dart`、
  `widgets/feedback.dart`、settings 原语。`features/` 不得再定义与组件层同职责的私有 widget（layering_test 按名称模式检查）。
- `IllustFeedGrid` 统一 8 处瀑布流，列数由宽度计算（手机恒为 2 列，与现状一致）。
- `PixivImage` 增加 decode 策略与 `feed/detail/viewer/avatar/hero` 变体构造器；Hero 组合与 `HeroRectClip` 进 `motion/`；
  转场契约不变。
- `motion_tokens.dart` 是所有时长/曲线的单一来源。
- 共享组件与收藏/关注/下载/导航四类动作满足无障碍基线：语义标签或 tooltip、≥ 48dp 触达、可聚焦可键盘激活；
  14 处 `GestureDetector` 逐个评估。
- 新组件必须有 ≥ 3 处现存重复作为证据（`NovelCard`/`UserCard` 先盘点）。
- C 只交付 enabler（`motion/` 单一来源、以 id 为形参的 `routes.dart` 门面、组件层收敛）；交互与视觉变化在 F（R10）。

### R9. 性能约束（设计轴二，静态约束，D-9 不做基线）

- feed/avatar 图片变体必须带按布局尺寸计算的 `memCacheWidth`，detail 按屏宽，viewer 不限；widget 测试断言。
- 共享组件只 `watch` 所需 provider 切片；`IllustCard` 不 `watch` 整个 feed state。
- 首帧前等待项固定为 settings + 账号；新增初始化 `unawaited`/后置；`startup_gate_test` 断言。
- `DownloadRecoveryStore` 记录数设上限或改增量写入（code review 决定）；history 查询计划在单测中断言不 `SCAN TABLE`。
- MediaStore/SAF 写块 ≥ 256 KiB。
- 不建请求缓存层；不做数值化性能结论；B 的 APK 体积门禁是唯一数值门禁，F 迁 material_ui 后重测更新。

### R10. 交互与视觉现代化（child F，D-10～D-13）

- `material_ui` + `cupertino_ui` 迁移（`dart fix --code=migrate_design_widgets`、localizations delegates、
  `MaterialUiCompatibilityBridge` 包 legacy 插件）；`go_router` 18、`cached_network_image` 4 随之升级；迁移后重测 per-ABI 体积。
- M3 主题：`ColorScheme.fromSeed` 以 `FuncTokens` 品牌色为 seed，组件主题单点定义；golden 重新生成。
- `go_router` `StatefulShellRoute.indexedStack`：首页各 tab 独立栈；页面为子路由；转场经 `motion_tokens`；`pixiv://`/web 深链映射；
  C 的门面改为 go_router 实现，调用点不变。
- 状态恢复：`restorationScopeId`；tab/栈/搜索词/viewer 页码可恢复；feed 滚动位置经 `PageStorageKey`；
  用"不保留活动" + `adb shell am kill` 验证。
- Predictive Back：manifest 开启，`PopScope` 用现代 API，Android 14+ 真机验证；API 29 自然降级。
- Hero 手势返回：viewer 拖拽关闭驱动 Hero 反向转场（`motion/` 原语）。
- M3 组件：`NavigationBar`、`SearchBar`/`SearchAnchor`、`SegmentedButton`（三档枚举）。
- README 项目原则改写（视觉冻结终止）；`component-guidelines.md` 记录新交互契约。
- 不做：`material_ui`/`go_router` 之外的 UI 库；平板专用布局（自适应列数由 C 的 `IllustFeedGrid` 提供即可）。

## 任务地图（child 已于 2026-09-07 创建，均 `planning`）

| Child | 交付物 | 开工 gate | 阻塞谁 |
|---|---|---|---|
| A `dependency-toolchain-health` | R2 全部；`cupertino_icons` 删除（`go_router` 保留给 F）；FRB 校验脚本；CI 覆盖 Kotlin/插件/Rust 测试 | 用户批准即可（与 09-01 仅 `pubspec.yaml` 级微冲突） | B（`cupertino_icons`、lock 刷新是 B 的基线） |
| B `release-size-per-abi` | per-ABI 发布流水线 + updater 多资产消费端 + Android 侧排除 `libsqlite3.so`（待验证）+ rhttp 裁剪/profile + obfuscate + cargokit 清理 + CI 体积门禁；记录图片 `CacheManager` 磁盘配置与缓存目录大小（R9） | D-1～D-5 已确认；`09-01-release-blockers` 的 R5（多资产 manifest）落地或与其并行协作；A 完成 | parent 真机 updater 验收 |
| C `dart-architecture-convergence` | R4 的分层修正与单一 owner、R3 的合并/删除、R5 的 i18n/状态范式/history 分页/颜色；**R8 组件层**（`lib/app/` 结构、`IllustFeedGrid`、`PixivImage` 变体与 decode 策略、`motion/`、settings 原语、共享组件无障碍基线）；**R9** 的图片 decode、`watch` 切片、持久化增长上限 | 所有触及 `lib/` 的 09-01 child 归档（settings-productization、behavior-correctness-cleanup、network-perf-ab、reverse-image-saucenao、comment-translation） | E |
| D `native-rust-hygiene` | channel 错误风格与线程约定、死分支、updater 去重、manifest/模板残留、`UPSTREAM.md` 补全、FRB 再生成流程文档；R9 的通道开销测量（MediaStore/SAF 分块）与原生 init 不阻塞首帧 | `09-01-settings-productization`（`android/` 改动）与 `09-01-release-blockers` 归档 | E |
| F `interaction-visual-modernization` | R10 全部：`material_ui`/`cupertino_ui` 迁移与 M3 主题、`go_router` 18 架构与深链、状态恢复、Predictive Back、Hero 手势、M3 组件、golden 与 README 更新、体积复测 | C 归档（组件层、`motion_tokens`、门面）；B 归档（体积门禁存在才能量化 material_ui 的影响） | E |
| E `spec-test-lint-hardening` | spec 填写与索引修正、模块级文档、共享测试假实现、断言整理、`dart format` + lint 两批启用；R8 共享组件 semantics 测试与 layering_test 的私有 widget 名称检查 | 文档部分可随 A 开始；测试/lint 部分在 C、D、F 之后 | parent 归档 |

## 跨 child 约束

1. **B 的 updater 多资产 schema 由 release-blockers 拥有。** 若 release-blockers 先实现单资产，B 必须
   做兼容迁移（schema 1→2 双读）；推荐 D-3 直接在 release-blockers 定稿多资产。
2. **C 的分层规则校验脚本先于大规模搬移提交**，否则搬移过程中无法判断是否引入新违反。
3. **E 的 lint 第二批（`strict-*`）在 C 之后**：C 会重写 JSON 读取路径，先开 strict 会产生两遍修改。
4. **`login_intercept_controller.dart` 与其原生端**由 09-01-settings-productization 决定；C/D 只清理残留。
5. **B 的 rhttp feature 裁剪不得删除 Dart 侧仍导出的 API 对应能力**（如 `HttpBody.multipart` 生成代码
   仍存在）；每去一个 feature 先跑插件 `flutter test` 与 `cargo test`。
6. 任何 child 不得以"重构"为名改变 `NetworkAccessPolicy` 的路由阶梯、ECH/SNI、证书校验或 fallback 顺序。
7. **`material_ui` 系依赖只在 F 升级**（A 不动 `go_router`/`cached_network_image` 的大版本）；F 迁移后重测 per-ABI 体积并更新 B 的门禁阈值。
8. **C 的 `routes.dart` 门面以 id/参数为形参**，F 用 go_router 重实现时调用点不变；C 不引入 go_router。
9. **只有 F 允许改变用户可见交互与视觉**；F 的每项变化写进 `component-guidelines.md`，golden 只在 F 重生成。

## Out Of Scope

- 在对应 09-01 child 归档前改写其代码或未定契约。
- 重新设计网络绕过路线、认证协议、下载产品行为、翻译/搜图产品决策、发布密钥策略。
- json 代码生成、Result 类型、全局缓存/预取/并发/请求缓存层；`material_ui`/`go_router` 之外的 UI/路由库；平板专用布局。
- 直接手改生成文件（`frb_generated.*`）、构建缓存或 vendored 第三方源码逻辑。
- F-Droid 上架元数据与其构建服务器配置（本 task 只保证产物形态与工具链声明满足要求）。
- 以性能为名的架构改造：Impeller 调优、大规模 isolate、自研图片缓存、自定义渲染管线；任何运行时性能基线或数值化性能结论（D-9）。

## Acceptance Criteria

- [ ] D-1～D-13 均有用户明确答复并记录在本文件。
- [ ] 每个 child 的 `prd.md`（复杂 child 另有 `design.md`/`implement.md`）通过 review 后才 `task.py start`。
- [ ] A：`flutter pub outdated` 直接依赖无可解析的落后项（`go_router`/`cached_network_image` 的 material_ui 系大版本除外，留给 F）；
      FRB 三元组校验通过；CI 跑 Kotlin/插件/Rust 测试与格式检查。
- [ ] B：release 产物为 per-ABI APK，arm64 ≤ 32 MB 且不含 x86_64；updater 在 API 29 与高版本 Android
      上各完成一次跨 ABI 选择的真实自更新；`libsqlite3.so` 不再出现在 APK；CI 体积门禁生效；
      每项裁剪都有前后测量记录。
- [ ] C：分层校验脚本通过（0 处 `core→features`，repository/controller 全部在 `core/`，页面环仅剩
      路由门面）；主机 allowlist 只在 `PixivClientIdentity` 一处声明；零引用文件/声明为 0；
      i18n key 集合校验通过；`settings_page.dart` 等 4 个文件拆分后单文件 ≤ 600 行；3 个
      ChangeNotifier 控制器已入 Riverpod；行为测试全绿且未删除测试。
- [ ] C（R8）：`lib/app/` 按 design §13.1 组织；8 处瀑布流统一为 `IllustFeedGrid` 且手机仍为 2 列；`PixivImage`
      变体覆盖全部 13 个调用点并带 decode 策略；`motion_tokens.dart` 是唯一时长/曲线来源；共享组件与四类高频动作
      有语义标签；`features/` 无与组件层同职责的私有 widget。
- [ ] C（R9）：feed/avatar 变体带 `memCacheWidth`（测试断言）；首帧前等待项未增加（`startup_gate_test`）；下载恢复表有上限或增量写入；
      history 查询计划单测不出现 `SCAN TABLE`；MediaStore/SAF 写块 ≥ 256 KiB。
- [ ] F（R10）：`lib/` 无 `package:flutter/material.dart` import；`useMaterial3` 为默认；`MaterialApp.router` + `StatefulShellRoute`
      承载全部页面，`Navigator.of(context).push(ReplicaPageRoute` 为 0；`pixiv://`/web 深链经 go_router；"不保留活动" 下回到同
      tab/同页/同滚动区间；Android 14+ 预测性返回可用；viewer 拖拽关闭；`NavigationBar`/`SearchBar`/`SegmentedButton` 落地；
      golden 重生成；README 原则改写；per-ABI 体积复测并更新 B 阈值；`flutter analyze`/`flutter test` 全绿。
- [ ] D：channel 错误风格与线程约定有 spec 并被代码遵守；12 处死分支清除；updater 重复函数合并；
      `UPSTREAM.md` 覆盖全部 fork 差异与再生成步骤。
- [ ] E：spec 索引状态列与实际一致，6 份模板已填；`lib/core/<domain>/` 均有 library 文档；测试假实现
      重复 ≤ 1 份；`dart format` 与两批 lint 在 CI 中为 0 问题。
- [ ] `flutter analyze`、`flutter test`、Kotlin/插件/Rust 测试、双 flavor per-ABI 构建通过；
      parent 真机矩阵无回退；未测量的改善没有被写成数值结论。

## Notes

- 本 task 是复杂 task，已于 2026-09-07 转为 parent。child 已创建（`task.py create --parent`，均 `planning`）：
  A `09-07-dependency-toolchain-health`、B `09-07-release-size-per-abi`、C `09-07-dart-architecture-convergence`、
  D `09-07-native-rust-hygiene`、E `09-07-spec-test-lint-hardening`、F `09-07-interaction-visual-modernization`（2026-09-07 晚创建）。
  各 child 的 `prd.md` 已从本文件派生；B、C、F 在 `task.py start` 前需补 `design.md`/`implement.md`，D 需补 `implement.md`。
- parent 自身没有直接实现工作；顺序 A → B → C → D → F → E（D 与 C 可并行）；先启动 A。
- 研究文件目录：`.trellis/tasks/09-02-performance-size-maintainability-refactor/research/`。
- 工作树在 2026-09-07 含大量 09-01 的未提交改动，所有研究计数以当天工作树为准，实现前须重算。
