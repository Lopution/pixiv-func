# 执行计划：平板与桌面宽屏布局

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(layout): expanded 断点接线与 TwoPane 容器`——`useTwoPaneDetail` + `app/layout/two_pane.dart` + 断点边界 widget 测试
- [ ] `feat(layout): 详情页宽屏双栏`——illust detail ≥1200 左图右信息 + 窄态原样 + 键盘/滚轮/scrollbar 复核 + widget 测试
- [ ] `feat(layout): 桌面展开侧栏与窗口约束`——≥1200 extended rail（三档导航）+ 桌面最小窗口尺寸 + 切换动画 + widget 测试
- [ ] `chore(09-16): tablet-desktop-layout journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test`（layout/detail/home 相关测试 + 全量）
- `git diff --check`

## 回滚点

TwoPane 是附加容器——详情页摘掉即回单 scroll；断点函数默认值回退即关双栏；rail extended 单点可回。

## 风险

- 详情页改造面最大（385 行单 scroll 拆 pane）→ 窄态路径零改动原则，双栏代码走条件分支而非重写。
- golden 基线若不存在 → widget 结构断言替代，不引入新 golden 基建。
- extended rail 在 1200–1400 中等宽度可能挤内容 → 测试覆盖边界，必要时 rail 宽度计入断点（two_pane 判据 = 内容区宽度而非窗口宽度）。
