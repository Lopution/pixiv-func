# 自适应图片管线（adaptive-image）

## Goal

冷启动图片加载按"档位竞速+路由记忆持久化"消除首图试错；缓存层扩容量并给可见图/prefetch 分优先级；新增 `auto` 图片源按实测速度选线；viewer 渐进显示与下载缓存复用降低感知等待；feed 入场动画改曝光触发；dev-only 帧探针支撑滚动卡顿定位。

## Requirements

- L1 冷启动：幂等图片请求在无路由记忆时对 top-2 候选档并行 send，先达者写路由记忆，败者静默回收；路由 kind 按 `networkIdentity` 持久化（不存裸 IP）；connectivity 变化 → `advanceNetworkRevision` 清记忆/池/冷却
- L2 缓存：`maxNrOfCacheObjects` 200→1500；prefetch 请求打标记头，FileService 双闸门（可见 ≤8、prefetch ≤3）共享同一磁盘 store
- L3 auto 源：`ImageSourceMode.auto` 可选档（默认仍直连）；对候选源做真实缩略图竞速（先达 2xx 者胜），胜者按 `networkIdentity` 持久化，持续失败自动降级
- L4 感知：feed 落地预热胜者路由连接；viewer 用已解码低档垫底（3a，零额外流量）+ 相邻页 medium 预取（3b）；下载命中图片缓存直接物化到 sink
- 动效：feed 入场动画由"挂载"改"首次进入视口"触发 + fling 速度门（快速滚动不播），`played` 幂等保留
- L5 设置收敛：网络主页面语义档 + "当前生效路由"展示 + 第三方可达性提示
- 探针：dev-only `addTimingsCallback` 帧探针记录滚动期 build/raster 耗时

## Acceptance Criteria

- [ ] 冷启动首图不再串行走完整个梯子；竞速只在无记忆时发生，败者请求不产生副作用
- [ ] 重启应用后同网络下首请求直接命中上次胜者档
- [ ] wifi↔移动网络切换清空路由记忆/连接池
- [ ] 缓存 1500 条；慢滑 prefetch 不阻塞可见图加载
- [ ] `auto` 源选线结果持久化且可手动切走；失败自动降级直连
- [ ] viewer 翻页先见略糊图后升级清晰；下载已缓存图片不再重复传输
- [ ] 慢滑每张新卡滚入播放上浮动画；fling 不播；回滚不重播
- [ ] 探针仅 dev 生效，可导出滚动帧耗时文本
- [ ] `flutter analyze lib test` 0 issues；全量测试通过
