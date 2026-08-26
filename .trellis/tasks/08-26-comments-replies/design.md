# 复刻评论与回复 — Design

## Objective

复刻beta56作品评论、回复和输入体验，并用正确的Comment ID/parent ID模型消除本地更新错误。

## Architecture and Boundaries

- CommentStore keyed by CommentId并维护thread index，parent/root字段类型分开。
- CommentRepository封装list/replies/create/delete；operation revision防止晚到结果覆盖。
- CommentComposer拥有mode(root/reply)、text/asset selection和submit state。
- 资产作为不可变bundle，translation通过独立service返回overlay而不改原文entity。

## Data Flow

detail comments → root paging/store → item/replies paging；reply icon → composer(parentId) → API → store insert/update；delete own → API → store remove；translate→overlay。

## Compatibility, Security, and Migration

- 保留beta56emoji/stamp资产和布局，内部不用GetX局部controller。
- translation provider变化不影响评论核心。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

