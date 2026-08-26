# 实现下载、Ugoira 与媒体流水线 — Design

## Objective

作为中间父任务，协调流式DownloadManager和Ugoira播放/导出两个叶子任务，统一媒体网络、磁盘、MediaStore和生命周期边界。

## Architecture and Boundaries

- 中间父任务拥有MediaSink、DownloadManager、Ugoira cache/export之间的共享contract，不作为实现task。
- Download leaf负责网络/队列/MediaStore，Ugoira leaf负责ZIP/frame/scheduler/GIF并只调用公开媒体接口。
- 性能/evidence以有界buffer、峰值内存、cleanup终态为核心。

## Data Flow

Download foundation → Detail integration → Ugoira player/export → media cross-feature/performance验收 → parent归档。

## Compatibility, Security, and Migration

- 保留beta56 toast/progress/play视觉，内部不迁移bytes bridge。
- 父任务不回滚独立叶子提交。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

