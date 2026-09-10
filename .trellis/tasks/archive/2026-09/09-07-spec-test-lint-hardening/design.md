# 技术设计：规范、测试与 lint 加固（child E）

本文件从 parent `prd.md` R4/R5/R7–R9、parent `design.md` §7–§8 和本 child `prd.md` 派生，并以
A–D 已合入后的 `origin/main` `973ca71` 为起点。E 分支已 fast-forward 至归档 F 的 `6d7ef46`，因此所有数量
在 E0 对 F 后的 HEAD 重算；本设计固定 owner、文件与收敛方式，不把 09-07 初稿中的旧数量当成验收事实。

## 1. 范围与当前差异

E 负责把已经落地的工程约定写成可发现的 spec，合并测试中等价的基础设施，补共享组件的关键语义测试，
最后一次性完成格式与 lint 收敛。它不再承担 A–D 已完成的实现工作。

| 09-07 初稿事项 | A–D 后现状 | E 的处理 |
|---|---|---|
| 发布专章 | B 已填实 `backend/release-artifacts.md` | 以该文件为唯一名称，修正仍指向已退役发布专章名称的旧引用 |
| Android/Rust 专章 | D 已填实 `android-channels.md`、`rust-plugin.md` | 只核对索引与交叉引用 |
| frontend 分层规则 | C 已填实 `frontend/directory-structure.md` | 保留规则，补 F 后的路由/组件归属 |
| preferences helper | C 已建 `test/helpers/test_preferences.dart`，直接构造只剩 helper 内 | 复用现有文件，不再创建 `test_prefs.dart` |
| `test/failures/` | 已在 `.gitignore` | 保持现状 |
| 私有共享 widget 名称检查 | `layering_test.dart` 已覆盖 Tail/Error/Empty/Card/Status/Placeholder | 核对 F 新增组件后仍通过，按实际缺口调整模式 |
| `unreachable_from_main` | C 已启用且 analyze 为 0 | 第一批 lint 中保持，不重复启用 |
| 类名字符串断言 | 还剩 `PersonAvatar`、`ApiParseError` 两处 | 改为类型 matcher/import |
| one-off 诊断测试 | `zz_diag_tabbar_geometry_test.dart` 仍被跟踪 | 删除；新增的语义测试补足正式测试数量 |
| semantics | `lib/`、`test/` 当前均无显式 `Semantics(` | 在共享交互组件上补标签/状态与聚焦测试 |

`frontend/hook-guidelines.md` 继续标为 `To fill`。项目没有 Flutter Hooks，也没有本 child 要固化的 hook
契约；E 不为填满索引而虚构一章。

## 2. spec 收敛

两份 index 的状态只表达文件是否描述当前代码：`Active`/`Filled` 的文档不得保留模板段，未覆盖的
`hook-guidelines.md` 明确保留 `To fill`。spec 正文继续遵循 index 约定使用 English。

| 文件 | E 写入或核对的事实来源 |
|---|---|
| `frontend/directory-structure.md` | C 的 `app/features/core` 依赖方向、`routes.dart` 门面、repository/controller 放置及 `layering_test.dart` |
| `frontend/component-guidelines.md` | C 的 feed/PixivImage/Hero/Pull-to-Refresh 契约，以及 F 最终的 M3、导航、恢复与拖拽交互 |
| `frontend/state-management.md` | Riverpod owner、family/autoDispose、`PagedFeedController` 三相语义、account 切换边界；移除遗留模板框架 |
| `frontend/quality-guidelines.md` | format、analyzer、layering、测试组织和生成文件边界；规则随 E4/E5 实际启用后更新 |
| `frontend/type-safety.md` | `Map<String, Object?>`、`json_read.dart`、sealed state、nullable/`!` 的现有用法及 strict analyzer 写法 |
| `backend/directory-structure.md` | `lib/core/<domain>` owner、repository/controller/store/platform 放置，以及 Pixiv/rhttp、third-party、resolver 三套 HTTP provider 的归属 |
| `backend/database-guidelines.md` | 单库 `history.db` schema v2、migration、索引、注入的 factory/path；移动端 SQLite 与桌面/测试 FFI 分支 |
| `backend/error-handling.md` | `ApiError`/`ApiHttpError`/`ApiParseError`、各 domain exception、平台错误与 UI `AsyncValue`/feed/SnackBar 的映射位置 |
| `backend/logging-guidelines.md` | `lib/core/log.dart` 是应用日志出口；release 编译不输出，现有消息前缀表达来源 |
| `backend/quality-guidelines.md` | 已有 secrets 约定，加上 lock、生成物、Android/Rust 原生验证入口并清除模板段 |
| `backend/release-artifacts.md` | B 的 split APK、schema 2、签名、体积阈值；F 的最终体积数字 |
| `backend/android-channels.md`、`backend/rust-plugin.md` | D 的最终实现；只修链接或已漂移的事实 |

错误和日志章节记录现有边界：E 不引入统一异常父类，也不改变 `log(String message)` 的签名。数据库章节同样
记录 `HistoryDatabase` 的真实 schema、factory 和查询计划测试，不增加第二套数据访问层。

## 3. core domain 可发现性

每个 `lib/core/` 直接子目录选择一个现有入口文件，在 import 前增加 library 级 `///` 注释与匿名
`library;` 指令。注释包含三项：该 domain 的职责、状态/IO 的 owner、对应 spec 链接。不会为此新增 barrel。

| Domain | 文档入口 | Domain | 文档入口 |
|---|---|---|---|
| `auth` | `account_store.dart` | `bookmark` | `bookmark_store.dart` |
| `comments` | `comment_store.dart` | `download` | `download_manager.dart` |
| `entity` | `illust_store.dart` | `history` | `history_database.dart` |
| `i18n` | `replica_language.dart` | `illust` | `illust_detail_controller.dart` |
| `mutation` | `mutation_boundary.dart` | `navigation` | `route_observer.dart` |
| `network` | `pixiv_http_client.dart` | `new` | `new_feed_controller.dart` |
| `novel` | `novel_store.dart` | `paging` | `paged_feed_controller.dart` |
| `platform` | `android_platform_interfaces.dart` | `profile` | `profile_edit_controller.dart` |
| `reverse_image` | `reverse_image_controller.dart` | `search` | `search_models.dart` |
| `settings` | `app_settings.dart` | `ugoira` | `ugoira_providers.dart` |
| `updater` | `update_service.dart` | `user` | `user_store.dart` |
| `widget` | `widget_coordinator.dart` |  |  |

E0 以 `find lib/core -mindepth 1 -maxdepth 1 -type d` 重算目录；后续若 F 新增 core domain，按同一规则加入。
`backend/directory-structure.md` 保留一张 domain → owner → spec 表，入口注释只给摘要与链接，避免在 23 个文件
复制整段约定。

## 4. 测试基础设施与断言

### 4.1 账号测试 helper

新增 `test/helpers/fake_account.dart`，只承载目前多文件重复的两类内存实现：

- 可按 account id 读写/删除的 `CredentialStore`；
- 可载入并保存 `AccountMetadataSnapshot` 的 `AccountMetadataRepository`；
- 一个生成 `credentialStoreProvider`、`accountMetadataRepositoryProvider` overrides 的小函数。

seeded、empty 两种常用状态由构造参数表达。需要固定失败次数、调用计数、并发门或特殊返回顺序的 test double
继续留在所属测试，因为这些行为就是测试输入，不并入共享 helper。C 已有的 `memoryPreferences()` /
`installMemoryPreferences()` 保持唯一 preferences 测试入口。

### 4.2 脆弱断言与诊断文件

- `illust_detail_page_test.dart` 直接 import `PersonAvatar` 并用 `find.byType(PersonAvatar)`，再读取强类型字段。
- `related_illust_repository_test.dart` 用 `throwsA(isA<ApiParseError>())`。
- 删除 `zz_diag_tabbar_geometry_test.dart`；它只打印几何数据，没有产品断言。
- `updater_flavor_contract_test.dart` 保留为跨 source-set 的构建表面测试：flavor、manifest capability、channel 注册和
  签名算法属于源/构建合同；运行时 manifest、ABI 选择和 Kotlin handler 行为继续由现有 Dart/Kotlin 测试负责。

测试文件目录重排仍为可选项，本 child 不移动 79 个现有测试文件。

## 5. 共享组件语义与静态契约

语义覆盖放入最接近现有行为测试的位置，不建立第二套 widget harness：

| 组件 | 覆盖内容 | 测试位置 |
|---|---|---|
| `IllustCard` | 作品标题/作者可读，点击动作可聚焦 | `recommended_home_test.dart` 或现有 card 使用方 |
| `FeedTail`/`FeedEmpty`/`FeedError` | loading、空态、错误与 retry 标签 | 新增 `shared_component_semantics_test.dart` |
| `showAppSnackBar` | 公告文本进入 live region | 同一共享组件测试 |
| settings 原语 | title、当前开关值、tap/changed action | `settings_test.dart` |
| 收藏/关注按钮 | checked/toggled 状态与 label | `bookmark_switch_button_test.dart`、`user_profile_test.dart` |

实现优先使用按钮、`IconButton.tooltip`、`SwitchListTile` 等控件已有 semantics；只有组合控件缺少可读名称或状态时，
才在共享组件根节点加 `Semantics`。

E0 同时核对以下现有静态约束，缺哪一项才补哪一项：

- `pixiv_image_variants_test.dart`：feed/avatar `memCacheWidth`，viewer 不限宽；
- `startup_gate_test.dart`：首帧只等待 settings 与 account；
- `history_persistence_test.dart`：三条查询 `EXPLAIN QUERY PLAN` 不含 `SCAN TABLE`；
- `download_sink_test.dart`：256 KiB 聚合、结束冲刷、取消丢弃；
- `layering_test.dart`：依赖方向、data-layer 文件位置和私有共享 widget 名称。

## 6. format 与 analyzer 分批

顺序固定，便于 review 与回滚：

1. `dart format lib test` 独立提交；同一提交给 `analyze-and-test` job 增加
   `dart format --output=none --set-exit-if-changed lib test`。
2. 第一批保留已启用的 `unreachable_from_main`，再启用 `sort_pub_dependencies`、
   `prefer_single_quotes`、`prefer_final_locals`。只应用这些规则对应的 fix。
3. 第二批启用 `strict-casts`、`strict-raw-types`、`strict-inference`、`unawaited_futures`、
   `avoid_dynamic_calls`；按 analyzer 报告补具体类型、typed collection 和明确的异步处理。

`public_member_api_docs`、`lines_longer_than_80_chars` 不在本 task。生成的 l10n 文件继续由生成流程维护；vendored
`plugins/rhttp/rhttp` 仍由自己的 package analyzer 管理。

## 7. 兼容性与提交边界

- helper 替换后保留每个测试原有 seed、失败路径和断言；正式产品测试总数不低于 E0 基线，删除的一项诊断测试由
  新增语义测试覆盖。
- lint 修复是等价的类型、引号、final 与 await 标注；如果 analyzer 暴露真实行为缺陷，单独记录并在对应阶段提交。
- E1 文档、E2 helper/测试、E3 format、E4 第一批 lint、E5 strict lint 均为独立回滚点。
- F 的 Hero、tab stack、状态恢复、Predictive Back 与拖拽测试在 E 全量测试中继续通过。
