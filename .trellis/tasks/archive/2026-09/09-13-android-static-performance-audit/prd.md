# Android static performance audit

## Goal

在不连接真机、不改变产品代码的前提下，定位 Android 滑动和 Hero 动画掉帧的高概率根因，解释为什么 Windows 构建不容易暴露同一问题，并用其它 Pixiv 客户端的实现做有边界的对照。

## Background and confirmed constraints

- 用户已确认 Android 高刷新率生效；本任务不把刷新率切换或 `flutter_displaymode` 当作根因。
- 当前没有可用真机，不能声称已经取得 Android 帧时间、Skia/Impeller timeline、内存曲线或实际掉帧比例。
- 当前工作树已有用户/前序性能修改，审计必须同时记录 `HEAD` 与工作树行为，不能把既有修改误报成审计期间新增修改。
- Windows 的鼠标滚轮路径与 Android 触摸滑动路径不同；必须分别分析，不能以 Windows 流畅推断 Android 路径无问题。

## Requirements

1. 追踪共享 feed 从触摸滚动到 sliver 布局、图片 provider/解码、缓存和栅格绘制的静态调用链，覆盖推荐、排行、新作、搜索、历史和用户 feed。
2. 分离分析四类成本：滚动期间的 build/layout、图片解码/缓存、栅格/裁剪合成、Hero/路由动画；每个结论标注证据等级（代码事实、强静态嫌疑、待运行时验证）。
3. 检查 Android 与 Windows 的平台分支、DPR/解码尺寸、缓存容量、预构建范围、滚动请求策略和动画叠加，重点核对 `PixivImage.decodeWidthFor`、`kFeedCacheExtent`、`SliverMasonryGrid`、`Hero` shuttle、`HeroRectClip` 与路由转场。
4. 对照至少三个固定版本的其它 Pixiv 客户端，其中至少一个 Flutter 客户端和一个原生 Android 客户端；只记录可由源代码支持的实现差异，不把“没有掉帧数据”写成性能结论。
5. 给出按优先级排序的根因候选、Windows 差异解释、最小后续验证/修复建议，并明确哪些项目必须有真机或 profile trace 才能定案。

## Out of scope

- 真机运行、ADB、帧率/帧时间测量、GPU profile、内存压力测试和截图对比。
- 本任务内修改 Dart/Android 产品代码、调节刷新率、替换图片库或重写 feed/导航架构。
- 将网络服务端延迟、API 数据量或账号状态当作本次渲染掉帧的根因。

## Acceptance Criteria

- [x] 产出 `.trellis/tasks/09-13-android-static-performance-audit/research/static-audit.md`，包含文件/符号定位、机制解释、证据等级、Windows 对照、未验证项和建议顺序。
- [x] 报告至少给出一个滚动 P0/P1 候选和一个 Hero P1 候选，并明确说明“高刷新率已生效，不是刷新率开关问题”。
- [x] 报告覆盖当前工作树与 `HEAD` 的关键差异，尤其说明解码尺寸上限是否一致，以及不能把工作树中的既有性能修改归因于本次审计。
- [x] 报告引用至少三个其它 Pixiv 客户端的固定 commit/tree 链接；其中包含 Pix-EzViewer 的按 cell 宽度解码和滚动中暂停图片请求证据，并说明其不可直接等同于 Flutter 行为。
- [x] 完成只读质量检查：`flutter analyze --no-pub`、本任务相关的现有测试（如可运行）和 `git diff --check`；结果如实记录，不宣称设备验证。全量测试中的既有契约失败也已记录。
- [x] 不修改产品源文件，不删除或覆盖工作树中与本任务无关的已有改动。

## Deliverable boundary

本任务的完成定义是“可审查的静态根因报告和后续验证顺序”，不是“已经在 Android 设备上证明修复”。任何实际修复应在报告获得用户确认后另立实现阶段。
