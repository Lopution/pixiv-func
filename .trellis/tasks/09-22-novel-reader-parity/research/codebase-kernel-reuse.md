# 代码库调研：NovelReader 内核复用点与舞台抽取缝

基线同 diff-matrix（HEAD == origin/main 产品代码）。`§` 引用父任务 design.md。

## 1. 内核已数据源无关

`NovelReader`（`lib/features/novel/novel_reader.dart:215-250`）的输入只有：

- `novel: NovelEntity` —— 本地端已用合成实体适配（`local_novel_reader_page.dart:97-114`：`id:-localId`、`contentVersion:'local:<id>:<len>'`、`paragraphs=text.split('\n')` 得 `p0..pN`），证明实体即数据缝。
- `settings`、`initialAnchor`、`textColor`、`onAnchorChanged`、`onCenterTap`、`onProgressChanged`、`handle`。

本地端已有字段全部满足内核契约，parity 差在「宿主舞台」不在内核。**结论：不新建平行阅读器，抽 `_NovelReaderStage` 为共享舞台即可**（两调用点同在 `features/novel/`，不触碰跨 feature 组件规则；目录规范要求 ≥3 消费者才上 `lib/app/widgets/`，本例 2 个 → 留在 feature 内私有）。

## 2. 内核能力清单（可复用点）

| 能力 | 位置 | 状态 |
|---|---|---|
| 横向 PageView + 30/40/30 点按区 | `NovelReaderController.zoneForTap` L153-159；GestureDetector L345-368 | 两端共享（本地中区空回调） |
| 页状态单一写入 | `onPageChanged` L373-377 | 直接复用 |
| 命令面 | `NovelReaderHandle` L204-212：`currentPage`/`pageCount`/`goToPage`（L277-284，`MotionTokens.fast` 动画） | goToPage 无调用方 → 进度跳转的直接接线点 |
| 代次闸 | `NovelReaderCommitGate` L45-134：beginLayout 取消旧 context；commit 按 generation/contentVersion/chapter/disposed 拒绝过期结果 | spec「Novel Typed Markup and Reader Commit Contract」固定行为，勿绕过 |
| 排版引擎 | `NovelLayoutEngine`：`layoutDocumentCancellable`（L408-432，chunk 让出+进度回调）、LRU 8 缓存（L272-298）、预算（L40-59） | 已支持 markup 文档与裸段落两种输入 |
| 锚点/字符换算 | `NovelAnchor`（L104-122）；`NovelLayout.pageIndexForAnchor` L231-243；`pageIndexForCharacter` L245-252（**当前零调用方**，恰为本地 read_offset 恢复/进度跳转预留）；`progressPercent` L254-258 | 复用，勿自写换算 |
| 恢复语义 | 首次布局吃 `initialAnchor`，后续重排保留当前页 startAnchor；布局期间用户翻页优先（L432-471） | 本地接上即得同语义 |
| 设置 → 排版 | `NovelLayoutStyle`（L9-35）由 `NovelReaderSettings` 映射（`_style` L389-392）；settings 变化触发 force relayout（L293-301） | 本地端传 settings 即生效 |

## 3. 舞台需要抽出的状态机（现状全在 `_NovelReaderStage`）

`novel_page.dart` 内私有实现，W5 需参数化的缝：

1. **prefs 加载闸**：`_loadPrefs` L173-191 → `_prefsReady` 才挂 `NovelReader`（L283-287，避免默认排版后再重排）。本地端需要同一闸：settings + `readOffset`→anchor 转换。
2. **chrome 可见性**：`_chromeVisible`/`_toggleChrome`/`_hideChrome` L157/193-197；`_ChromeBar` L653-729（滑入滑出、隐藏即无命中/无语义）。
3. **PopScope**：L229-233（语义修正归 W1，W5 消费修后契约）。
4. **页脚 tip**：L243-261。
5. **设置弹层**：`_showReaderSettings` L456-518 + `_SettingsSliderRow` L601-646 —— 移入共享舞台，本地同用。
6. **进度簇**：底栏 `页/总页 · %` L424-434 + 字号 ± L417-441 + 设置入口 L442-446 —— 共享；进度文本升级为可操作控件（见 implementation-draft）。
7. **顶栏**：back/title 共享；右侧动作槽按数据源分化（在线：share/bookmark/info；本地：文件信息）。
8. **底部附加行**：系列/前后篇 bar 仅在线（L408-414）→ 以可选 slot 注入。
9. **信息弹层**：在线作品信息 sheet L536-596 ↔ 本地文件信息 sheet —— 同一入口图标位，不同 builder。
10. **历史包裹**：`HistoryVisibility` L311-329 仅在线 → stage 参数（可选包装器）。
11. **持久化缝**：在线 `NovelProgressStore`（账号×novelId，anchor {p,o}）vs 本地 `read_offset`（字符）→ 抽 `load()/save(anchor)` 两方法的最小绑定，本地适配器做 anchor↔charOffset 双向换算（`_persistCursor` L86-95 已有正向算法；反向 = 累加 `len+1` 定位段落，越界/失效应返回 null 或 clamp——W1 恢复条目同一处代码）。

## 4. 须保持不动的内核行为

- `_relayout` 只吞 `ApiCancelled`（L485-487）；预算超限冒泡为未捕获异步错误 —— 两端同缺口，若要兜底属新行为。
- `didChangeAppLifecycleState(resumed)` 触发 force relayout（L310-314）。
- `didUpdateWidget`：`contentVersion` 变 → 清缓存重排；`settings` 变 → force 重排（L293-301）。
- 页面渲染为定高 `Text` 行（L516-541）：`softWrap:false`、`overflow:clip` —— 行长限宽必须发生在排版 viewport 层，不能只缩 widget。

## 5. 周边契约（消费而非重建）

- `showAppBottomSheet`/`showAppDialog`（`app/motion/app_overlays.dart:9-53`）已带 reduced-motion 闸；M3 modal sheet 默认 `maxWidth:640`（Flutter bottom_sheet.dart:1511）—— 宽屏无需自写限宽。
- `AppBreakpoints`（`app/layout/app_breakpoints.dart:5-28`）：compact=medium=600、expanded=1200、`useTwoPaneDetail`≥1200 —— 唯一断点来源。
- `TwoPane`（`two_pane.dart`，55:45）为详情双栏，阅读器行长限宽不走双栏。
- `MotionTokens.fast/sheet/dialog`；`FeedLoading/FeedError/FeedEmpty`（`app/widgets/feed/`）；`CaptionRichText`/`AuthorSummary`/`TagChip`/`BookmarkSwitchButton(isNovel)`/`WatchlistToggle` 均可直接复用。
- l10n 现有 `novel*` 键（app_en.arb:833-856）覆盖 loading/错误/系列/字号/行距/主题/进度/设置；**目录、跳转、文件信息、上一章/下一章等新键需新增**（l10n 属高冲突文件，§8）。
