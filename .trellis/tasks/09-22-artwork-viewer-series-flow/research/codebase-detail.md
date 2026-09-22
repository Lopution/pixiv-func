# 代码调研：详情页 sliver 结构 / 详情 pager / InfoBlock / 标签与长按

对应设计 §4.4 W4 条目：移动端长图详情信息可达；桌面端双栏等价；标签直接菜单；批量下载显式模式。行号已对当前 HEAD 逐一核对。

## 1. IllustDetailPage — `lib/features/illust/detail/illust_detail_page.dart`

### 页面级状态
- `bool _downloadMode = false`（L58）、`bool _blockMode = false`（L59）：两个本地“模式位”，分别驱动下载批量模式与标签屏蔽模式。
- `_toggleDownloadMode()`（L90）是唯一切换入口，被 4 处复用：Ugoira 封面长按 L291、首图长按 L312、其余页长按 L336、双栏 DetailImagePager 长按 L409。

### AppBar（L154-200）
- 标题固定为通用文案 `illustDetailTitle`（L159-160）；注释 L155-158 明确说明作品标题放正文 InfoBlock（Shaft hero_title / 官方客户端布局），AppBar 单行槽位放不下长标题。
- actions 顺序：分享 `Icons.share_outlined`（L162-166，tooltip=`cardActionShare`）→ 仅 `_downloadMode` 时出现的下载全部 `Icons.file_download_outlined`（L167-185，tooltip=`downloadAll`）→ `BookmarkSwitchButton(isButton: false)`（L188-198，注释 L187-188：tap 切换、未收藏时长按弹 sheet）。
- `_share`（L202-219）走 `shareServiceProvider`，`ShareOutcome.copiedToClipboard` 时 `showAppSnackBar(linkCopied)`。

### imageSlivers（L271-347）与 metaSlivers（L348-362）
- imageSlivers：
  - L273-291：`entity.isUgoira` 分支 → `UgoiraViewer`（注释 L276-：与 DetailPageImage 同一契约），`downloadMode: _downloadMode`（L290）、`onLongPress: _toggleDownloadMode`（L291）。
  - L302-312：首图 `DetailPageImage`（`downloadMode` L311、`onLongPress` L312）。
  - L316-341：`SliverList` 渲染第 2..N 页 `DetailPageImage`（`onLongPress` L336）。**窄屏多页作品是纵向排布的图片列表，不是 pager。**
- metaSlivers 顺序（窄屏接在 imageSlivers 之后）：
  1. `IllustSeriesSection(illustId)`（L352）——系列卡片在最前；
  2. `InfoBlock`（L354-358），`blockMode: _blockMode`、`onToggleBlockMode: () => setState(_blockMode = !_blockMode)`（L356-357）；
  3. `RelatedIllustsSlivers`（L362）。

### 布局分派（L398-425）
- 外层 `GestureDetector`（L398-402）：`onTap` 仅在 `_downloadMode` 为真时退出下载模式——给“点击空白退出模式”兜底；图片自身的 GestureDetector 在手势竞技场中优先，所以图片 tap 仍进查看器。
- `AppBreakpoints.useTwoPaneDetail(width)`（L403，阈值 ≥1200，见 `lib/app/layout/app_breakpoints.dart`）：
  - true → `TwoPane(primary: DetailImagePager(... onLongPress: _toggleDownloadMode L409), secondary: meta CustomScrollView L388-394)`；
  - false → 单 `CustomScrollView(slivers: [...imageSlivers, ...metaSlivers])`（L422-425），包在 `SmoothWheelScroll` 里（桌面滚轮平滑，非桌面透传并回落到 `PrimaryScrollController`——见 smooth_wheel_scroll.dart L150-157）。
- `_entityOf`（L221-229）：Ready/Restricted/Error 三态都尽量回落到 store/入参快照实体，保证 sliver 结构在降级态也可用。

### 信息可达性现状（设计 §4.4 待办）
- 窄屏：标题/作者/页数/标签/评论入口全部在 InfoBlock，长多页作品需滑过全部图片才到——**当前没有任何跳转/常驻信息入口**；`SmoothWheelScroll` 的 controller 在 builder 内生成（L390/422），没有外露给“跳到信息区”的锚点。
- 双栏：信息列常驻右侧，天然满足“等价命名/可达”。

## 2. IllustDetailPagerPage — `lib/features/illust/detail/illust_detail_pager_page.dart`

- 契约注释 L15-16：通过 `IllustPagerSource` 与 feed 共享列表，滑近已加载边缘时调用 feed `loadMore`。
- `static const loadAhead = 4`（L42）；`PageController(initialPage: _index)`（L68）。
- `_onPageChanged`（L79-86）：更新 `_currentId = ids[index]`；`index >= ids.length - loadAhead`（L86）时触发 `source.loadMore()`。
- 列表变更重定位：L122-125 按 `_currentId` 在 ids 中重新 `indexOf`，找不到则 `clamp` 回原 index——**id 锚定而非 index 锚定**，插入/删除后工作不跳。
- `PageView.builder`（L137）+ `itemBuilder` 内 `HeroMode`（L148-149）：只有当前页持有活跃 Hero，避免多页 hero tag 冲突。
- 入口：`openIllust` 携带 `IllustRouteExtra.pagerSource`（routes.dart L91, L378-384）；feed 卡片在 `illust_card.dart` L144-153 通过 `IllustPagerScope.maybeOf(context)` 提供 source。

## 3. DetailImagePager — `lib/features/illust/detail/widgets/detail_image_pager.dart`（仅双栏）

- `count = entity.isUgoira ? 1 : entity.pageCount`（L75）：Ugoira 恒为单页。
- 键盘：`LogicalKeyboardKey.arrowLeft/arrowRight` → `_page ± 1`（L59-60），外层 `Focus`（L76）承接焦点。
- 页面内容：Ugoira → `UgoiraViewer`（L88-89）；普通页 → `DetailPageImage`（L110）；`AspectRatio(aspectRatio: entity.pageAspectRatioAt(index))`（L130-131）做 contain。
- 页码指示 `n / total`：L157 的 `Text`。
- 注意：双栏图片页是**横向 PageView**；窄屏同内容用纵向 sliver 列表——同一对象两种呈现变体，交互语义需在实现稿中明确等价关系。

## 4. InfoBlock — `lib/features/illust/detail/widgets/info_block.dart`（PR #50 布局基线）

自上而下（保留此顺序，不回旧版式）：
1. `SelectableText` 标题（L49）；
2. `AuthorSummary`（L54-61），整块 `onTap → openUser`，`avatarKey: 'illust-author-avatar'`（L60）；
3. caption 级日期/统计行（L66-98，注释 L66-68 指明向 PixEz/Shaft 收敛到次色小字）；
4. 尺寸 + `SelectableText('ID: ...')`（L102-106；注释说明用 SelectableText 而非 SelectionArea 的原因——SelectionArea 会引入全局 Selection 行为，AOT 下别处没有用过）；
5. `CaptionRichText`（L109-115，caption 非空才渲染）；
6. `TagChips`（L119-145）：`TagChip(blockMode: blockMode)`（L125）；tap——blockMode 下切换该 tag 的屏蔽状态，否则 `openTagSearch`（L128-）；`onLongPress: onToggleBlockMode`（L142）；
7. 评论按钮（L148）→ `openIllustComments`。
- meta 数字样式 `_metaNumeric`（L158-162）：tabular figures + caption 字号 + 次色。

### TagChip — `lib/app/widgets/tag_chips.dart`
- `InkWell`（L42-44）承载 tap/longPress，天然有焦点/键盘/水波纹；`blockMode` 时 `Semantics(selected: blocked)`（L62-64）。
- 当前长按语义：**进入/退出 tag 屏蔽模式**（toggle），不是 PixEz 那种直接弹出“ban/bookmark/copy”菜单。

## 5. 卡片长按动作 — `lib/app/widgets/card_actions/`

- `illust_card.dart` L173/L186：`onLongPress → showCardActionSheet(context, entity)`。
- `illust_card_actions.dart` 动作集：bookmark（L39-63，收藏态驱动图标/文案）、download（L71-96，`downloadAll(entity)` → `downloadQueuedMessage`/`downloadSubmissionFailed` snackbar）、watchLater（L112-141）、mute-work（L156-180，本地屏蔽，无官方端点）、mute-user（L190-197）。
- `card_action_sheet.dart` L28 为 `ListTile` 条目，走共享 `showAppBottomSheet` 通道。

## 6. 下载模式（downloadMode）语义小结

- 进入：任一详情图片/ugoira/双栏 pager 长按；退出：AppBar 下载全部提交后仍保持（需再点空白或再长按）、页面外层 tap 空白处。
- 进入后：每页图片叠加遮罩+角标（page_image.dart L191-211 的 `filterColor/BlendMode` + L221-225 `_DownloadBadge`）；AppBar 出现“下载全部”。
- **这是当前唯一的“显式批量模式”雏形**，与设计 §4.4 要求的“带标题/计数/完成/取消的显式模式”最接近的落点；但缺模式标题、已选计数、完成/取消按钮，且退出手势不可发现。

## 关键缺口（供 implementation-draft 引用）

1. 窄屏无“跳到信息区”锚点；InfoBlock 深度 = 全部图片高度之后。
2. 长按同时承担“进入下载模式”（图片）和“切换 tag 屏蔽模式”（tag），与 §4.4 “长按收敛为选择/管理入口 + 二级确认”需要对齐。
3. 下载模式缺标题/计数/完成/取消的显式 chrome。
4. tag 的搜索/复制/屏蔽目前分散在 tap/长按两态，无统一菜单；`TagChip` 无复制入口。
