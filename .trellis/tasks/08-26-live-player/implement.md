# 复刻 Live 播放器 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 执行当日Live endpoint/auth/schema/HLS可用性研究并保存脱敏证据。
2. 定义LiveEntity/Repository/StreamResolver/errors和player resource contracts。
3. 实现list/preview/detail及player state/gestures/quality/controls。
4. 实现fullscreen/orientation/wakelock/lifecycle/owner/follow。
5. 增加gesture、quality、resource cleanup、error/ended和mapper tests。
6. 运行analyze/test/build，并在获准真实Live上完成设备验证。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-live-player
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- endpoint/schema fixtures与host policy。
- single/double gesture arbitration。
- quality position/state、buffer/retry/ended。
- orientation/wakelock/player/socket cleanup。
- owner/follow shared state和no-chat audit。

## Risky Files and Rollback Points

- video/HLS依赖、lib/features/live/、Androidorientation/wakelock、真实Live endpoints

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

