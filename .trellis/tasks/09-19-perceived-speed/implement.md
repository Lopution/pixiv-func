# 09-19-perceived-speed — 实施清单

一个勾一个 commit，提交信息即条目文本。每步后 `flutter analyze` + 相关测试。

- [ ] 1. feat(motion): hero shuttle 圆角按 flight progress 插值——卡片端 12 → 详情端 0，两端形态连续不再有落地跳变
- [ ] 2. feat(nav): 底栏下滑隐藏——ScrollNotification 累计过 slop 触发 SizeTransition 收高，上滑/切分支即回，隐藏态重发布 hero 底边
- [ ] 3. feat(settings): 设置页分组重排（内容入口独立成组）+ 用户面文案去术语（SNI/DoH/ECH/梯子只留在高级页）
- [ ] 4. feat(debug): 关于页连点解锁开发者入口——帧探针脱离 kReleaseMode 门控
- [ ] 5. feat(network): auto 图源设默认 + 竞速改真实小图吞吐判定
- [ ] 6. perf(feed): 预取窗口 12→24、prefetch 车道 3→4、fling 停止即补当前视口
- [ ] 7. perf(detail): 点卡瞬间对详情档发预热请求
- [ ] 8. feat(settings): 预览画质自适应——低吞吐网络自动降 medium 档（竞速实测值驱动）
- [ ] 9. chore(task): 收尾簿记（勾选、journal、归档）
