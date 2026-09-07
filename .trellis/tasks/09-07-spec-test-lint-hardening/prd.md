# 规范、测试与 lint 加固（09-02 child E）

## Goal

让 `.trellis/spec/` 描述真实约定而不是模板、让每个 `lib/core/<domain>/` 有一段说明职责与 owner 的文档、
把测试的重复假实现与脆弱断言收敛、并用格式化与更严格的 lint 把上述约定固化进 CI。这是四个目标里
"可维护性（LLM 可发现性）"的直接交付。对应 parent R4（可发现性、测试）、R5（格式/lint）；事实来源
`research/dart-architecture-audit.md` E–F、`research/modernization-gaps.md` A、D.5、E；目标见 parent `design.md` §7–§8。已确认决策 D-7。

## 开工 gate

- E0（文档）可随 child A 开始。
- E1–E3（spec 填写、测试收敛、格式与 lint）在 child C、D、F 之后：C 会重写 JSON 读取与目录结构、F 会改导入与导航，先开
  `strict-*` 会产生多遍修改；`dart format` 一次性格式化必须等前序 child 落地，否则冲突。

## Requirements

### R1. spec 索引与文档（E0/E1）

- 两份 `index.md` 状态列改为真实状态（现状：`state-management.md` 1,259 行、`component-guidelines.md`、`quality-guidelines.md`、
  `in-app-web-profile.md` 已填却标 "To fill"）。
- 填写 6 份模板：`frontend/directory-structure.md`、`backend/directory-structure.md`（C 的分层规则与放置规则、校验测试位置、
  三套 HTTP 栈的归属）、`backend/error-handling.md`（三簇错误的传播与 UI 映射位置，不引入新基类）、
  `backend/database-guidelines.md`（单 DB `history.db` schema v2、工厂注入、Android 用平台 SQLite、桌面/测试用 FFI）、
  `frontend/type-safety.md`（`Map<String, Object?>` + `json_read.dart`、`strict-*` 下的写法、`!` 使用边界）、
  `backend/logging-guidelines.md`（`log()` 出口与 tag）。
- 新增 `backend/android-channels.md`、`backend/rust-plugin.md`（由 D 填实）、`backend/release-pipeline.md`（由 B 填实：per-ABI、
  versionCode、manifest schema 2、符号归档、体积门禁）；本 child 负责骨架、索引与最终一致性检查。
- `lib/core/<domain>/` 每个目录一段 `library` 级 `///` 文档：职责、owner 契约、对应 spec 段落。
- 增加升级策略段：跟随 Flutter stable 的官方 Android 验证矩阵；`material_ui` 系依赖与 app 整体迁移绑定。

### R2. 测试收敛（E2）

- `test/helpers/fake_account.dart`（凭据/元数据/账号 store 假实现 + 标准 `overrides` 列表）、`test/helpers/test_prefs.dart`
  （`InMemorySharedPreferencesAsync` 初始化）替换 17 个文件的重复假实现与 19 处 prefs 样板。只合并，不删测试。
- 去除类名字符串断言：`hero_transition_test.dart` 6 处 `'_GlobalRectClip'`（C 已公开该类型）、`illust_detail_page_test.dart:547`、
  `related_illust_repository_test.dart:121`；`updater_flavor_contract_test.dart` 的源文本断言改为构建/行为断言或明确保留理由。
- 删除 `test/zz_diag_tabbar_geometry_test.dart`（自述 one-off）；`.gitignore` 加 `test/failures/`。
- 测试目录按 `core/`/`features/` 镜像分层为可选项，不作为验收。

### R2b. 组件层与无障碍校验（parent R8）

- 共享组件（`IllustCard`、`FeedTail/Empty/Error`、`showAppSnackBar`、settings 原语、收藏/关注按钮）的 semantics 测试
  （`SemanticsTester` / `find.bySemanticsLabel`）。
- `test/architecture/layering_test.dart` 增加"`features/` 不得定义 `_*Tail/_*Error/_*Empty/_*Card`"名称检查。

### R2c. 静态性能约束的最终核对（parent R9，D-9 不做基线）

- 确认 C/D/F 落地的约束测试都存在并通过：变体 `memCacheWidth` 断言、`startup_gate_test` 首帧等待项、history 查询计划、
  通道分块；`backend/release-pipeline.md` 记录 B 的体积阈值（F 更新后的值）。

### R3. 格式与 lint（E3，D-7）

- `dart format lib test` 一次性提交（纯格式，不含逻辑）；CI 加 `dart format --output=none --set-exit-if-changed lib test`。
- 第一批：`sort_pub_dependencies`、`prefer_single_quotes`、`unreachable_from_main`、`prefer_final_locals`（`dart fix --apply`）→ 0 issues。
- 第二批：`analyzer.language.strict-casts/strict-raw-types/strict-inference: true`、`unawaited_futures`、`avoid_dynamic_calls` → 0 issues。
- 不开 `public_member_api_docs`、`lines_longer_than_80_chars`。违规数以 analyzer 为准，研究中的 grep 估算只作规模参考。

### R4. 不做

- 不新增文档生成工具或站点；不把 spec 写成理想化规范——只写代码里真实存在的约定；不删测试。

## Acceptance Criteria

- [ ] 两份 `index.md` 状态列与实际一致；6 份模板无 `(To be filled by the team)` 占位；3 份新专章存在且与 B/D 的实现一致。
- [ ] `lib/core/` 每个子目录都有 library 级文档并指向 spec 段落。
- [ ] 同名测试假实现重复 ≤ 1 份；`InMemorySharedPreferencesAsync` 直接调用只在 helper 内；测试数不减少。
- [ ] `rg "runtimeType.toString()" test` 为 0；诊断测试已删；`test/failures/` 被忽略。
- [ ] CI 的 `dart format` 检查通过；两批 lint 在 `analysis_options.yaml` 生效且 `flutter analyze` 0 问题。
- [ ] `flutter test` 全绿。
- [ ] 共享组件 semantics 测试存在并通过；layering_test 含私有 widget 名称检查。
- [ ] R2c 的约束测试全部存在并通过；`release-pipeline.md` 的体积阈值与 CI 一致。

## Notes

- 轻量到中等：PRD-only 即可；E3 的两批 lint 各自一个提交，格式化提交单独且不含逻辑改动。
- 本 child 是 parent 归档前的最后一个；完成后执行 parent `prd.md` 的验收清单。
