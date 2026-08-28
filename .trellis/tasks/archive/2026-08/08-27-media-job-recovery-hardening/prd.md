# 补强下载与 Ugoira 任务恢复

## Goal

补强下载与 Ugoira 的 task-group、提交快照、临时输出所有权、进程重启和 MediaStore pending item 恢复语义，避免幽灵 running task、重复后处理、跨账号文件污染和无界媒体资源。

## Scope and current facts

- 目标历史任务：`08-26-download-manager-mediastore`；相关在途任务为 `08-26-ugoira-player-export`，当前 Ugoira 产品实现完成前本叶子不得开始或修改其相同文件边界。
- 现有范围包括 `DownloadManager`、`DownloadTask`、coordinator/sink、Pixiv download transport 和 MediaStore channel；以当前代码和测试为准。
- 下载和 Ugoira 都必须消费 shared network policy；本叶子不搭建反代、不改变 beta56 的单页/全部页选择。

## Requirements

- R1：group/job/subtask 使用明确状态机和 immutable submission snapshot（account、illust/page/frame、destination、format、policy revision）；每个 job 只有一个终态。
- R2：临时文件、目录、MediaStore pending URI 和后处理输出都记录 owner；成功 finalize、失败 cleanup、取消和超限退出必须 exactly-once 且幂等。
- R3：进程重启时只恢复可证明仍拥有且可重试的 pending；running/unknown 状态按显式策略转为 retryable 或 failed，不把旧账号任务隐式重放。
- R4：Ugoira frame 数、像素、压缩和内存预算有界；中间帧和 GIF/视频转换支持取消，不能重复 finalize 或泄漏 pending item。
- R5：下载失败区分认证、限流、网络、存储空间、权限、解码和资源超限；Retry-After/用户操作可观察，非幂等后处理不自动无限重试。
- R6：MediaStore API 级别、FileProvider/共享 URI 和 scoped storage 错误保持安全失败；不把临时路径暴露给其他应用。

## Acceptance Criteria

- [ ] 单页、全部页、Ugoira group 的提交 snapshot 在账号切换或设置改变后仍可追溯，迟到结果不能写新账号或新目的地。
- [ ] 成功、失败、取消、重启恢复和重复回调都只产生一次终态；pending MediaStore 项目按 owner 清理或恢复。
- [ ] 崩溃/进程重启 fixture 覆盖 running、pending、finalizing 和 orphaned 临时输出；恢复策略和用户可见状态明确。
- [ ] 大媒体资源在 frame/pixel/memory budget 内，取消和错误不留下临时文件或 pending URI；Android API 36 至少有平台验证计划。
- [ ] 本叶子只在 `08-26-ugoira-player-export` 完成后启动；归档目录无 diff，integration release 消费恢复 evidence。

## Dependencies and Out of Scope

- 依赖：`08-26-restricted-compat-network`、`08-26-ugoira-player-export`；后者完成前保持 blocker。
- 不负责重新实现 Ugoira 播放器、增加云端任务队列、修改服务器协议或引入无上限缓存。
