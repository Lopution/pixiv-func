# W4 实现草案：artwork-viewer-series-flow

> 状态：研究阶段的实现决策草案，供 trellis-implement 落地时取舍。每条标注对应设计 §4.4 条目。所有引用行号见各 codebase-*.md。

## 0. 设计 §4.4 条目 → 落点映射

| §4.4 条目 | 落点 | 复用/新建 |
|---|---|---|
| 移动端长图信息可达；桌面双栏等价 | `illust_detail_page.dart` 窄屏分支 + InfoBlock | 新建跳转锚点（§1 决策） |
| 普通下载可见入口；批量页选择=显式模式（标题/计数/完成/取消） | `_downloadMode` 扩展为显式 mode chrome | 改造现有模式位 + AppBar/底部条 |
| tag 搜索/复制/屏蔽直接菜单 | `TagChip.onLongPress` → 菜单 | 新建 `showAppBottomSheet` 菜单，保留屏蔽模式 |
| 查看器 chrome/缩放/页码/保存分享信息/空态/键鼠等价 | `image_viewer_page.dart` 全面扩展 | 改造 |
| Ugoira 保存+状态一致入口 | `ugoira_viewer.dart` overlay 导出钮 + 状态文案 | 改造 |
| 系列“继续阅读”诚实语义 | `illust_series_page.dart` + 详情系列卡片 | 见 §4 决策（不建假进度） |
| 触觉唯一 owner，首消费者=下载/保存 | `lib/app/haptics/`（新建）+ 下载触点接入 | 新建 |

## 1. 信息可达策略：跳转锚点 vs 常驻 compact 标题区 —— 决策

### 方案 A（PixEz 式）：图片区内嵌 `expand_less` 锚点
- 做法：在窄屏 imageSlivers 首图 overlay 加一个 `Icons.expand_less` 圆形按钮，`Scrollable.ensureVisible(metaAnchorContext)` 滚到 InfoBlock。
- 优点：实现最小（一个 `GlobalKey` 挂 metaSlivers 首个 sliver 的 `SliverToBoxAdapter` 包裹，或给 InfoBlock 的 `key`）；不占用常驻屏幕面积；与 PixEz 行为对齐（external-pixez-viewer.md §2）。
- 缺点：**可发现性差**——用户要知道箭头含义；进入锚点前仍看不到标题/作者；滚动回顶部后无“信息在哪”的持续提示；目标随图片数变化（长图作品 ensureVisible 要滚很远，动画时长需 clamp）。

### 方案 B：常驻 compact 标题区（SliverAppBar floating/pinned 变体）
- 做法：窄屏 `CustomScrollView` 加一个 pinned 的 compact header（标题一行 + 作者 + 页数 + 核心动作图标），滚到 InfoBlock 后自动隐藏或变形。
- 优点：§4.4 原文就是“标题/作者/页数/核心动作在首图附近可见”——常驻 header 语义最贴；可发现性天然满足；动作（书签/下载/分享/评论）始终可达。
- 缺点：侵占图片面积；与现有 AppBar（L154-200 已有 share/bookmark/download-all）职责重叠——需要把动作收敛到一处，否则出现双份 share/bookmark；pinned sliver 与 `IllustDetailPagerPage` 的横向 PageView 共存时手势不冲突（纵轴 vs 横轴），但 hero flight 期间 pinned 元素要提前就位，过渡需谨慎。

### 决策建议：**B 为主、A 为辅**
- §4.4 验收措辞是“首图附近可见”+“信息/评论直达”——单有 A 不满足“首图附近可见标题/作者/页数”，单有 B 则直达成本仍是一屏高按钮。
- 落地形态：窄屏在 AppBar 之下做 **pinned compact header**（一行标题 ellipsize + 作者 + `n/共N页` + 书签/下载动作），InfoBlock 保持 PR#50 完整布局不变；compact header 内放一个“信息/评论”滚动动作（即方案 A 的 ensureVisible），双栏布局不出现该 header（右栏已是等价信息面）。
- 关键约束：compact header 的收藏/下载动作必须与 AppBar/卡片同名同结果（§4.1 一致性）；下载动作在 header 中建议放“下载当前作品全部”而非页级——页级仍在图片长按/下载模式内。

## 2. 查看器改造 — `image_viewer_page.dart`

### chrome 显隐
- 默认可见；**点击媒体区切换**（GestureDetector.onTap 包在 InteractiveViewer 外层，注意 InteractiveViewer 会消费手势——tap 不会被吞，双击需 `onDoubleTap`）+ 底部工具栏“全屏”按钮 + 键盘 `F`。
- 隐藏内容：AppBar + 底部动作栏 + （移动端）系统栏 `SystemUiMode.immersiveSticky`；恢复即全部回来。动画用 `MotionTokens.fast`（180ms）淡入淡出，符合 §5.2 “动画表达状态但不影响可达性”。
- 隐藏态下手势仍可用（缩放/翻页）——chrome 只是视觉层。

### 缩放
- 保留 InteractiveViewer + 每页 TransformationController + 0.9–6.0。
- 新增 **双击缩放循环**：fit(1.0) → 2.5 → fit；实现=记录当前 scale，双击 setMatrix 到下一档，`MotionTokens.fast` 动画。双击与单击 chrome 切换的冲突：单击延迟 ~250ms 等双击判定，或双击只缩放、单击只切 chrome（PixEz 用 PhotoView 自带 cycle，本项目需自实现）。
- 显式“适应屏幕”动作：工具栏 `Icons.fit_screen` → 复位当前页 controller 到 identity；键盘 `0`。

### 页码与跳页
- 底部栏左侧 `n / total` 文本改为**可点击** → `showAppBottomSheet` 页码网格/滑条（多页）或直接 `PageController.jumpToPage`；跳页后 `replaceImageViewerPage` 照旧走 L144 链路。

### 动作（同名同义对齐详情页）
- 保存当前页：`illustDownloadControllerProvider` 单页提交（复用 page_image 的 stateFor 判定 → 角标/禁用/乐观态同语义）；保存全部：`downloadAll`。
- 分享：`_share` 同 `ShareService` 调用。
- 信息：`Icons.info_outline` → pop 回详情并 ensureVisible 到 InfoBlock（或直接弹 bottom sheet 显示标题/作者/日期/ID——**推荐后者**，查看器不回详情即可读元信息，更贴“chrome 内可达”）。
- 全部 icon-only 动作必须带 tooltip（spec 硬性要求）。

### 空态
- `_pageCount == 0`：title 不再渲染 `1 / 0`——计数位整体隐藏（`SizedBox.shrink`）或显示通用标题；body 保留 `viewerNoImages`；动作栏禁用保存/分享/跳页（语义化 disabled 而非移除，读屏可感知“存在但不可用”）。

### 键盘/鼠标等价（桌面 + 物理键盘移动设备）
- `←`/`→` 或 `J`/`K` 翻页；`+`/`-`/`0` 缩放/复位；`F` 全屏；`Esc`/`Backspace` 返回；`S` 保存当前页、`I` 信息。
- 鼠标：滚轮缩放（Listener onPointerSignal → 以指针位置为焦点缩放）或滚轮翻页——**建议滚轮=缩放**（与 InteractiveViewer 语义一致），Shift+滚轮或侧键翻页；需要 `CallbackShortcuts` + `Focus(autofocus)`。
- 实现载体：整页包 `CallbackShortcuts`（而非 Shortcuts/Actions 全家桶——本项目无既有 shortcut 基建，CallbackShortcuts 最小）。

### 返回优先级（PopScope）
- `canPop` 判定链：zoomed → 复位当前页缩放并消费返回；chrome 隐藏 → 恢复 chrome 并消费；否则放行退路由。该优先级要在实现稿写明并加 widget test。

## 3. 下载模式显式化

- 进入仍走长按（保留现有习惯），但进入后挂**显式 chrome**：AppBar title 变“选择下载页（已选 n）”或底部条「已选 n / 共 N · 全选 · 完成 · 取消」。
- 完成=提交所选页 `downloadPages(indices)`（协调器已有 downloadPage/组提交基建，需要新增“指定页集合”入参或循环 downloadPage——后者已 dedupe 安全）；取消/点空白/返回=退出模式。
- 退出模式时若已有进行中任务不取消（任务归 DownloadManager 管，模式只是 UI 选择层）。
- 迁移：旧“长按即切模式 + 点空白退出”保留；新增 chrome 让模式可发现、可确认——**旧手势语义不变**，符合 §4.1 不破坏既有肌肉记忆。

## 4. 系列“继续阅读”语义 —— 决策

- **本 leaf 不新增持久化进度 schema**（§5.2：不得把更新游标伪装成阅读进度；真进度需要新 schema+迁移+回滚文档，超出 W4 最小闭环，列为后续任务——见 risks.md R3）。
- 落地语义限定为：
  - 系列页/详情系列卡：「开始阅读」（= 第 1 话，feed 首个 ID）、「上一话 / 下一话」（沿用 `IllustSeriesSection` 的 prev/next）、「查看全部」（series feed 页）。
  - 「继续阅读」按钮仅当能给出真实目标时才出现：可基于**会话内**最近打开的该系列作品 ID（内存态，不落盘）实现“回到上次那一话”，文案用「返回第 n 话」而非「继续阅读」，避免暗示持久进度。
  - `WatchlistReadCursor.markSeen` 保持现状（打开系列页推进 seen 游标）——它只驱动“有新内容”角标，文案上不向用户呈现为“阅读进度”。

## 5. 触觉薄包装 —— 位置与 API（§5.6）

### 位置
- `lib/app/haptics/app_haptics.dart`（新建目录，与 `app/motion/`、`app/widgets/` 平级——触觉与 motion 同属“交互反馈层”，放 app 层供所有 feature 消费；不放 `core/`：它是平台能力封装不是领域逻辑；不放 `widgets/`：它不是 widget）。

### API 形态
```dart
abstract final class AppHaptics {
  /// 轻确认：选择、toggle、入队成功、复制成功
  static void selectionClick() { ... }

  /// 强确认：进入管理/选择模式、危险确认、保存/发送成功、失败
  static void heavyImpact() { ... }
}
```
- 内部三件套（借鉴 PixEz，external-pixez-haptics.md §3）：
  1. enabled 判定：读 settings 开关——建议 `AppHaptics.configure({required bool Function() isEnabled})` 在 bootstrap 注入，或静态持有 `bool Function()? _isEnabled`；不直接 import Riverpod provider（保持静态可调，避免 widget 层到处 `ref.read`）。
  2. 节流：`selectionClick` 50ms / `heavyImpact` 120ms 最小间隔（对齐 PixEz）。
  3. `try/catch` 吞 `MissingPluginException`/平台异常。
- 不做：success/warning/error/vibrate/自定义时长——YAGNI，W6/W7 需要时再扩；公开注释写明“唯一入口，禁止直连 `HapticFeedback`”，并在 `.trellis/spec/frontend` 增补一条约束（spec 修改属实现任务范围）。

### 首个消费者接线点（W4 内）
| 触点 | 位置 | 级 |
|---|---|---|
| 进入/退出下载模式（长按） | `_toggleDownloadMode`（detail L90） | selectionClick |
| 单页下载入队成功 | `_DownloadBadge.onTap`（page_image L232-） | selectionClick |
| 下载全部提交成功 | detail AppBar L167-185 的 await 后 | heavyImpact |
| 下载提交失败（各处 catch） | snackbar 同一处 | heavyImpact |
| Ugoira 导出完成/失败 | ugoira_viewer export 监听处 | 成功 selectionClick / 失败 heavyImpact |
| 查看器保存成功 | 新工具栏保存动作 | heavyImpact |
- 原则：触觉调用与既有 snackbar/状态变更**同点同分支**——不把触觉塞进单独监听，保证“视觉先自足、触觉冗余”。

## 6. tag 菜单与长按迁移

- 新行为：`TagChip.onLongPress` → `showAppBottomSheet` 菜单：「搜索该 tag / 复制 / 屏蔽该 tag / （已进入屏蔽模式时）取消屏蔽」；屏蔽模式仍保留（现在是唯一能批量屏蔽的路径），菜单内加「批量屏蔽模式」入口项。
- 迁移表：
  | 旧 | 新 |
  |---|---|
  | tag 长按 = 切屏蔽模式 | tag 长按 = 动作菜单（含屏蔽+复制+进入批量模式） |
  | 图片长按 = 切下载模式 | 保留；模式 chrome 显式化（§3） |
  | 屏蔽模式内 tag tap = 切屏蔽态 | 不变 |
  | PixEz 式 tag 收藏 | **不做**——本项目无 tag 收藏 store，菜单不出现该选项 |
- 兼容说明：旧长按=模式切换是隐式且无确认；新长按=菜单是显式有撤销——符合 §4.4 “二级确认+可撤销”。

## 7. Ugoira 入口一致化

- 导出按钮保留 overlay 原位；状态文案对齐下载任务语义：idle=“导出 GIF”、running/finalizing=进度%、“已加入导出/导出失败·重试”。
- 下载模式下 ugoira 页：长按进下载模式后，ugoira 角标语义=“导出 GIF”（而非页下载），保证“同位同义”。
- 不在 ImageViewerPage 给 ugoira 做查看器变体（缩放对逐帧动画无意义）；在设计/实现稿中显式注明该排除。

## 8. 实施顺序建议（与 implement.md stage 对齐用）

1. `AppHaptics` 包装 + 详情/卡片下载触点接入（首个消费者闭环，最小可验证）；
2. 下载模式 chrome 显式化（标题/计数/完成/取消）；
3. 查看器 chrome/工具栏/双击/跳页/空态/键鼠/PopScope；
4. 窄屏 compact header + 信息跳转锚点；
5. tag 长按菜单；
6. Ugoira 状态文案对齐；
7. 系列入口文案（开始阅读/上一话·下一话/返回第 n 话内存态）。
