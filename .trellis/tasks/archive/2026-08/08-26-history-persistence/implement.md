# 实现浏览与 Pixiv History — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 核验beta56页面/删除行为和当前Pixiv history endpoint/计时语义。
2. 选择DB库并定义schema/index/migrations/repository接口。
3. 实现local upsert/page/delete/clear和shared entity hydration。
4. 实现visibility/lifecycle tracker与account outbox/重试。
5. 接入Settings/Detail/Novel等view events并实现History UI。
6. 增加DB migration/transaction、tracker时序、账号/离线和Widget测试。
7. 运行analyze/test/build，在真机验证前后台、旋转、关闭设置与删除。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-history-persistence
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- single DB lifecycle、schema/index/migration。
- upsert/order/page/delete/clear/concurrency。
- visibility/foreground Stopwatch时序，无periodic timer。
- outbox merge/retry/account isolation。
- settings toggles、deleted entity/offline UI。

## Risky Files and Rollback Points

- pubspec DB依赖、lib/core/history/、schema/migrations、Detail/Novel lifecycle hooks、Settings integration

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

