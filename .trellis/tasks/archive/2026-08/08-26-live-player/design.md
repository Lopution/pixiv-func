# 复刻 Live 播放器 — Design

## Objective

重新核验Pixiv Sketch Live当前可用链路，并在可用时复刻beta56播放器、清晰度、全屏和作者关注体验。

## Architecture and Boundaries

- FeasibilityProbe 是实现前独立阶段，只产出日期、endpoint/auth/schema/HLS/host-policy 结论；通过后才引入 LiveRepository、player dependency 与 UI。
- LiveRepository隔离list/detail/schema，LiveStreamResolver验证HLS variants/redirect hosts。
- LiveRepository/LiveStreamResolver 只能消费 shared `NetworkAccessPolicy`；当天核验通过的 HLS host 才能进入 registry，未知 redirect/第三方 relay 仍被拒绝。
- LivePlayerController封装player state、gesture arbitration、quality/position、lifecycle和resource leases。
- Orientation/Wakelock adapters为引用计数lease并在finally/dispose恢复。
- Owner/follow只引用UserStore/FollowStore。

## Data Flow

feasibility evidence → approved dependency → Live list/detail → strict stream resolver → player controller → 16:9 UI/controls；fullscreen lease；lifecycle/error cleanup；owner→shared profile/follow。

## Compatibility, Security, and Migration

- 保持beta56可见播放器，不使用旧proxy。
- 若大陆兼容网络已通过，Live 只复用 strict route/health；loopback/WebView 规则不适用于原生 HLS，不能把 WebView route steering 当播放器代理。
- endpoint研究结论写task research且限定日期，后续失效不改为伪数据。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。
