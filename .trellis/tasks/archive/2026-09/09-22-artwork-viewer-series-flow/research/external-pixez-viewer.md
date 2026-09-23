# 外部调研：PixEz 查看器与详情信息可达（Notsfsssf/pixez-flutter @ master）

来源文件（临存 /tmp，行号对该快照核对）：`pixez_photo_zoom_page.dart`、`pixez_illust_lighting_page.dart`、`pixez_illust_detail_content.dart`、`pixez_save_store.dart`、`pixez_zoom_page.dart`。

## 1. PhotoZoomPage — `lib/page/picture/photo_zoom_page.dart`（344 行）

### 缩放与翻页
- `PhotoView` / `PhotoViewGallery.builder`（L90、L116）；`initialScale: PhotoViewComputedScale.contained`（L92、L125）——初始 contain，双击走 PhotoView 默认 scaleStateCycle。
- **显式扩展拖拽设备**（L105-115）：`ScrollConfiguration.copyWith(dragDevices: {touch, stylus, invertedStylus, trackpad, mouse})`——桌面鼠标/触控板可直接拖图，这是本项目 InteractiveViewer 默认已支持但值得对照的点（InteractiveViewer 默认 panAxis 接受所有指针；差异在 PixEz 是滚动语义，本项目是变换语义）。

### chrome
- **常驻 `BottomAppBar`**（L188-304），透明背景，左侧 `photo_library` 图标 + `"${_index + 1}/${pageCount}"` 文本（L198-208），右侧：
  - back `arrow_back` → `Navigator.pop`（L214-217）——**返回按钮在底部栏而非 AppBar**；
  - `fullscreen` → `SystemChrome.setEnabledSystemUIMode(manual, overlays: [])`（L220-229）进沉浸；
  - `copy`（ClipboardPlugin 支持时，L233-）；
  - `save_alt` 包 `GestureDetector`：tap 存当前页/全页，`onLongPress → HapticUtil.heavy()` + `saveStore.saveImage`（L239-261）——**长按保存带 heavy 触觉**；
  - `share`：`shareShow` 状态控制 opacity 1/0.5（L265），`SharePlus.instance.share(... sharePositionOrigin)`（L289-292）；
  - 画质切换 IconButton（L304-）。
- **fullscreen 态的 chrome**（L164-186）：`BottomAppBar` 只剩退出全屏按钮 + `SystemUiMode.manual, overlays: SystemUiOverlay.values` 恢复（L169-176）；顶部 _fullScreen 时 also 恢复 overlays（L58-61）。
- 结论：PixEz 的 chrome = **常驻底栏 + 显式全屏开关**，不是“点图隐藏全部 chrome”流派；页码以图标+文本呈现但**不可点击跳页**——这一点本项目可做得比它好（§4.4 要求页码选择）。

### 保存反馈
- `save_store.dart`：`saveStream` → SUCCESS `Toaster.downloadOk("title (pN) saved")`（L128-131）；JOIN/INQUEUE → 带 `JobPage` 入口的 toast（L133-220）；ALREADY → 带重试图标的 toast（L221-）。
- `saveImage`（L435-469）：iOS 先 DocumentPlugin 权限；单页/多页分发 `_saveInternal`；文件名经 `_handleFileName`（支持 JS 自定义命名）。

## 2. 详情页“信息可达”方案 — `illust_lighting_page.dart`

- `ScrollController _scrollController`（L137/149，attach 于 L462）。
- `expand_less` IconButton（L224-228）→ `Scrollable.ensureVisible(context of 信息区)`——**详情图片区内嵌一个向下箭头按钮，一键滚到信息区**，与悬浮/常驻入口相比侵入性低。
- 对照本项目窄屏“滑过全部图片才到 InfoBlock”：PixEz 方案是**图片内嵌跳转锚点**；备选是常驻 compact 标题区（实现稿做对比决策）。

## 3. 详情页 tag 长按 — `illust_detail_content.dart`

- `_longPressTag`（L405-470）：`SimpleDialog` 标题=tag 名+译名，三选项：
  - `ban`（0）→ `muteStore.insertBanTag(BanTagPersist)`；
  - `bookmark`（1）→ `bookTagStore.bookTag(name)`——**tag 收藏**（本项目无对应物）；
  - `copy`（2）→ `HapticUtil.light()` + `Clipboard.setData` + SnackBar。
- 接线（L474-478）：`GestureDetector.onLongPress → HapticUtil.heavy() + _longPressTag`；`onTap` → 进 tag 页。
- 与本项目差异：本项目 tag 长按=切换屏蔽模式（就地多选），PixEz 长按=直接弹三选菜单。**本项目已有的“屏蔽模式”多选能力比 PixEz 强**（可批量屏蔽），缺的是单 tag 快捷菜单（复制/收藏）。

## 4. `zoom_page.dart` 备注

- 是图片内 pinch-zoom overlay（Listener 计 pointer 数，≥2 指时把图抬到 OverlayEntry 缩放，松手 reverse 回原位），用于**详情内直接捏合预览**，不进全屏查看器——本项目无对应物；W4 若不做该能力，需在设计中显式说不做（属于可选增强）。

## 5. Material 3 viewer-chrome 惯例（规范参照）

- M3 对全屏媒体的惯例：chrome（app bar / 系统栏）默认可见，单击媒体区切换显隐；显隐用淡入淡出（~150-250ms，对应本项目 `MotionTokens.fast/medium`）；系统 UI 用 `SystemUiMode.immersiveSticky` 或 manual overlays；返回手势优先退出沉浸态。
- PixEz 选择了“显式全屏按钮”而非“点击媒体切换”——M3 两者皆可，但 §4.4 写明了“chrome show/hide”，点击切换是更低门槛发现路径；实现稿建议**两者并存**：点击媒体区切换 + 工具栏全屏按钮（键盘 `F` 等价）。
