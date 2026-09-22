# W4 风险与未决项

## R1. 双击缩放 vs 单击切 chrome 的手势冲突（中）
InteractiveViewer 不自带双击缩放；自实现 `onDoubleTap` 时单击需延迟等待双击判定（~250-300ms），否则单击切 chrome 会在双击第一击时误触发。方案：单击延迟判定，或 chrome 切换改挂“点击非图片空白区/专用按钮”。需 widget test 覆盖单击-不触发-双击-再单击的序列。

## R2. PopScope 优先级与路由 replace 的交互（中）
查看器翻页走 `replaceImageViewerPage`（routes.dart L144）会 `replace` 路由——`PopScope.canPop` 回调在 replace 后的新 route 上要重新生效；若用 `canPop`+`onPopInvoked` 拦截返回做“先复位缩放”，需确认 replace 生成的新 route 也携带同一拦截。测试：放大→滑动→replace→按返回，应复位缩放而非退路由。

## R3. “继续阅读”语义边界（高 · 决策点）
`WatchlistReadCursor` 是“见过最新 ID”游标（watchlist_store.dart L249-275，只前进），不是阅读位置。若产品要真章节进度，需新 schema（account+series → lastReadContentId）+ 迁移 + 回滚文档——**建议不放进本 leaf**，只用“开始阅读/上一话·下一话/返回第 n 话（内存态）”诚实语义；若 W4 验收硬性要求持久续读，需先回父任务改范围。

## R4. 下载模式退出手势的可达性（中）
现退出靠“点非图片空白处”——全屏图片作品没有可见空白，用户可能找不到退出路径。显式 chrome（取消按钮）可解，但需同时决定“查看器/详情双入口下模式状态放哪”：`_downloadMode` 是 `IllustDetailPage` 本地 state，若查看器也要页选择下载，模式态要么提升到 route extra/provider，要么查看器内独立实现——建议各自本地、语义对齐（不共享状态对象）。

## R5. compact header 与 AppBar 动作重叠（低-中）
新增常驻 compact header 后，书签/分享可能与 AppBar L162-198 动作重复。原则：同动作同图标同结果可共存（冗余不伤一致性），但每多一处就多一处状态同步面；建议 header 只放“下载全部+信息跳转”，书签留在 AppBar。

## R6. tag 菜单新增“复制/搜索”与现有屏蔽模式并存（低）
长按从“切模式”改为“弹菜单”改变了既有肌肉记忆；菜单内保留“批量屏蔽模式”入口可保能力不回退。需在 changelog/实现稿标注行为变更。

## R7. Ugoira 导出的登录态/上下文前置（低）
`_export` 依赖可用登录态与下载提交上下文；查看器/下载模式复用时若上下文缺失要显式报错（已有抛错语义），不要静默。

## R8. 触觉包装的 settings 接线（低）
PixEz 式 `try{读开关}catch→false` 需要项目内等价物：`settingsController` 是否已有 haptic 开关位？没有则需新增设置项（schema/默认值/迁移）——属本 leaf 内小项，但要先于 wrapper 落地。

## R9. `SmoothWheelScroll` controller 归属（低）
窄屏跳转锚点需要拿到 scroll controller；非桌面时 controller 回落 `PrimaryScrollController`（smooth_wheel_scroll.dart L150-157），锚点 ensureVisible 用目标 context 而非 controller 可规避该问题——**推荐 ensureVisible 方案**，不动 controller 所有权。

## R10. 查看器无实体上下文（低）
`ImageViewerPage` 只收 urls（构造 L24-45）；做“保存当前页/信息面板”需要 illustId+page → 经 routes.dart builder 里已有的 `entity`（L126）把 `illustId`/`pageIndex`/必要实体传进 widget——签名扩展属本 leaf 内改动，注意 `ImageViewerRouteExtra` 已带 entity 快照，直接透传即可，勿新建并行数据通道。

## 未运行时验证项（implementation 阶段必须补）
- 双击/单击序列、PopScope 三级优先、下载模式 chrome 在横竖屏与双栏下的呈现；
- Ugoira 导出在弱网/未登录态的错误文案；
- `replaceImageViewerPage` 后系统返回键与 AppBar back 的一致性；
- 读屏：查看器各动作 label、下载模式角标语义、compact header 焦点顺序。
