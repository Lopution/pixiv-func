# 复刻 Android Home Widgets — Design

## Objective

在不复制明文凭据的前提下，复刻beta56推荐/刷新Android Home Widgets及后台更新、点击导航。

## Architecture and Boundaries

- WidgetDataBridge只暴露最小、短期、非秘密render model；认证请求通过安全native credential adapter或headless Dart方案。
- WidgetUpdateCoordinator使用unique WorkManager并按widget IDs/account revision生成。
- RemoteViews renderer加载受限bitmap并创建immutable/explicit PendingIntents。
- Account change event触发清除旧render cache和立即refresh。

## Data Flow

widget schedule/refresh → secure account capability → strict recommended request → bounded image → RemoteViews update；click→typed deep link；account invalid→redacted state。

## Compatibility, Security, and Migration

- 保持beta56推荐/刷新Widget体验，内部不读取旧Flutter prefs账号JSON。
- 若后台安全认证不可行，保留打开app刷新降级并明确记录，不泄密。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

