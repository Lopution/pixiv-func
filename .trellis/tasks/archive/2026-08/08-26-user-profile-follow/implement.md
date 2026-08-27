# 复刻用户主页与关注关系 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 提取beta56 User/Me header、tabs、selector、share和follow行为，核验当前user endpoints。
2. 定义UserEntity/Store、FollowStore/Repository和tab key/paging接口。
3. 实现ReplicaProfileHeaderDelegate及expanded/collapsed actions/title。
4. 实现User/Me页面、各tab adapters、scroll/state保持和shared routes。
5. 实现follow mutation竞态/rollback/账号隔离。
6. 增加delegate geometry、tabs、paging、follow同步、错误/账号测试。
7. 运行analyze/test/build并在真机验证长滚动、header、follow和多个Profile。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-user-profile-follow
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- header geometry/title/action across extents。
- tab/type/restrict re-tap与scroll state。
- follow pending/success/failure/concurrency/account isolation。
- 404/blocked/me/share/deep link。
- cross Search/Comment/Live user sync。

## Risky Files and Rollback Points

- lib/core/entity/user_entity.dart、UserStore/FollowStore、lib/features/profile/、custom sliver delegate、shared routes

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

