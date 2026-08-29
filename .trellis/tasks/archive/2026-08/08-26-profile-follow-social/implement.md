# 复刻用户主页、关注与资料编辑 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 审阅并批准 user-profile-follow 叶子规划并完成实现/归档。
2. 基于稳定 UserStore 审阅并批准 profile-edit。
3. 核对 AccountStore/UserStore/FollowStore 原子更新、mutation revision、ProfileCapabilities/minimal patch和账号隔离。
4. 运行 Search/Comments/Live route、header scroll、follow late-response/429、edit confirmed/verification-pending 跨功能回归。
5. 记录父 acceptance evidence并归档。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-profile-follow-social
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 两个 child task 状态和 evidence。
- 同 ID user/follow cross-view sync。
- Me/User header/actions/tabs integration。
- account switch、404、edit failure。
- Search/Comments/Live typed user routes。

## Risky Files and Rollback Points

- 两个 child contracts、UserStore/FollowStore、AccountStore metadata sync；父任务不直接编辑业务代码

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
