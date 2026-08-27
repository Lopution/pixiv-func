# 实现 Ugoira 播放与 GIF 导出

## Goal

以磁盘ZIP和有界解码窗口复刻beta56 Ugoira封面、播放/暂停、离屏生命周期和GIF保存体验。

## Confirmed Facts

- beta56将ZIP完整读入Uint8List、全部解压为bytes并把所有frame解码为ui.Image，内存随帧数无界增长。
- 父PRD要求ZIP到temp/disk、bounded frame cache、bounded decoded window和ticker/deadline scheduler。
- 可见行为是cover、中心约70px play、tap load/play、播放tap pause、paused overlay、离屏停止/重新可见恢复、保存GIF。

## Dependencies

- 08-26-illust-detail-viewer与download-manager-mediastore完成。
- 08-26-pixiv-network-token-refresh提供metadata/ZIP安全请求。

## Requirements

- R1: 核验当前ugoira metadata/ZIP schema，使用统一 `NetworkAccessPolicy` 的严格 host/TLS 和 stream 下载到任务私有 temp 文件；兼容网络未完成时仍不得自行实现 fixed IP/代理 fallback。
- R2: 用 typed `UgoiraLimits` 同时限制 archive compressed/uncompressed bytes、entry count、重复/未知/path entry、compression ratio、frame count、单帧 bytes、dimensions 与 pixel budget；frame顺序和delay严格来自metadata并交叉验证。
- R3: frame header/format/dimensions 通过后才允许像素分配；实现有界compressed/frame cache和decoded ui.Image窗口，淘汰时dispose；不得保留全部frame bytes/images。
- R4: 使用ticker/deadline scheduler基于每帧delay推进，处理jank补偿、pause/resume和app lifecycle。
- R5: 初始显示cover与约70px play；tap加载/播放，播放时tap暂停并显示overlay；完全离屏停止，重新可见按原状态恢复。
- R6: 加载、metadata、ZIP、decode、取消、磁盘不足和单帧损坏错误明确，退出清理temp/cache/ticker。
- R7: GIF导出作为 user-visible job 的 post-process，使用流式/有界encoder，经owned pending output/DownloadManager/MediaStore保存；最终 progress 不被节流吞掉，成功后才commit，失败/取消只清理本job拥有的输出。
- R8: 多个Ugoira页面/列表可见时限制并发解码与播放，账号/route切换不泄漏资源。

## Acceptance Criteria

- [ ] 参考Ugoira按metadata delay顺序播放，tap/overlay/offscreen/lifecycle行为符合beta56。
- [ ] 长帧样本峰值decoded frame和buffer不超过配置窗口，淘汰image被dispose。
- [ ] archive/frame limit 在分配完整像素前拒绝超额输入；重复entry、伪造size、极端dimensions、总pixel budget和compression ratio都有测试。
- [ ] zip-slip/zip bomb/metadata mismatch/损坏frame被拒绝且temp清理。
- [ ] 取消/退出/切后台停止网络、ticker和decode；返回可恢复且不重复生成。
- [ ] GIF导出帧序/delay正确，MediaStore成功/失败/取消恰好一个终态。
- [ ] Ugoira metadata/ZIP、封面与导出均能切换到后续 shared Pixiv transport；本任务不私有化 host/IP/代理策略，兼容网络集成前后不出现两套下载出口。
- [ ] analyze、全量test、debug build、性能测试和API36真机播放/导出验证通过。

## Out of Scope

- 导出MP4/WebM。
- 预加载整个推荐流Ugoira。
- 修改原版播放按钮/手势。

## Risks and Deferred Items

- GIF编码可能CPU/耗时高；必须有取消、进度、单批工作预算和资源上限，不能在UI isolate长时间阻塞，也不能为 encoder 一次收集全部 decoded frames。

## Source Anchors

- beta56 lib/pages/illust/ugoira_viewer/*、components/frame_gif/*
- DownloadManager/MediaStore与Illust Detail Ugoira route

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
