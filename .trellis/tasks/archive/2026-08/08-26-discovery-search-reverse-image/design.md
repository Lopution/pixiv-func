# 复刻发现、搜索与反向搜图 — Design

## Objective

作为中间父任务，协调 Ranking、New、Search 和 Reverse Image 四个独立功能，使它们复用已验证的实体、分页、收藏和 Android 输入边界。

## Architecture and Boundaries

- 中间父任务只拥有跨子任务契约和验收矩阵，不成为实现 task。
- 共享 entity/paging/bookmark/router 来自已完成基础任务；各叶子只提供 feature repository/controller/UI。
- Reverse Image 通过 SearchCatalog 的结果路由和 AndroidPlatform 的 content URI adapter 连接，不复制详情/图片加载。

## Data Flow

基础 store/router → Ranking/New/Search leaves → Reverse Image input/result → 跨入口集成验收 → 中间父任务归档。

## Compatibility, Security, and Migration

- beta56 用户可见 tab、搜索入口和结果行为为准；第三方服务内部可以更换但必须保持可观察错误。
- 父任务回滚只解除 task links/集成文档，不回滚已独立提交的叶子功能。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚已独立提交的叶子任务。

