# 实现流式下载与 MediaStore — Design

## Objective

建立可取消、可恢复且内存有界的作品下载基础，使详情页下载模式和后续 Ugoira 导出使用同一可靠流水线。

## Architecture and Boundaries

- DownloadManager 持有队列和 scheduler；DownloadTransport 复用 PixivHttpClient 的安全 transport；MediaSinkFactory 创建 pending sink。
- 任务状态机使用不可逆终态和幂等 finalize/abort；事件由单一 reducer 发出，避免 onComplete 重复。
- progress 按时间/百分比阈值节流，UI 订阅只读 task snapshot。
- 详情页只提交 typed DownloadRequest，不持有 Dio response 或文件句柄。

## Data Flow

UI request → normalize/dedupe → queue → concurrency permit → strict streaming HTTP → MediaStore pending sink → finalize | abort → one terminal event。

## Compatibility, Security, and Migration

- 默认目录和并发保持原版；内部不保留旧 broad storage permission。
- 后续 Ugoira/Updater 只能复用公开队列接口，不复制下载逻辑。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

