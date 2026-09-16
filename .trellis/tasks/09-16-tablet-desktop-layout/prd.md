# 平板与桌面宽屏布局

## Goal

宽断点下的双栏/侧栏/窗口化深化，手机单列行为不回归。

## Requirements

- `app/layout/` 断点体系与 two-pane 容器；详情页宽屏双栏（图+信息/评论）
- 桌面/平板侧栏导航或扩展 rail；窗口尺寸自适应细节（含 Windows/Linux 桌面）
- 卡片网格列数策略与现有 `illustColumnsFor` 归并

## Acceptance Criteria

- [ ] 宽断点详情 two-pane 可读可用；窄屏/手机单列无回归
- [ ] 桌面窗口缩放/最小宽度行为正确；键鼠滚动路径不受影响
- [ ] golden/layout 测试覆盖断点切换

## References

- Shaft：tablet two-pane（README capabilities）；freepiv：自适应导航+自定义标题栏
- 本仓：`app/layout/`、`app/widgets/feed/feed_grid.dart`、`platform_caps.dart`

## Dependencies

建议功能面稳定后做（wave 5）。
