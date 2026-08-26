# 实现流式下载与 MediaStore

## Goal

建立可取消、可恢复且内存有界的作品下载基础，使详情页下载模式和后续 Ugoira 导出使用同一可靠流水线。

## Confirmed Facts

- beta56 downloader 及 Android bridge 存在全量 bytes、isolate 传输和旧存储路径；这些只用于复刻进度/toast/顺序。
- 父 PRD 要求共享连接池、streaming queue、默认并发 3、MediaStore Pictures/PixivFunc 和 progress throttle。
- 该基础必须在作品详情下载操作完成前可用，Ugoira 导出后续复用其队列/媒体提交能力。

## Dependencies

- 08-26-pixiv-network-token-refresh 完成。
- 08-26-android-platform-parity 提供 MediaStore 流式接口。

## Requirements

- R1: 实现应用级 DownloadManager、共享 HTTP connection pool 和可配置并发队列，默认并发严格为 3。
- R2: 响应体直接流向临时/MediaStore pending sink；禁止 readAsBytes、完整大文件 Uint8List 和大 bytes isolate 传递。
- R3: 任务模型至少包含 queued/running/canceling/succeeded/failed/canceled，且 completion 只能发出一次。
- R4: 按作品 ID、页 index、资源 URL 归一化和目标类型去重；文件名、MIME 和相对目录安全规范化。
- R5: 支持排队取消、运行中取消、失败重试、进度节流、未知 content-length 和应用生命周期恢复策略。
- R6: 成功写入 Pictures/PixivFunc 后再发完成 toast/状态；失败或取消必须 abort pending MediaStore item。
- R7: 对 URL host、重定向和 TLS 使用统一网络安全策略，不允许下载器自建证书绕过。

## Acceptance Criteria

- [ ] 默认最多 3 个下载并发，调整设置后新调度遵循上限；同一目标不会重复启动。
- [ ] 大文件测试显示数据流有界，不构造完整文件 bytes；取消会停止网络并清理 pending output。
- [ ] 成功/失败/取消/重试每个任务恰好一个终态和一次完成事件，progress 更新经过节流。
- [ ] 单图、多页 Download All 能按页命名并写入 Pictures/PixivFunc，已有文件策略确定且可测试。
- [ ] 网络重定向到非允许 host、TLS 失败和路径穿越名称被拒绝。
- [ ] 单元/集成测试、analyze、全量 test、debug APK 和 API 36 MediaStore 设备验证通过。

## Out of Scope

- Ugoira ZIP 解码/GIF 合成。
- 后台常驻下载服务或跨重启保证，除非子任务研究证明原版必需。
- Updater APK 安装。

## Risks and Deferred Items

- Android OEM 的 MediaStore/取消行为不同；pending item 清理和重复完成必须在至少一个 API 36 真机验证。

## Source Anchors

- beta56 lib/app/downloader/downloader.dart、lib/models/download_task.dart、lib/pages/downloader/downloader.dart、Android PlatformApi.kt
- 父 PRD R9 与 Android PlatformMediaStore 契约

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
