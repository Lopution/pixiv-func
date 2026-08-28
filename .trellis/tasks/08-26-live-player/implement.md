# 复刻 Live 播放器 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 在不新增player依赖的 feasibility probe 中执行当日Live endpoint/auth/schema/HLS/redirect可用性研究并保存脱敏证据；同时确认候选 host 可纳入 shared `PixivDestinationRegistry`，否则保持 blocker。
2. 若证据不可用或条款不允许第三方播放，记录blocker并停止；不得进入后续步骤或生成mock页面。
3. 证据通过后定义LiveEntity/Repository/StreamResolver/errors和player resource contracts，并单独审查依赖/build影响。
4. 实现list/preview/detail及player state/gestures/quality/controls。
5. 实现fullscreen/orientation/wakelock/lifecycle/owner/follow。
6. 增加gesture、quality、resource cleanup、error/ended和mapper tests。
7. 运行analyze/test/build，并在获准真实Live上完成设备验证。

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
- strict route policy/HLS host allowlist、system proxy/VPN off failure evidence；无独立 fixed-IP/proxy branch。

## Risky Files and Rollback Points

- video/HLS依赖、lib/features/live/、Androidorientation/wakelock、真实Live endpoints

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
