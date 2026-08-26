# 实现下载、Ugoira 与媒体流水线

## Goal

作为中间父任务，协调流式DownloadManager和Ugoira播放/导出两个叶子任务，统一媒体网络、磁盘、MediaStore和生命周期边界。

## Confirmed Facts

- 该范围已拆为download-manager-mediastore与ugoira-player-export。
- DownloadManager是Illust Detail下载模式的前置；Ugoira播放器在Detail完成后接入。
- beta56两条路径都使用大量bytes/旧Android bridge，现代实现必须共享有界流水线。

## Dependencies

- 08-26-pixiv-network-token-refresh与android-platform-parity完成。

## Requirements

- R1: 本中间父任务不直接实现产品代码，只维护两个叶子任务的顺序、共享MediaStore/scheduler/cleanup契约和集成验收。
- R2: 先完成download-manager-mediastore，供illust-detail-viewer的单页/Download All真实使用。
- R3: 在Detail完成后实施ugoira-player-export，并复用DownloadManager/MediaStore而非复制transport或保存逻辑。
- R4: 两个叶子分别规划审批、实现、验证、提交和归档；媒体内存/磁盘基准分别记录。
- R5: 完成前验证详情下载、Downloader页、Ugoira播放/暂停/导出和Settings并发上限的跨功能行为。

## Acceptance Criteria

- [ ] 两个叶子任务均归档，依赖/evidence完整。
- [ ] 单图/多页/Ugoira共享安全transport、MediaStore和终态语义，无重复下载器。
- [ ] 默认并发3、取消/失败cleanup、离屏Ugoira生命周期和GIF保存跨功能通过。
- [ ] 大文件/多帧性能测试无完整大媒体bytes和无界ui.Image列表。
- [ ] 全量analyze/test/debug build及API36真机媒体集成验证通过。

## Out of Scope

- 后台常驻媒体服务，除非叶子研究证明原版必要。
- 视频Live。
- Updater APK安装。

## Risks and Deferred Items

- 媒体任务容易因内存/取消竞态产生设备特定失败，父验收必须使用真实大样本。

## Source Anchors

- beta56 app/downloader、illust/ugoira_viewer、components/frame_gif、Android PlatformApi
- 两个叶子task artifacts与父PRD R9

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
