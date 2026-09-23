# External: Material 3 自适应布局与内容限宽惯例

调研日期：2026-09-22。用途：为 §4.9/§5.5 的限宽、行长、sheet/dialog 语义提供外部依据。

## M3 断点（m3.material.io/foundations/layout/breakpoints）

| Breakpoint | 宽度 | 设备 |
|---|---|---|
| Compact | <600dp | 手机竖屏 |
| Medium | 600–839dp | 平板竖屏/展开折叠机竖屏 |
| Expanded | 840–1199dp | 手机横屏/平板横屏/桌面 |
| Large | 1200–1599dp | 桌面 |
| Extra-large | ≥1600dp | 超宽桌面 |

M3 对 compact 的弹层惯例是 full-screen dialog / bottom sheet；更大断点用 simple/dialog、side sheet、menu 等非全宽形态。**本仓库 AppBreakpoints 只有 600/1200 两档，与 M3 的 600/840/1200/1600 不同——按 §5.5 不得以 M3 数值替换仓库断点；M3 档位只作内容角色的论证依据。**

## 内容限宽惯例

- Android adaptive 指南（developer.android.com/design/ui/mobile/guides/layout-and-content/adapt-layout）：**"Set a max width on content and components to prevent stretching full width"**，宽屏上组件应变 presentation（bottom sheet→side sheet 等）而不是拉伸。
- M3 lists 指南（m3.material.io/components/lists/guidelines）：理想行长 **40–60 字符**，大屏最多 ~120 字符，接近上限时应加大行高。
- 排版实践（多个 M3 参考实现）：正文 50–75 字符/行；内容最大宽度典型值 **840–1040dp** 并居中。
- 任务要求的 form ~600–840dp max 与 article ~60–80 字符行长与上述一致。换算：bodyMedium 14sp 下，~680–760dp 内容宽 ≈ 60–80 拉丁字符 / ~45–55 CJK 字符一行。Spotlight 以中文为主，**建议正文限宽落在 ~640–760dp 区间**（按字符数反推，最终值需以真实文章目测微调并写明理由）。
- 仓库已有先例：welcome_page 的 `maxWidth:520`（form/引导角色）。

## 弹层形态

- compact：modal bottom sheet / fullscreen dialog。
- medium+：simple dialog、side sheet、dropdown menu——同一信息同一动作，容器换形态。
- §4.9 对 W9 的要求正好对应："手机与宽屏保持相同信息顺序和动作语义，不强求相同位置"。

## Flutter 侧实现事实

- `SingleChildScrollView` + `Center` + `ConstrainedBox(maxWidth)` 是仓库 welcome 页已用的限宽模式；列表型页面等价做法是 `SliverPadding`/`Center+ConstrainedBox` 包 sliver，或在 ListView 外层 `Align+ConstrainedBox`。
- 注意 `MediaQuery.sizeOf(context).width*.1` 这类比例 padding 在超宽屏无限增长，不构成限宽。

## 参考链接

- https://m3.material.io/foundations/layout/breakpoints
- https://developer.android.com/design/ui/mobile/guides/layout-and-content/adapt-layout
- https://developer.android.com/develop/adaptive-apps/guides/use-window-size-classes
- https://m3.material.io/components/lists/guidelines（Line length）
