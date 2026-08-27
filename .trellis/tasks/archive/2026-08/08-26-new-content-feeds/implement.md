# 复刻 New 内容流 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start；中间父任务本身不作为实现目标。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 提取 beta56 三 tabs、re-tap selector、Illust/Novel source 与 state keep 行为。
2. 核验每个 scope/type 当前 endpoint，定义 NewFeedKey/repositories。
3. 实现 keyed lazy paging、账号隔离、refresh/error/cancel。
4. 实现 New UI、type selector、shared previews/routes 和状态保持。
5. 增加组合矩阵、re-tap、空/权限、账号切换和 Widget 测试。
6. 运行 analyze/test/build，真实账号验证可用 scope/type。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-new-content-feeds
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 三 scope×两 type key/routing。
- re-tap selector、tab/type keep-alive。
- per-key paging/error/refresh 独立。
- 空 follow/my-pixiv 与账号切换。
- Illust/Novel shared state/routes。

## Risky Files and Rollback Points

- lib/features/new/、Home New tab、Novel/Illust preview integration、多个 paging controller 生命周期

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

