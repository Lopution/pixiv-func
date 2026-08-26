# 复刻作品详情与全屏查看器

## Goal

从推荐卡片进入真实作品详情，完整呈现单图/多页信息、下载模式和 beta56 全屏查看手势。

## Confirmed Facts

- 当前仓库没有 Illust Detail route、页面、viewer 或下载模式。
- beta56 lib/pages/illust/illust.dart、controller.dart 和 scale/scale.dart 展示详情、长按下载模式及全屏图片交互。
- 父 PRD要求单图/多页、R18/AI/Ugoira/page count、作者、tags、summary、点击 viewer、长按下载模式和 0.9x–6.0x zoom。

## Dependencies

- 08-26-recommended-feed-paging 完成。
- 08-26-download-manager-mediastore 提供可真实执行的单页/全部页下载请求。
- 08-26-android-platform-parity 提供返回、MediaStore 与 typed route 基础。

## Requirements

- R1: 支持按作品 ID 导航；先显示 IllustStore snapshot，再请求 detail 并 merge，ID 不存在/已删除/受限时显示明确状态。
- R2: 详情复刻单图、多页、R18、AI、Ugoira、page count badges、作者入口、tags、caption/summary 和原版信息顺序。
- R3: 点击图片进入水平 Fullscreen Viewer，标题显示 n / total，zoom 范围严格为 0.9–6.0，并保持页 index/返回状态。
- R4: 长按图片进入下载模式；多页每页右上有独立下载动作，顶部提供 Download All，动作必须接入真实 DownloadManager 状态而非空操作。
- R5: tag tap 提供第一条链需要的最小真实 tag 结果路由，并设计为后续 Search 复用；tag long press 进入 tag block mode，不实现旧教程的复制 tag。
- R6: 处理图片失败、detail API 失败、删除/限制作品、多页部分失败、旋转/后台恢复和重复导航。
- R7: Ugoira 本任务只显示 badge、封面和进入播放器的 typed route；真正播放/导出由 ugoira task 完成前不得显示伪播放成功。
- R8: 详情和 viewer 使用统一 Hero/转场与 Replica 右进左出/返回节奏，不引入 Material 默认导航漂移。

## Acceptance Criteria

- [ ] 从 Recommended 卡片点击进入同一 ID 详情，先后数据 merge 不导致卡片/详情字段或收藏状态倒退。
- [ ] 单图、多页及各 badge、作者、tags、summary 在参考样本上符合 beta56。
- [ ] Viewer 水平翻页、n/total、0.9–6.0 zoom、初始页和返回恢复通过 Widget/手势测试。
- [ ] 长按进入下载模式，单页与 Download All 产生真实队列任务并显示正确状态；失败可重试。
- [ ] tag tap 返回真实 tag 结果，long press 进入 block mode；无复制 tag 行为。
- [ ] API error、受限/删除、图片失败、旋转与生命周期场景无崩溃/空白/伪成功。
- [ ] analyze、全量 test、debug build 和真实 Android 详情/viewer/下载模式验证通过。

## Out of Scope

- 完整 Search 首页与过滤器。
- Bookmark mutation 交互。
- Ugoira 解码播放/GIF 导出。
- Comments 页面。

## Risks and Deferred Items

- beta56 依赖旧图片 viewer；现代实现需要自行限制 gesture boundary，必须用手势测试和真机验证缩放/返回冲突。

## Source Anchors

- beta56 lib/pages/illust/illust.dart、controller.dart、scale/scale.dart、related/source.dart
- beta56 components/illust_previewer；IllustStore、DownloadManager 和 Replica route 契约

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
