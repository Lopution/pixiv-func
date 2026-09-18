# 设计：平板与桌面宽屏布局

## 1. 断点与容器

```
lib/app/layout/
  app_breakpoints.dart   — 加 useTwoPaneDetail(w)=w>=expanded；语义注释唯一来源
  two_pane.dart          — TwoPane(left, right, ratio, breakpoint)
```

- `TwoPane`：`LayoutBuilder` 拿约束宽度；`>=breakpoint` → `Row`（左 `flex:55` 右 `flex:45`，中间 1px 分隔 `VerticalDivider`）；否则只渲染 `left`？不——详情页语义是「宽=两个并列 pane，窄=单子」：`TwoPane` 接受 `narrow` 构造参数指定窄态渲染体（详情的现有单 scroll）。pane 边界即滚动边界。
- 断点判定一律走 `AppBreakpoints` 静态方法；页面零散落宽度判断视为违规（spec 可写）。

## 2. 详情页改造

`illust_detail_page.dart` 现状：单 `CustomScrollView`（image slivers → author/info → caption → comments sliver）。

```
>=1200: TwoPane(
  left:  ImagePane — 图片 pager 占满 pane（PageView/画廊现有组件），页码指示、
         键盘左右、滚轮缩放语义不变
  right: InfoPane — CustomScrollView：作者条/标题/caption/标签/操作/评论 sliver
         （现单 scroll 里「图之后」的全部内容原样搬入）
)
<1200:  现有单 CustomScrollView 原样
```

- 评论加载策略不变（现有懒加载/sliver 分页），右 pane 独立滚动到尾触发 loadMore。
- 图片 pane 顶部操作条（下载/收藏/分享）保持在右 pane 信息区顶部——与窄屏同一操作位置，避免双份实现。

## 3. 导航

`home_page.dart`：`AppBreakpoints` 三档——

| 宽度 | 形态 |
|---|---|
| <600 | 底部 NavigationBar（现状）|
| 600–1199 | NavigationRail 紧凑（现状）|
| >=1200 | NavigationRail `extended: true`（图标+标签常驻）|

- settings 入口位置三档一致（rail 尾部 peer）；切换档用 `AnimatedSwitcher`/尺寸动画避免跳动，遵循「布局变化预留空间或动画过渡」。

## 4. 窗口细节

- 桌面入口（`main`/window init）设最小尺寸 `minWidth 360 × minHeight 480`（仅 `PlatformCaps.isDesktop`）。
- 滚动条：详情右 pane 滚动条贴 pane 右缘（即窗口右缘）；feed 页 scrollbar 现状复核，违规则修。
- `illustColumnsFor` 不动；文档注释加「唯一列数策略」一句。

## 5. 测试

- golden/layout：详情页 599/600/1199/1200 四档 pump 断言 pane 结构（`TwoPane` 存在性、左右 pane finder）；导航三档断言 rail extended 属性。
- 交互：双栏下图片 PageView 翻页 + 右 pane 滚动 loadMore 互不干扰；键盘事件分发。
- 无 golden 基建则 widget 断言布局结构（项目现有 layout 测试范式）。

## 6. 不做的事

- 自定义 titlebar、可拖分隔条、其他页双栏化、多窗口。
