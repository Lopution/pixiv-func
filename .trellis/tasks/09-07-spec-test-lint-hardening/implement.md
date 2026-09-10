# 执行计划：规范、测试与 lint 加固（child E）

## 执行前状态

- 当前 task：`.trellis/tasks/09-07-spec-test-lint-hardening/`；计划分支
  `task/09-07-spec-test-lint-hardening`。
- A–D 已归档；F 已归档，E 分支在开工前 fast-forward 至 F 的 `6d7ef46`。再运行
  `python3 ./.trellis/scripts/task.py start 09-07-spec-test-lint-hardening --allow-empty-context`；本 task 由当前
  agent inline 执行，阶段上下文通过 `trellis-before-dev` 读取。
- F 后快照 `6d7ef46` 有 79 个 `*_test.dart`、713 个 test case；E0 记录该基线。
- 每个 checkbox 一个提交；测试 helper、格式化和两批 lint 分开 review。
- 工具链：`export PATH=/opt/flutter-3.47.2/bin:$PATH`；常规检查使用 `--no-pub`。

## E0：F 后重算与计划对齐

- [x] 对开工 HEAD 重算 spec 占位、core direct domains、重复 account fake、preferences 构造、
  `runtimeType.toString()`、诊断测试、semantics、测试文件/测试 case 数及五项静态性能契约；写入
  `research/hardening-recount.md`。确认发布专章的 canonical path 是 `backend/release-artifacts.md`，把本 task
  `design.md`/`implement.md` 中受 F 影响的文件名和基线同步到实测值。
  提交 `docs(test): recount spec and test hardening baseline`。

## E1：spec 与 domain 可发现性

- [x] 填实 `backend/directory-structure.md`、`database-guidelines.md`、`error-handling.md`、
  `logging-guidelines.md` 和 `frontend/type-safety.md`；清除 `component-guidelines.md`、
  `state-management.md`、frontend/backend `quality-guidelines.md` 中遗留的模板段；合并 C/D/F 已交付的真实契约。
  更新两份 `index.md` 状态，保留 `hook-guidelines.md = To fill`；修正 active task/parent 中仍指向
  `backend/release-artifacts.md` 的链接为 canonical path。
  提交 `docs(spec): replace templates with shipped contracts`。

- [x] 按 `design.md` §3 给 E0 重算出的每个 `lib/core/<domain>/` 选择一个现有入口，增加 library 级职责、owner、
  spec 链接；`backend/directory-structure.md` 增加 domain 索引。运行 analyzer 确认匿名 `library;` 指令合法。
  提交 `docs(core): document domain ownership`。

## E2：测试收敛

- [x] 新增 `test/helpers/fake_account.dart`，提供常用的内存 credential/metadata 实现和 provider overrides；
  替换 E0 确认等价的重复类。带失败脚本、调用记录、并发控制的 test double 保持本地。继续复用
  `test/helpers/test_preferences.dart`，并确认 `InMemorySharedPreferencesAsync` 只在该 helper 内出现。
  提交 `test: share account and preferences fixtures`。

- [x] 两处类名字符串断言改强类型 matcher；删除 `test/zz_diag_tabbar_geometry_test.dart`；在
  `updater_flavor_contract_test.dart` 顶部记录其跨 flavor/source-set 构建合同范围，并删掉与 Dart/Kotlin 行为测试
  重复的断言（若 E0 仍发现）。运行对应 illust、updater 和 home 测试。
  提交 `test: replace brittle type checks and remove diagnostic case`。

- [x] 给 `IllustCard`、`FeedTail/FeedEmpty/FeedError`、SnackBar、settings 原语、收藏/关注按钮补关键 semantics，
  扩展最近的现有测试并新增一个共享组件 semantics 测试；核对 `layering_test.dart` 对 F 后共享组件名称的覆盖。
  提交 `test(ui): cover shared component semantics`。

## E3：一次性格式化

- [ ] 运行 `dart format lib test`，本提交只含格式差异；在 `.github/workflows/ci.yml` 的
  `analyze-and-test` job 中把 `dart format --output=none --set-exit-if-changed lib test` 放在 analyze 前。
  提交 `style: format Dart sources and enforce CI check`。

## E4：第一批 lint

- [ ] 在 `analysis_options.yaml` 保留 `unreachable_from_main: true`，启用 `sort_pub_dependencies`、
  `prefer_single_quotes`、`prefer_final_locals`；先看 `dart fix --dry-run`，再只应用这三条规则的 fix，处理剩余
  analyzer 报告。同步 `frontend/quality-guidelines.md`。
  提交 `lint: enable dependency order, quotes and final locals`。

## E5：strict analyzer 与异步 lint

- [ ] 启用 analyzer language 的 `strict-casts`、`strict-raw-types`、`strict-inference`，以及 linter 的
  `unawaited_futures`、`avoid_dynamic_calls`；按报告补明确类型、typed collections 和 await/unawaited，直到
  `flutter analyze --no-pub` 为 0。同步 type-safety/quality spec。
  提交 `lint: enable strict types and async checks`。

## 阶段验证

每个代码阶段运行 analyzer 和受影响测试；E2、E3、E5 后各跑一次全量：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
cd /root/Pixiv-func-E
flutter analyze --no-pub
flutter test -j 4 --no-pub
flutter test --no-pub test/architecture/layering_test.dart
dart format --output=none --set-exit-if-changed lib test
git diff --check
```

spec 与任务状态检查：

```bash
rg -n '\(To be filled by (the )?team\)' .trellis/spec/frontend .trellis/spec/backend
find .trellis/spec .trellis/tasks/09-02-performance-size-maintainability-refactor \
  .trellis/tasks/09-07-spec-test-lint-hardening -type f -name 'release-*.md' \
  ! -name 'release-artifacts.md'
rg -n 'runtimeType\.toString\(\)' test
rg -n 'InMemorySharedPreferencesAsync' test
```

预期：模板匹配只允许 `frontend/hook-guidelines.md`；旧发布文件名和 runtime type matcher 为 0；preferences
实现只在 `test/helpers/test_preferences.dart`。

## 完成门槛与交付

- 两份 spec index 与实际状态一致；Active/Filled 文档无模板段，B/D/F 交叉引用一致。
- E0 的每个 core direct domain 都有一个 library 文档入口。
- 共享账号实现已合并，行为特化的 doubles 保持在原测试；正式测试 case 数不低于 E0 基线。
- 共享组件 semantics、layering 和四项现有静态性能契约测试通过。
- CI format、第一批 lint、strict/async lint 同时生效，`flutter analyze` 与全量 `flutter test` 通过。
- `trellis-check`、`git diff --check` 通过后，按 workflow 完成 journal、archive、rebase、PR、CI 与 merge；E 归档后再处理
  parent 收尾。
