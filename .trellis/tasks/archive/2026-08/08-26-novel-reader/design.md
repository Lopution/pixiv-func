# 复刻 Novel 阅读器 — Design

## Objective

在第一条插画链完成后，用当前 Novel API 重建 beta56 的水平分页阅读体验，并在旋转/字体变化后稳定恢复阅读位置。

## Architecture and Boundaries

- NovelRepository 返回 domain entity 和 versioned content blocks；ReaderLayoutEngine 将 blocks+style+viewport 转为 page spans。
- StableAnchor 使用段落 ID/字符 offset，而不是 page index；重新布局后映射到新 page。
- 布局 cache 有内存上限和 content/style/viewport key，耗时计算可放 isolate但不传大图片 bytes。
- ReaderController 管 PageController、anchor、settings 和 lifecycle；UI 只渲染当前分页结果。

## Data Flow

novel ID → repository/current API → content blocks → cancellable layout → pages + anchor map → horizontal reader；viewport/style change → capture anchor → re-layout/cache → restore page。

## Compatibility, Security, and Migration

- 可见阅读交互跟随 beta56；内部不迁移旧 HTML scraper/GetX viewer。
- 后续 History 通过 reader visibility/anchor 事件接入，不在 Reader 中每秒 Timer。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

