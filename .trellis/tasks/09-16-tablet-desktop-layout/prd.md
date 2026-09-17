# 平板与桌面宽屏布局

## Goal

统一断点体系下的宽屏深化：详情页双栏、桌面侧栏导航、窗口自适应——手机单列行为零回归。

## Confirmed Facts

- `app/layout/app_breakpoints.dart` 已有阶梯：`<600` compact（底部 NavigationBar）、`≥600` medium（NavigationRail）、`≥1200` expanded 已声明但仅注释「可约束可读列宽」，未接线任何双栏。
- `illustColumnsFor(crossAxisExtent)` 已按实际横轴宽度自然加列（手机 2 列起步），全 feed 共用——列数策略已是单点，「归并」= 保持其唯一来源并文档化。
- `home_page.dart` ≥600 切 NavigationRail，settings 在 rail 尾部 peer 位。
- `illust_detail_page.dart` 是单 `CustomScrollView`（图页 sliver → 信息 → 评论），385 行——双栏改造面明确：左图右文两独立滚动区。
- `PlatformCaps.isDesktop`：Windows 唯一支持桌面目标；`isWindows` 分支可测试注入。
- Shaft 平板：详情/列表双栏 + 侧栏；freepiv：自适应导航 + 自定义标题栏。
- AGENTS 规范：窗口主纵向滚动条在窗口最右缘；布局切换预留空间/动画过渡。

## Requirements

- `AppBreakpoints` 接线 `expanded≥1200`：新增 `useTwoPaneDetail(width)`；断点语义集中在此文件，页面不得自行散落宽度判断。
- `TwoPane` 容器（`app/layout/`）：左 pane 定比（默认 55:45，可配置）+ 分隔线；窄断点退化为单子组件——详情页用它而非手写 Row 判断。
- 详情页双栏：`≥1200` 时左=图片 pager（占满 pane 高、页码指示保留），右=信息+评论独立滚动列；`<1200` 完全保留现有单 scroll 结构。
- 桌面导航：`≥1200` 时 NavigationRail 展开为**带标签常驻侧栏**（`extended` rail 语义）；600–1200 保持紧凑 rail。
- 窗口细节：桌面最小窗口约束（`PlatformCaps.isDesktop` 下最小宽 360）；缩放跨过断点时 rail/双栏动画或瞬时切换不留布局垃圾；主滚动条保持窗口最右缘。
- `illustColumnsFor` 维持唯一列数策略；双栏详情中相关推荐/同作者网格仍读它。
- 键鼠路径不回归：`SmoothWheelScroll`、键盘左右翻页在双栏下各自作用于所在 pane。

## Acceptance Criteria

- [ ] `≥1200` 详情页左图右信息可读可用；图片 pane 翻页、信息 pane 滚动互不干扰。
- [ ] `<1200`（含全部手机）详情单列结构与交互零变化。
- [ ] `≥1200` 导航为展开侧栏；600–1200 紧凑 rail；<600 底部栏——三档切换无闪烁/错位。
- [ ] 桌面拖窗口跨断点缩放无布局错误；最小宽度约束生效。
- [ ] golden/layout 测试覆盖 599/600/1199/1200 边界与双栏结构；既有手机 golden 不变。
- [ ] 键盘/滚轮在双栏下正常工作；scrollbar 在窗口最右缘。

## Out of Scope

- 自定义标题栏（freepiv 有；属窗口 chrome 独立改造）。
- 除详情页外的双栏化（novel 详情、series 页、搜索页后续按需扩展）。
- 可拖拽 pane 分隔条/用户记忆宽度比。
- 多窗口/自由窗口（Android 桌面模式适配）。

## Dependencies

建议最后做（wave 5）——功能面稳定后布局收口。无代码硬依赖。

## 默认决策（待用户确认）

| 项 | 默认 | 备选 |
|---|---|---|
| 双栏断点 | ≥1200（复用 expanded）| 新阈值（如 1000/1440）|
| 首批双栏页 | 仅 illust 详情 | +novel 详情/series 页 |
| ≥1200 导航 | extended NavigationRail | 常驻 NavigationDrawer 侧栏 |
| pane 比例 | 固定 55:45 | 可拖分隔条 |
