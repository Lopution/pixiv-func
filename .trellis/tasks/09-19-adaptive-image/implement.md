# implement — adaptive-image

一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] 1. perf(cache): 磁盘缓存上限 200→1500 条 + PriorityFileService 双闸门（可见 8 / prefetch 3，标记头 x-pixiv-func-prefetch 出队即剥）
- [ ] 2. feat(network): 幂等图片请求冷启动竞速——无路由/组记忆时 top-2 档并行 send，先达者写记忆，败者 drain 回收；双败落回串行梯子
- [ ] 3. feat(network): 路由 kind 按 networkIdentity 持久化到 SharedPreferences——重启 warmUp 播种组偏好，首请求免探测
- [ ] 4. feat(network): connectivity_plus 监听——网络类型变化即 advanceNetworkRevision(identity=连接类型) 清记忆/池/冷却
- [ ] 5. feat(motion): feed 入场动画挂载触发→首次曝光触发 + fling 速度门 + index 上限放宽
- [ ] 6. perf(viewer): 已解码低档垫底(3a)+相邻页 medium 预取(3b)——不动 hero/转场/淡入路径
- [ ] 7. perf(download): 下载命中图片磁盘缓存直接物化到 sink
- [ ] 8. perf(network): feed 数据落地对胜者路由预热连接（HEAD 节流）
- [ ] 9. feat(settings): ImageSourceMode.auto——候选源缩略图竞速选线，胜者按 networkIdentity 持久化，失败降级
- [ ] 10. feat(settings): 网络主页面语义档 + 生效路由展示 + 第三方可达性提示
- [ ] 11. feat(debug): dev-only 滚动帧探针（FrameTimings 采集+导出）
- [ ] 12. chore(task): 收尾簿记（勾选、journal、归档）
