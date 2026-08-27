# 同步跨页面收藏状态与交互 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 从 beta56 组件和 final fix 提取交互、尺寸、错误与同步证据。
2. 定义 BookmarkKey/State/Store/Repository 和远端 snapshot merge 规则。
3. 实现 add/delete、operation revision、并发抑制、失败 rollback 与账号隔离。
4. 替换 Recommended/Detail 的局部状态，复刻短按、长按 sheet 和 spinner。
5. 增加跨页面 provider、竞态、晚到响应、API snapshot、账号切换和 Widget 手势测试。
6. 运行 analyze/test/build，并用真实账号验证 public/private/add/delete 与页面往返。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-bookmark-state-sync
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- short public、long public/private、bookmarked long no-sheet。
- pending no optimistic、failure rollback、success sync。
- 同 ID 双页面并发、重复点击、晚到 response/revision。
- remote snapshot merge 和账号隔离。
- sheet 高度/选项、spinner 类型尺寸与错误反馈。

## Risky Files and Rollback Points

- lib/core/entity/bookmark_store.dart、BookmarkRepository、Recommended/Detail bookmark widgets、IllustStore mapper

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

