# 复刻用户主页与关注关系 — Design

## Objective

复刻 beta56 UserPage/MePage 的 header、tabs、分页内容和关注交互，并用共享 UserStore/FollowStore 保证跨页面一致。

## Architecture and Boundaries

- UserStore canonicalize UserEntity，FollowStore独立保存关注mutation状态，类似BookmarkStore。
- ReplicaProfileHeaderDelegate根据shrinkOffset计算背景/头像/title/action，纯函数状态可测试。
- ProfileTabController以userId+tab+type+restrict为key持有paging/scroll。
- UserRepository封装detail/work/bookmark/relations endpoints和typed errors。

## Data Flow

typed user route → store snapshot + detail fetch → profile header/tabs；follow action → FollowStore/API → all subscribers；tab key → repository/paging → shared entities。

## Compatibility, Security, and Migration

- 保持beta56视觉/手势，内部替换extended_sliver/GetX。
- Profile edit通过UserStore merge接入，不直接修改页面state。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

