# 复刻用户主页、关注与资料编辑 — Design

## Objective

作为中间父任务，协调用户主页/关注关系和本人资料编辑两个叶子任务，保证 User、Me 与跨页面用户状态一致。

## Architecture and Boundaries

- 中间父任务拥有 UserEntity/UserStore/FollowStore 跨子任务契约，不作为实现 task。
- User/Profile leaf 提供只读/关注域，ProfileEdit leaf 只通过同一 store 提交成功状态。
- 其他 features 只依赖 typed User route/store，不读取页面 controller。

## Data Flow

user/profile foundation → profile edit → cross-feature user/follow integration → 中间父任务验收归档。

## Compatibility, Security, and Migration

- beta56 header/tabs/selector/actions冻结；内部不 fork extended_sliver。
- 中间父任务回滚不撤销独立叶子提交。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

