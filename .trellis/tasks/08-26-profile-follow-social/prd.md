# 复刻用户主页、关注与资料编辑

## Goal

作为中间父任务，协调用户主页/关注关系和本人资料编辑两个叶子任务，保证 User、Me 与跨页面用户状态一致。

## Confirmed Facts

- 该范围已拆为 user-profile-follow 与 profile-edit 两个叶子任务。
- beta56 User/Me 依赖 extended_sliver/extended_nested_scroll_view；父 PRD要求改为项目自己的 ReplicaProfileHeaderDelegate。
- Follow、Profile metadata 和 Account current user 会被 Search、Comments、Live、Widgets 等功能共同消费。

## Dependencies

- 08-26-bookmark-state-sync 完成。
- Search/User route 与 AccountStore/UserStore 契约可复用。

## Requirements

- R1: 本中间父任务不直接实现产品代码，只维护两个叶子任务范围、顺序和共享 UserStore/FollowStore 契约。
- R2: user-profile-follow 先完成 User/Me、tabs、header、关注状态和共享 user entity。
- R3: profile-edit 后完成当前账号资料编辑，并在成功后原子更新 AccountStore/UserStore。
- R4: 两个叶子任务分别走 planning approval、inline实现、验证、提交和归档。
- R5: 完成前运行 Search/Comments/Live 所需用户 route、账号切换、header scroll 和 follow/edit 集成回归。

## Acceptance Criteria

- [ ] 两个叶子任务均归档，依赖和证据完整。
- [ ] 任意入口打开同一 user ID 时 Profile 与 follow 状态一致，当前账号资料编辑后所有视图同步。
- [ ] expanded/collapsed header、tabs、scroll position、public/private selector 和 Settings action符合 beta56。
- [ ] 账号切换、404/受限、网络失败和编辑失败不串用户状态。
- [ ] 全量 analyze/test/debug build 与 Profile 集成真机验证通过。

## Out of Scope

- 私信、发布作品、完整 Live。
- 更换 Profile 可见设计。

## Risks and Deferred Items

- 当前 Profile edit endpoint/网页流程可能变化；叶子任务必须实时核验，失败时明确 blocker。

## Source Anchors

- beta56 lib/pages/user/*、components/follow_switch_button/*
- 父 PRD Profile 行为与两个叶子任务 artifacts

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
