# 实现推荐作品流与可靠分页 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 从 beta56 卡片和推荐 source 提取 UI/字段/分页证据，核验当前推荐 endpoint/schema。
2. 定义 IllustEntity、IllustStore、Page、PagedFeedState/controller 与 repository 接口。
3. 实现推荐 mapper、请求取消、cursor allowlist、ID 去重和账号隔离。
4. 实现 Recommended Illust 页面各状态、卡片和 Home tab/scroll 保持。
5. 增加 entity merge、分页竞态、恶意 cursor、错误分层、账号切换和 Widget 测试。
6. 运行 analyze/test/build，并用真实账号验证首屏、刷新、分页、错误和状态恢复。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-recommended-feed-paging
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 实体 merge 和账号隔离。
- initial/refresh/load-more 并发、旧请求晚到、cursor exhausted 与 dedupe。
- malicious next URL、schema error、cancel/timeout。
- Home tab scroll/state 恢复和 card badges。
- 真实 API 响应仅作集成验证，不固化私有数据快照。

## Risky Files and Rollback Points

- lib/core/entity/、lib/core/paging/、lib/features/recommended/、lib/features/home/home_page.dart、图片缓存依赖

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

