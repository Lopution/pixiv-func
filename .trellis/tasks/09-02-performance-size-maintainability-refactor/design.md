# 技术设计：全项目重构（可维护性 / 可迭代性 / 简洁性 / 现代化）

本文件把 `prd.md` 的目标翻译成可执行的边界、规则与机制。所有数字来自 `research/` 五份文件
（2026-09-07 工作树）；实现时以当时 HEAD 重算。

## 1. 设计原则

1. **规则先于搬移**：任何目录重排前，先落地能自动检查该规则的测试/脚本，再迁移代码。
2. **一个契约一个 owner**：主机/头、设置键、偏好存储、i18n 访问、用户反馈、JSON 读取、
   channel 错误码、版本三元组，各自只有一处声明；其余全部派生。
3. **删除优先于抽象**：先删零引用、合并等价实现，再考虑是否需要新组件；新组件必须有 ≥3 个
   现存重复作为证据。
4. **体积只按测量说话**：每项裁剪附前后 `unzip -l`；未测量的写"待测量"，不进验收结论。
5. **源文件优先**：`frb_generated.*`、`Cargo.lock`、`pubspec.lock`、cargokit 输出只由源与命令产生。
6. **不改语义**：路由阶梯、ECH/SNI、证书校验、下载恢复、OAuth、数据迁移 key、flavor 契约保持不变；
   唯一的契约变更（per-ABI 发布 + updater 多资产）经 D-1～D-3 显式批准并由 release-blockers 定稿。

## 2. 现状与目标形态

### 2.1 分层（Dart）

现状（`dart-architecture-audit.md` B）：`features→core` 269 条边、`core→core` 392、
**`core→features` 1**（`core/widget/widget_feed_loader.dart:21` → `features/home/recommended/recommended_repository.dart`）、
`app↔features/onboarding` 互引；`core/` 13 个模块落在同一强连通分量；12 个页面文件因 `IllustCard`
定义在 `recommended_illust_page.dart:236` 而成环；18 个 repository 中 5 个、16 个 controller 中 6 个
放在 `features/`。

目标规则（写入 `.trellis/spec/frontend/directory-structure.md`，由 `test/architecture/layering_test.dart`
以 import 图断言）：

```text
lib/main.dart
lib/app/            应用壳：MaterialApp、主题、共享 UI 基件、路由门面     可 import: core, app
lib/features/<f>/   仅 widget/页面/页面级 controller 的 UI 胶水          可 import: core, app；features 之间
                    只能 import lib/app/navigation/routes.dart 门面
lib/core/<domain>/  entity / repository / controller(Notifier) / store /   可 import: core（同层）
                    platform 适配器；不得 import features 或 app
```

允许的例外只有一个：`lib/app/navigation/routes.dart`（路由门面）import 各 feature 的页面入口，
页面 import 门面——这是唯一被批准的环，并在 layering_test 中白名单化。

具体修正项：

| 现状 | 目标 |
|---|---|
| `IllustCard`（`recommended_illust_page.dart:236`）被 6 个模块 import | 移到 `lib/app/widgets/illust_card.dart`；瀑布流 `SliverMasonryGrid.count` 8 处形态一致 → `lib/app/widgets/illust_masonry_sliver.dart` |
| 32 个私有 `_*Tail/_*Error/_*Empty/_*Status` | `lib/app/widgets/feed_states.dart`：`FeedTail`/`FeedEmpty`/`FeedError` 消费 `PagedFeedState`；先列各页文案/按钮差异矩阵，差异以参数表达 |
| `features/` 内 5 个 repository、6 个 controller（`ranking_repository.dart:112`、`recommended_repository.dart:87` 内含 controller） | 全部迁入 `lib/core/<domain>/`，controller 与 repository 分文件 |
| `core/widget/widget_feed_loader.dart` → features | 随 `recommended_repository` 迁入 core 自然消除 |
| `app/app.dart:14` → `features/onboarding/startup_gate.dart`，onboarding 又依赖 `lib/app/*` | `startup_gate.dart` 上移到 `lib/app/`（它是应用壳的一部分） |
| `lib/core/navigation/`（27 行）、`lib/app/navigation/`（16 行）、`lib/app/replica_page_route.dart` | 合并为 `lib/app/navigation/`：`replica_page_route.dart`、`route_observer.dart`、`routes.dart`（门面）；`home_shell_metrics.dart` 的两个可写静态量改为 `InheritedWidget`/provider |
| `replicaRoute()`（3 处）与 `ReplicaPageRoute`（30 处） | 只留 `ReplicaPageRoute` |

### 2.2 单一 owner 契约

| 契约 | owner | 派生方（改为从 owner 读取） |
|---|---|---|
| Pixiv 主机、Referer、UA、Accept-Language | `lib/core/network/pixiv_client_identity.dart` | `PixivDestinationRegistry._allows`（`network_contracts.dart:53–63`）、预热列表（`network_policy.dart:176–180`）、`PixivFastRouteStore._bootstrap`（`network_fast_route_store.dart:27–35`）、`network_probe_page.dart:56–59`、`login_intercept_controller.dart:135–139`（若仍存在）、`illust_detail_repository.dart:18` 自建 host；新增 `PixivHeaders.image()` 供 `download_manager.dart:563`、`widget_feed_loader.dart:275`、`ugoira_repository.dart:138`、`pixiv_image.dart:67` |
| 偏好存储实例 | `lib/core/settings/preferences.dart`：`sharedPreferencesProvider = Provider<SharedPreferencesAsync>` | 7 处直接构造（`blocked_tags.dart:18,36`、`settings_repository.dart:30`、`update_download.dart:163`、`network_fast_route_store.dart:40`、`download_recovery.dart:372`、`account_repository.dart:38`）改为注入；测试 19 处 `InMemorySharedPreferencesAsync` 样板改为一个 helper |
| 偏好键 | 同文件 `PreferenceKeys`：保留现有键值不变（`replica.settings.v2`、`pixiv.network.fast_routes.v1`、`pixivfunc.download.recovery.v1`、`blocked_tags` …），只集中声明并注明版本；**不迁移用户数据** |
| i18n 访问 | 见 §3 | 27 个 `_xxxText` 包装、38 处 `Localizations.localeOf(context).toLanguageTag()` |
| 用户反馈 | `lib/app/widgets/feedback.dart`：`showAppSnackBar(BuildContext, String, {action})`，内部 `ScaffoldMessenger.maybeOf` | 44 处手写 `showSnackBar`（`.of`/`.maybeOf` 混用统一为 maybeOf 语义） |
| JSON 读取 | `lib/core/entity/json_read.dart`：`readMap/readList/readString/readInt/readBool/readNextUrl` 纯函数，容器类型统一为 `Map<String, Object?>` | `_map`×5、`_firstString`×3、`_positiveInt`×5、`_optionalString`×4、`_requiredString`×2、`_nonNegativeInt`×2、`_nextUrl`×5 |
| 日志 | `lib/core/log.dart`：`log(String tag, String message)` 极小函数（release 静默或保留 `debugPrint`） | 23 处 `debugPrint`/5 文件 |
| 错误分类 | 不新增基类；`backend/error-handling.md` 记录三簇（`ApiError` sealed、download/ugoira/network 各簇、`*Failure` 值对象 + `enum *Kind`）的传播规则与 UI 映射位置 | 页面级 `_errorText` 函数保留但集中到各 feature 一处 |
| HTTP 客户端实例 | provider 注入：`pixivNetworkFactoryProvider`（已有）、新增 `thirdPartyHttpClientProvider`（翻译/SauceNAO）、`resolverHttpClientProvider`（DoH/probe 的 `dart:io HttpClient`，保留原因：需要绕过策略层自身） | `http.Client()` 默认参数 5 处、`HttpClient()` 3 处；生产代码禁止内联构造 |

三套 HTTP 栈的处置：不合并（各有真实原因），但每一套都只能通过 provider 拿到实例，并在
`backend/directory-structure.md` 写明"Pixiv 流量 → rhttp 策略客户端；DoH/探测 → 原生 HttpClient
（策略层的依赖不能反向依赖策略层）；第三方 → 默认 IOClient"。

### 2.3 状态范式

- 3 个 `ChangeNotifier`（`ProfileEditController`、`ReverseImageSearchController`、
  `NovelReaderController`）改为 `Notifier`/`AsyncNotifier` + `.autoDispose(.family)`；页面用
  `ref.watch`；平台适配器（`MethodChannelReverseImageInputPlatform` 等）经 provider 注入，
  页面不再 `new`。契约以 `frontend/state-management.md` 的 "Profile Edit Contract" 等为准，
  测试改为 provider override。
- `history_page.dart:113–145` 手写分页改为 `PagedFeedController` 子类 `HistoryFeedController`；
  account 切换与 outbox 语义按 "Browsing History Contract" 保持。
- 可写静态全局（`HomeShellMetrics.*`、`RhttpGate.ready`）：前者改 provider；后者保留但加文档
  （它是启动期唯一的进程级 gate）。

### 2.4 文件规模

| 文件 | 拆分方式 | 约束 |
|---|---|---|
| `features/settings/settings_page.dart` 2,088 行 / 14 页面类 | `features/settings/pages/<name>_page.dart` 每页一文件；共享 tile 进 `features/settings/widgets/`；`MePage` 只保留 `user_page.dart` 的定义，设置页的本地账号卡改名 `AccountCard` | 保留公开类名或同步改 import；`settings_test.dart` 695 行只改 import |
| `features/illust/detail/illust_detail_page.dart` 1,388 行 | `_GlobalRectClip` RenderObject → `hero_rect_clip.dart`（改公开类型，`hero_transition_test.dart` 6 处类名字符串断言改 `find.byType`）；`_CaptionRichText` → `caption_rich_text.dart`；`_InfoBlock` 等 → `widgets/` | Hero tag 与缓存键契约不变（`component-guidelines.md`） |
| `features/profile/user_page.dart` 959 行 | 3 种 tab feed 各自成文件 | — |
| `core/network/compat/network_policy.dart` 1,198 行 | `network_policy.dart` 只留 `NetworkAccessPolicy`；`pixiv_network_factory.dart`；`image_cache.dart`（`CacheManager` 所有权 + 3.4.x 关闭 bug 绕行注释） | 受 network-perf-ab 契约约束，等其归档；`restricted_compat_network_test.dart` 1,133 行回归 |
| >150 行方法：`update_service.dart:194 _checkOnce`(228)、`download_manager.dart:528 _run`(187)、`login_page.dart:149 build`(186)、`recommended_illust_page.dart:243 IllustCard.build`(180)、`profile_edit_controller.dart:223 submit`(167) | 按阶段拆私有方法/子 widget，行为不变 | `_run`/`_checkOnce` 有恢复/签名语义，先确认测试覆盖 |

## 3. i18n 设计（D-6）

推荐方案 (a) `gen-l10n`：

- `l10n.yaml`：`arb-dir: lib/l10n`，`template-arb-file: app_zh.arb`，`output-class: AppStrings`，
  `nullable-getter: false`（缺 key 编译期报错）。
- 迁移脚本（一次性，放 `tool/`）：从 `ReplicaStrings._values` 生成 `app_zh.arb`、`app_en.arb`、
  `app_ja.arb`、`app_ru.arb`；带 `{0}` 参数的 key 转为 ARB 占位符；ru 缺失的 `detailQuality`
  显式补齐（沿用 zh 文案并标注 TODO-翻译）；修复 `networkDohEndpointsInvalid`。
- 调用点：`AppStrings.of(context).<key>`；删除 27 个包装与 `ReplicaStrings`；`ReplicaLanguage` 保留为
  设置枚举并映射到 `Locale`；`MaterialApp.locale` 由设置驱动（现有行为）。
- 校验：`gen-l10n` 本身保证四个 ARB key 集合一致（`untranslated-messages-file` 输出非空即失败）；
  删除 `i18n_network_keys_test.dart` 的手写 key 列表。
- 体积：`flutter_localizations` 301 KB 不变；生成类按 locale 拆分，tree-shaking 后不大于现字典
  （待测量）。

备选 (b)：`lib/core/i18n/strings/<lang>.dart` 每语言一个 `const Map`，`lib/core/i18n/keys.dart`
生成 `abstract final class K { static const welcome1 = 'welcome1'; … }`，`context.strings(K.x)`
扩展；测试自动比对四表 key 集合与 `K` 常量集合。

## 4. 体积与发布流水线（D-1～D-5）

### 4.1 per-ABI 发布

- 构建：`flutter build apk --release --flavor <f> --split-per-abi --target-platform android-arm64,android-arm --obfuscate --split-debug-info=build/symbols/<f>`
  产出 `app-<f>-arm64-v8a-release.apk`、`app-<f>-armeabi-v7a-release.apk`；不产出 x86_64、不产出 universal。
- versionCode：保留 Flutter 偏移（arm64 = `2000 + n`，armeabi-v7a = `1000 + n`）。
- cargokit 只为请求的 target 编译（`build_gradle.dart:23–30`），但输出目录 `build/rhttp/jniLibs/<buildType>/`
  不清理非目标 ABI（实测污染 10.4 MB）→ 在 `plugins/rhttp/rhttp/cargokit/gradle/plugin.gradle` 的
  cargo 任务前 `delete(cargoOutputDir)`（vendored cargokit 差异，记入 `UPSTREAM.md`）。
- CI：`ci.yml` 与 `release.yml` 改为遍历两个 split APK：逐个 `apksigner verify`、拒绝 debug 签名、
  输出 `unzip -l` 汇总（总大小、`lib/<abi>`、`classes.dex`、assets）并与阈值比较；`--split-debug-info`
  的符号目录作为 release 资产上传（私有或附件）。

### 4.2 updater 多资产（由 release-blockers 定稿，此处给出建议 schema）

```json
{"schema":2,"repository":"Lopution/Pixiv-func","tag":"v1.0.0","channel":"stable",
 "version":"1.0.0","versionCode":1,
 "assets":[
   {"abi":"arm64-v8a","url":"https://github.com/Lopution/Pixiv-func/releases/download/v1.0.0/pixiv-func-v1.0.0-github-arm64-v8a.apk",
    "size":0,"sha256":"…","versionCode":2001},
   {"abi":"armeabi-v7a","url":"…-armeabi-v7a.apk","size":0,"sha256":"…","versionCode":1001}
 ],
 "packageName":"io.github.lopution.pixivfunc","signingCertificateSha256":"…"}
```

- Kotlin `platformInfo` 增加 `supportedAbis`（`Build.SUPPORTED_ABIS`）；Dart `update_service.dart:391–404`
  选择第一个匹配 ABI 的资产，版本比较改为 `semver(version)` 再比 `versionCode % 1000`。
- `tool/update_release.py generate` 接受多个 `--apk <abi>=<path>` 与对应 URL；schema 1 保留只读兼容
  一个版本周期（若 release-blockers 已按 schema 1 发布过）。
- `isStrictUpdateManifestAssetUrl` 与 200 MiB 上限不变。

### 4.3 单 ABI 内的裁剪（全部"待测量"，逐项前后对比）

| 项 | 机制 | 预期 |
|---|---|---|
| `libsqlite3.so` | D-4 保留桌面分支与 `sqflite_common_ffi` 主依赖，`sqlite3` 的 hook 无按 OS 跳过选项（`hook/build.dart` 对每个 targetOS 编译）。因此改为 Android 侧排除：`android/app/build.gradle.kts` `packaging { jniLibs { excludes += "**/libsqlite3.so" } }`；`history_database.dart:27–31` 的 `Platform.isAndroid` 分支保证运行时不加载它；新增测试断言 Android 平台工厂不是 FFI 工厂。**待构建验证**：native asset 是否经过 Gradle jniLibs 合并（`NativeAssetsManifest.json` 仍会列出该库，但不被加载）。不可行则接受 1.73 MB/ABI | 每 ABI −1.73 MB（已测量的文件大小；排除可行性待验证） |
| `librhttp.so` features | `Cargo.toml` 去掉 `multipart`、`form`、`socks`、`cookies`、`query`、`charset`（应用零使用，`native-and-build-audit.md` B.2）；`tokio = full` → `rt-multi-thread, net, time, sync, io-util, macros`（按编译错误补齐）；压缩四项保留（Pixiv 响应依赖 gzip；zstd/brotli 单独实验） | 待测量；`icu_*` 不会消失（`url` 需要） |
| `librhttp.so` profile | `opt-level = "s"` 实验；比较 `.so` 大小与一次图片/下载吞吐 | 待测量 |
| `rustls-platform-verifier` | 应用固定 webpki 根，Verifier 仅初始化不使用 → 评估移除（增大 fork diff，且 `RootCertSource.platform` 不可用）；默认**保留**，只记录 | 待测量 |
| `libapp.so` | `--obfuscate --split-debug-info` | 待测量（作用于符号段） |
| `cupertino_icons` | 删依赖 | −0.26 MB（已测量） |

每去一个 reqwest feature：`cargo build --release --target aarch64-linux-android`（经 cargokit）→
插件 `flutter test` → 应用 `test/rhttp_client_factory_test.dart`、`restricted_compat_network_test.dart`
→ 真机一次登录+图片+下载。

## 5. 原生层与 Rust 插件约定

### 5.1 MethodChannel 约定（写入 `.trellis/spec/backend/android-channels.md`）

- 载荷：`Map<String, Any?>`，字节用 `ByteArray`/`Uint8List`；无 JSON 字串。
- 参数缺失：显式 `result.error("invalid_argument", "<name> missing", null)`，禁止 `call.argument<T>()!!`。
- 错误码：`<channel>_<reason>` 小写下划线；三种现存风格（笼统 `*_error`、细分码、Map 内 `errorCode`）
  统一为细分码 + `result.error`；updater 的 `{valid:false, errorCode}` 形态保留（Dart 端已有映射），
  在文档中标注为例外。
- 线程：写文件/解析 APK/复制流等 IO 用 `BinaryMessenger.TaskQueue`（`makeBackgroundTaskQueue`）或
  协程切到 IO 线程，`result` 回主线程；MediaStore/SAF/反查复制/updater 解析四处按此改。
- API 分支：`minSdk = 29`，删除 12 处 `SDK_INT < Q/P/O` 恒真分支；只保留 `TIRAMISU` 3 处。
- 契约表（10 个 channel 的方法/参数/返回/错误码/线程）与 `widget_snapshot/active.json` 文件契约
  一并写入 spec；`login_webview_intercept` 单侧残留由 09-01 决定后删除。
- 两份 `DistributionUpdaterChannel.kt` 的 `platformInfo/packageInfo/signerSha256` 抽到
  `src/main/kotlin/.../updater/UpdaterPlatformInfo.kt`。
- Manifest/模板残留：`drawable-v21/launch_background.xml` 删除；debug/profile 的冗余 `INTERNET`
  删除；`http://pixiv.net` 深链 filter 保留（改变接管范围属产品决策）。

### 5.2 rhttp fork 策略（D-5）

`UPSTREAM.md` 增加"构建配置差异"一节，逐项列出：`Cargo.toml` features/profile、
`android/build.gradle.kts` AGP 9 适配（已存在但未记录）、cargokit 清理步骤、`rust-toolchain.toml`、
`rust/tests/ech_*`、`rust/examples/ech_reqwest_probe.rs`。同步指引扩为：

1. diff 上游 → 2. 重打 ECH 两处 → 3. 重打构建配置差异 → 4. `tool/frb_check.sh` 校验三元组 →
5. 必要时 `flutter_rust_bridge_codegen generate` → 6. 插件 `flutter test` + `cargo test` →
7. 更新 `UPSTREAM.md` commit/版本。

### 5.3 FRB 三元组

`tool/frb_check.sh`：读取 `pubspec.lock` 的 `flutter_rust_bridge` 版本、`frb_generated.dart` 的
`codegenVersion`、`Cargo.toml` 的 pin、`flutter_rust_bridge_codegen --version`；任一不等即失败。
插件 `pubspec.yaml` 的 `flutter_rust_bridge: ^2.12.0` 改为与 Cargo 相同的精确版本，消除漂移来源；
`rhttp.dart:19` 的 `forceSameCodegenVersion: false` 保留（上游行为），由脚本承担校验。

## 6. 依赖与工具链（child A）

按 `dependency-upgrade-audit.md` 的分层执行，顺序：删依赖 → Flutter 3.47.2 → lock 刷新
（`flutter pub upgrade`、`cargo update`）→ Android 工件 → CI actions → 耦合升级（archive+image）→
`flutter_secure_storage` 11 + AGP 9.1.1 → 工具链固化。每步一个提交，每步跑全量检查。
`material_ui` 系（`cached_network_image` 4、`go_router` 18）不升级，记录原因。

CI 目标形态：

| job | 内容 |
|---|---|
| `dart` | `flutter pub get --enforce-lockfile`、`dart format --output=none --set-exit-if-changed lib test`、`flutter analyze`、`flutter test` |
| `plugin` | `plugins/rhttp/rhttp`：`flutter test`；`rust/`：`cargo fmt --check`、`cargo test --locked`；`tool/frb_check.sh` |
| `android-unit` | `./gradlew :app:testGithubDebugUnitTest :app:testFdroidDebugUnitTest` |
| `android-release` | per-ABI 构建（github 签名校验；fdroid 无签名）、体积汇总与门禁 |
| `deps-report`（不阻塞） | `flutter pub outdated`、`cargo update --dry-run` 归档为 artifact |

## 7. lint 与格式（D-7，child E）

- 第一批（C 之前或与 A 同期，低冲突）：`dart format lib test`；`sort_pub_dependencies`、
  `prefer_single_quotes`、`unreachable_from_main`、`prefer_final_locals`（`dart fix --apply`）。
- 第二批（C 之后）：`analyzer.language.strict-casts/strict-raw-types/strict-inference: true`、
  `unawaited_futures`、`avoid_dynamic_calls`。
- 不开：`public_member_api_docs`、`lines_longer_than_80_chars`。
- 每批以"0 issues"提交；违规数以 analyzer 输出为准，研究文件中的 grep 估算只做规模参考。

## 8. 面向 LLM 维护者的文档交付

| 文件 | 内容 |
|---|---|
| `frontend/directory-structure.md`、`backend/directory-structure.md` | §2.1 分层规则、目录职责、放置规则、校验测试位置 |
| `backend/error-handling.md` | §2.2 三簇错误的传播与 UI 映射位置 |
| `backend/database-guidelines.md` | 单 DB `history.db` schema v2、工厂注入、测试用 FFI |
| `frontend/type-safety.md` | `Map<String, Object?>` + `json_read.dart`、strict-* 开启后的写法、`!` 使用边界 |
| `backend/logging-guidelines.md` | `log()` 出口与 tag 约定 |
| `backend/android-channels.md`（新） | §5.1 契约表 |
| `backend/rust-plugin.md`（新） | fork 策略、构建配置差异、再生成、三元组、cargokit ABI 行为 |
| `backend/release-pipeline.md`（新） | per-ABI 构建、versionCode、manifest schema 2、符号归档、体积门禁 |
| 两份 `index.md` | 状态列改为真实状态 |
| `lib/core/<domain>/<domain>.dart` 或目录首文件 | `library` 级 `///` 说明职责、owner 契约、对应 spec 段落 |

## 9. 兼容性与迁移

- 数据：偏好键、`replica.settings.v2` JSON、`history.db` schema、凭据、下载恢复记录全部不变；
  i18n 迁移不触及持久化（`ReplicaLanguage` 枚举保留）。
- updater：schema 2 上线前发布的客户端只认 schema 1；若 release-blockers 已发布过 schema 1，
  发布侧同时生成两份 manifest 一个周期。
- Android：API 29 路径不变；`--obfuscate` 不影响 MethodChannel（名称为字串）与 `@pragma('vm:entry-point')`
  的 `widgetBackgroundMain`（需回归验证 widget 后台入口）。
- 生成代码：只在 FRB 版本变更时再生成；`frb_generated.*` 变更必须与 `Cargo.toml`/`pubspec` 同提交。

## 10. 回滚策略

- 每个 child 内按 §implement 的阶段单主题提交；删除旧路径（旧 repository 位置、`ReplicaStrings`、
  `replicaRoute()`、桌面 SQLite 分支、旧 manifest schema 读取）是显式回滚点，先有新路径的测试再删。
- 体积类改动每项独立提交并附测量；任一导致真机功能回退（TLS 握手、解压、widget 后台）的裁剪整项回滚。
- 不使用 `git reset --hard`/`git clean` 覆盖用户改动。

## 11. 验证矩阵

| 层 | 必须验证 |
|---|---|
| Dart | `dart format --set-exit-if-changed`、`flutter analyze`（两批 lint 生效后）、`flutter test`、`test/architecture/layering_test.dart`、i18n key 校验 |
| Rust/FFI | `cargo fmt --check`、`cargo test --locked`、`tool/frb_check.sh`、插件 `flutter test` |
| Android | Kotlin JVM 测试；channel 错误码/线程约定的单测；API 29 真机 |
| 构建/产物 | 双 flavor per-ABI 构建；`unzip -l` 汇总与阈值；签名与 manifest；符号目录存在 |
| 许可/维护 | `cargo update` 后新 crate 许可证；NOTICE/UPSTREAM.md 同步 |
| 用户回归 | parent 真机矩阵 + 跨 ABI 自更新 |

## 13. 设计轴一：UI 组件体系（2026-09-07 GPT 评审后补入）

结论：不做 "M2 → M3"，也不单开 UI task；把 `lib/app/` 明确定义为项目的组件层（design system），
C 中已有的 `IllustCard`/瀑布流/feed 状态/反馈/导航收敛都归入这条轴。事实依据：8 处 `SliverMasonryGrid.count`
全部写死 `crossAxisCount: 2`；Hero + `PixivImage` 组合 3 处（卡片、详情、Ugoira viewer）；`PixivImage` 13 个调用点、
`PersonAvatar` 9 处；`ReplicaPageRoute` 与 `_GlobalRectClip` 是仅有的转场原语；`Semantics(` 0 处但 37 个 `IconButton`
中 35 个带 `tooltip`，14 处 `GestureDetector` 无语义。

### 13.1 `lib/app/` 目标结构

```text
lib/app/
  app.dart, startup_gate.dart                 应用壳
  theme/      func_tokens.dart, replica_theme.dart     颜色/间距/字号 token（C9 把散落的 Color(0x…) 收进来）
  motion/     replica_page_route.dart, hero_rect_clip.dart, motion_tokens.dart（时长/曲线单一来源）
  navigation/ routes.dart（门面）, route_observer.dart
  widgets/
    feed/     illust_card.dart, feed_grid.dart（列数由宽度计算）, feed_states.dart（Tail/Empty/Error）
    pixiv_image.dart（含 decode 策略与变体构造器）, person_avatar.dart
    feedback.dart（showAppSnackBar）, replica_button/scaffold/switch_tile/empty_state（现有）
    settings/ settings_section.dart, settings_tile.dart（若 settings 子页之外也复用；否则留在 features/settings/widgets）
```

规则：`features/` 不得再定义与 `lib/app/widgets/` 同职责的私有 widget（layering_test 以名称模式检查
`_*Tail/_*Error/_*Empty/_*Card`）；新组件必须有 ≥3 处现存重复作为证据（`NovelCard`/`UserCard` 先盘点，
不预设）。

### 13.2 组件契约

- **FeedGrid**：`feed_grid.dart` 提供 `IllustFeedGrid`（sliver）与 `illustColumnsFor(double crossAxisExtent)`：
  以最小卡片宽度计算列数，手机宽度下恒为 2（与现状一致），折叠屏/平板自然增加。8 个调用点统一。
- **PixivImage**：保持单一 widget，增加 `decodePolicy`（`PixivImageSize.feed/detail/viewer/avatar`）与对应
  命名构造器；feed/avatar 传 `memCacheWidth`（按布局宽度 × devicePixelRatio 取整，上限 1.5× 逻辑像素），
  detail 按屏宽，viewer 不限制。Hero 组合由 `PixivImage.hero(tag:)` 承担，`_GlobalRectClip` 公开为
  `HeroRectClip` 放入 `motion/`。转场契约（`component-guidelines.md` Detail Transition）不变。
- **Motion**：`motion_tokens.dart` 集中 300ms/`easeInOutCubic` 等常量；`ReplicaPageRoute`、Hero、bottom sheet、
  dialog 的时长曲线只从这里取。这是日后 Predictive Back / M3 motion 的接入点，本 task 不启用它们。
- **Settings 原语**：`SettingsSection`/`SettingsTile`/`SettingsControl` 在 C8 拆分时抽出；每个设置子页只
  组合原语，不再各自布局。
- **反馈**：`showAppSnackBar` 是唯一 SnackBar 出口（§2.2）。
- **无障碍基线（只对共享组件与高频动作）**：共享组件带 `Semantics` 标签/`tooltip`、触达 ≥ 48dp、
  可聚焦可键盘激活；14 处 `GestureDetector` 逐个评估改 `InkWell`/`IconButton` 或补 `Semantics`；
  收藏、关注、下载、导航四类动作必须有语义标签。E 增加共享组件的 semantics 测试。

### 13.3 明确延后（记入后续 task 候选，不在 09-02）

| 项 | 原因 | 09-02 内的 enabler |
|---|---|---|
| Predictive Back（`enableOnBackInvokedCallback`）、Hero 手势返回 | 改变用户可见交互 | `motion/` 单一来源；`PopScope` 用法保持现代 API |
| 按 tab 嵌套 Navigator / `StatefulShellRoute` | 改变返回栈语义，且 parent 明确不引入 `go_router` | `routes.dart` 门面：所有跳转经门面，日后只改门面内部 |
| 进程被杀后的状态恢复（tab/滚动/搜索词/viewer 页码） | 产品能力，需单独验收 | feed 状态在 `PagedFeedController` 中集中；`PageStorageKey` 用法保留 |
| Material 3 / M3 Expressive / `NavigationBar` / `SearchBar` / `SegmentedButton` | 打破 beta56 视觉冻结与 golden | 组件层收敛后切换范围只在 `lib/app/` |

顺序：beta56 replica → Func 组件层（本 task）→ 现代交互模型 → M3 视觉。

## 14. 设计轴二：性能约束（2026-09-07 GPT 评审后补入）

性能不作为独立目标或 child，而是 C/B/D/E 的验收维度，并以一次真机基线（P0）为前提。事实依据：
`PixivImage` 无任何 decode 尺寸限制（全仓库仅反查页 `cacheWidth: 1024`），瀑布流按原始像素 decode；
`illustStoreProvider` 是普通 `Provider`，卡片重建不由它驱动，rebuild 是否有问题需实测；`main.dart`/`app.dart`
已把 `Rhttp.init`、网络 warmUp、widget coordinator 置为不阻塞首帧，首帧前只等 settings 与账号读取；
下载恢复记录以 `setStringList` 整表重写；应用自有 SQLite 仅 `history.db`（3 个索引）。

### 14.1 约束

| 维度 | 约束 | 落点 |
|---|---|---|
| 图片内存 | feed/avatar 按布局尺寸 decode；detail 按屏宽；viewer 原尺寸。刷 N 张图后的 PSS 不高于基线，目标显著下降（待测量） | C（`PixivImage.decodePolicy`），B 记录 `CacheManager` 磁盘配置 |
| 重建范围 | 先用 DevTools rebuild 统计与 `Performance Overlay` 实测 feed 滚动；只有证据显示整卡重建时才引入 `select`/拆分边界；不预设 | C |
| 启动 | 保持"首帧前只等 settings + 账号"；任何新初始化必须 `unawaited` 或后置；冷启动到首帧不高于基线 | C、D（原生 init 不进主线程同步路径） |
| 持久化增长 | 下载恢复表在 500 条记录时的写入耗时可接受；history 三条查询 `EXPLAIN QUERY PLAN` 命中索引 | C（若需改写为增量写入则单独提交） |
| 请求复用 | 只统计（一次脚本会话内相同 URL 的重复请求数），有证据再决定是否在 repository 层共享 in-flight Future；不建全局缓存层 | P0 基线 → C 决策 |
| 通道开销 | MediaStore/SAF 写入按块经 MethodChannel（`media_store_channel.dart:123`）：测块大小与调用次数，必要时调块大小；不改协议 | D |
| 回归门禁 | C 之后重跑 P0 协议，任一指标劣化超过阈值（首帧 +10%、内存 +10%、jank 帧率 +2 个百分点）即回滚该项 | E |

### 14.2 基线协议（`research/runtime-baseline.md`，真机、用户执行，C 之前与之后各一次）

- 构建：`flutter build apk --profile --flavor fdroid --split-per-abi --target-platform android-arm64`（B3 之前用 `--target-platform`
  单 ABI 即可）。
- 冷启动/首帧：`adb shell am start -W -n io.github.lopution.pixivfunc/.MainActivity` 取 `TotalTime`；`flutter run --profile --trace-startup`
  取 `timeToFirstFrameMicros`；各 5 次取中位数。
- 滚动 jank：DevTools Performance 录制推荐页匀速滚动 60 秒，记录 build/raster 超 16ms 的帧占比。
- 内存：`adb shell dumpsys meminfo io.github.lopution.pixivfunc` 在冷启动、浏览 50 张、200 张后各取 `TOTAL PSS`
  与 `Graphics`。
- 详情打开延迟：从点击卡片到详情首图完成的时间（DevTools timeline 事件或屏幕录制帧数），10 次中位数。
- 请求重复：`NetworkAccessPolicy` 诊断日志开启，统计一次脚本会话（推荐 → 详情 → 用户页 → 返回 ×5）中相同 URL 的重复请求。
- 图片缓存：`flutter_cache_manager` 目录大小与对象数。

### 14.3 明确不做

Impeller 参数调优、大规模 isolate 改造、自研图片缓存、自定义渲染管线、自适应布局之外的平板专用 UI。

## 15. 明确不采用

- 不迁 Material 3 / `material_ui`；不引入 `go_router`/声明式路由/嵌套 Navigator；不引入 json 代码生成、freezed、
  Result/Either；不新建全局 revision/epoch/journal/registry/请求缓存层；不用无界缓存/预取/并发换观感。
- 不为体积替换 aws-lc-rs（ECH 依赖 HPKE）、不删压缩支持、不删 API 29、不删 flavor、不删测试。
- 不把 `--target-platform` 当作单 ABI 打包手段。
- 不在本 task 启用 Predictive Back、状态恢复、M3 视觉（见 §13.3）。
