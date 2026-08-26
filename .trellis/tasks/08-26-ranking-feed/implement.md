# 复刻 Ranking 榜单 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start；中间父任务本身不作为实现目标。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 记录 beta56 11 mode、TabBar 和 card layout 证据，核验当前 ranking endpoint/modes。
2. 定义稳定 RankingMode 映射和 repository。
3. 实现 per-mode lazy paging、refresh/loadMore/cancel 和账号隔离。
4. 实现 Ranking page/tab/state keep-alive 与 shared card/detail/bookmark。
5. 增加 mode mapping、per-tab state、错误隔离和 Widget 测试。
6. 运行 analyze/test/build，并用真实账号抽验多个普通/R18 mode。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-ranking-feed
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 11 mode 显式映射和顺序。
- lazy load、tab state/scroll/cursor 独立。
- refresh/load-more race 和账号切换。
- 跨 Recommended/Detail bookmark/entity sync。
- 无权限/未知 mode/error/empty。

## Risky Files and Rollback Points

- lib/features/ranking/、Home Ranking tab、RankingMode/API mapper、shared paging lifecycle

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

