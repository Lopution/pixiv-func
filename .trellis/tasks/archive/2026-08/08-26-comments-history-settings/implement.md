# 复刻评论、历史与设置 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 审阅settings-parity并冻结schema/default/provider接口。
2. 审阅并完成history-persistence与comments-replies叶子任务。
3. 核对settings toggles、account isolation、shared entity routes、restore/dispose ownership和non-destructive DB migration/downgrade。
4. 运行comment late-response/no-auto-replay、history restore/remove竞态及Profile/Detail/Download跨功能回归。
5. 记录父acceptance evidence并归档。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-comments-history-settings
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 三个child task状态/evidence。
- settings→history/service integration。
- comments/history→shared user/illust routes。
- account switch、DB migration、offline/error。
- restart persistence和secret separation。

## Risky Files and Rollback Points

- 三个child contracts、settings schema/history events；父任务不直接编辑业务代码

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
