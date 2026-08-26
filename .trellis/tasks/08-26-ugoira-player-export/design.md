# 实现 Ugoira 播放与 GIF 导出 — Design

## Objective

以磁盘ZIP和有界解码窗口复刻beta56 Ugoira封面、播放/暂停、离屏生命周期和GIF保存体验。

## Architecture and Boundaries

- UgoiraRepository返回metadata和streamed ZIP temp handle；SafeZipIndex验证entry/limits。
- FrameSource按index读磁盘，DecodedFrameCache为LRU window并负责ui.Image dispose。
- UgoiraScheduler使用monotonic clock/deadline而非递归Future.delayed漂移。
- GifExportJob复用媒体queue/sink并以有界encoder输出。

## Data Flow

detail Ugoira → metadata → stream ZIP temp → safe index → bounded decode/cache + scheduler → render；export → frame stream/encoder → MediaStore → cleanup。

## Compatibility, Security, and Migration

- 保留beta56play/paused/offscreen体验，内部完全替换全量解码。
- 后续cache参数可调但默认需经设备基准，不能改变可见帧时序。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

