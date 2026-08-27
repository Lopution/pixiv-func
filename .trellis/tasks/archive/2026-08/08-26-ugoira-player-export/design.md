# 实现 Ugoira 播放与 GIF 导出 — Design

## Objective

以磁盘ZIP和有界解码窗口复刻beta56 Ugoira封面、播放/暂停、离屏生命周期和GIF保存体验。

## Architecture and Boundaries

- UgoiraRepository返回metadata和streamed owned ZIP temp handle；SafeZipIndex在解码前应用 typed archive/frame limits 与 header-only dimension/format 检查。
- UgoiraRepository 只依赖 shared Pixiv media transport contract；`NetworkAccessPolicy` 接入由 `restricted-compat-network` 完成，本叶子不得复制 host allowlist、固定 IP 或 proxy URL。
- FrameSource按index读磁盘，DecodedFrameCache为LRU window并负责ui.Image dispose。
- UgoiraScheduler使用monotonic clock/deadline而非递归Future.delayed漂移。
- GifExportJob是媒体 task-group 的 exactly-once post-process，复用queue/sink并以滑动窗口 encoder 输出 owned pending item，成功后commit。

## Data Flow

detail Ugoira → metadata → stream ZIP temp → safe index → bounded decode/cache + scheduler → render；export → frame stream/encoder → MediaStore → cleanup。

## Compatibility, Security, and Migration

- 保留beta56play/paused/offscreen体验，内部完全替换全量解码。
- 后续cache参数可调但默认需经设备基准，不能改变可见帧时序。
- 大陆无外部代理验收在兼容网络 leaf/最终集成统一执行；本叶子提供可注入的 network failure/cancel seam，不把单一 API 成功当作图片或 ZIP 可用。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。
