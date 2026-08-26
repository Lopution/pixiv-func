# 复刻 Live、Widgets 与 Updater — Design

## Objective

作为中间父任务，协调Live播放器、Android Home Widgets和Updater flavors三个高时效叶子任务，并隔离它们的网络、后台和权限风险。

## Architecture and Boundaries

- 中间父任务只拥有后台/权限/账号安全cross-cutting验收，不作为实现task。
- Live用Dart网络/player，Widget用最小native/headless bridge，Updater用flavor service；三者不共享私有实现。
- 基础Manifest保持最小，feature manifests/flavors独立merge。

## Data Flow

stable content/platform → three leaves → background/permission/account/flavor integration → parent归档。

## Compatibility, Security, and Migration

- 保留beta56可见体验，替换不安全/过时内部实现。
- 父任务不回滚独立叶子提交。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

