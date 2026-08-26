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

- R1: 核验当前ugoira metadata/ZIP schema，使用严格host/TLS和stream下载到任务私有temp文件。
- R2: ZIP解析防zip-slip、重复/未知entry、压缩炸弹、异常frame count/size；frame顺序和delay严格来自metadata并交叉验证。
- R3: 实现有界compressed/frame cache和decoded ui.Image窗口，淘汰时dispose；不得保留全部frame bytes/images。
- R4: 使用ticker/deadline scheduler基于每帧delay推进，处理jank补偿、pause/resume和app lifecycle。
- R5: 初始显示cover与约70px play；tap加载/播放，播放时tap暂停并显示overlay；完全离屏停止，重新可见按原状态恢复。
- R6: 加载、metadata、ZIP、decode、取消、磁盘不足和单帧损坏错误明确，退出清理temp/cache/ticker。
- R7: GIF导出使用流式/有界encoder，经DownloadManager/MediaStore保存；保存状态、toast和失败cleanup与下载语义一致。
- R8: 多个Ugoira页面/列表可见时限制并发解码与播放，账号/route切换不泄漏资源。

## Acceptance Criteria

- [ ] 参考Ugoira按metadata delay顺序播放，tap/overlay/offscreen/lifecycle行为符合beta56。
- [ ] 长帧样本峰值decoded frame和buffer不超过配置窗口，淘汰image被dispose。
- [ ] zip-slip/zip bomb/metadata mismatch/损坏frame被拒绝且temp清理。
- [ ] 取消/退出/切后台停止网络、ticker和decode；返回可恢复且不重复生成。
- [ ] GIF导出帧序/delay正确，MediaStore成功/失败/取消恰好一个终态。
- [ ] analyze、全量test、debug build、性能测试和API36真机播放/导出验证通过。

## Out of Scope

- 导出MP4/WebM。
- 预加载整个推荐流Ugoira。
- 修改原版播放按钮/手势。

## Risks and Deferred Items

- GIF编码可能CPU/耗时高；必须有取消、进度和资源上限，不能在UI isolate长时间阻塞。

## Source Anchors

- beta56 lib/pages/illust/ugoira_viewer/*、components/frame_gif/*
- DownloadManager/MediaStore与Illust Detail Ugoira route

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
