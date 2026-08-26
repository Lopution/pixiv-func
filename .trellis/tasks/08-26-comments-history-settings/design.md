# 复刻评论、历史与设置 — Design

## Objective

作为中间父任务，协调评论、历史和设置三个独立叶子任务，并确保它们共享账号、实体、数据库和配置契约而不互相耦合。

## Architecture and Boundaries

- 中间父任务拥有跨子任务setting keys、history event和entity route契约，不作为实现task。
- SettingsRepository、HistoryRepository、CommentStore各自生命周期独立。
- 跨功能只用typed provider/events，不直接读对方数据库表或controller。

## Data Flow

Settings foundation → History/Comments leaves → cross-feature toggle/entity/account integration → 中间父任务验收。

## Compatibility, Security, and Migration

- 保持beta56页面表现，内部拆分旧全局service。
- 父任务回滚不撤销已独立提交的叶子功能。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

