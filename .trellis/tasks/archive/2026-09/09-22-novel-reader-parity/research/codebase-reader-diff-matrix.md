# 代码库调研：在线 vs 本地小说阅读器差异矩阵

基线：`docs/09-22-ui-leaf-planning @8067b2d`；`git diff origin/main...HEAD -- lib/` 为空，产品代码与 `origin/main@3c6c1bd` 一致。行号已对 HEAD 逐一核对。

## 1. 入口与路由

| 项 | 在线 | 本地 |
|---|---|---|
| 页面类 | `NovelPage`（`lib/features/novel/novel_page.dart:64`） | `LocalNovelReaderPage`（`lib/features/novel/local_novel_reader_page.dart:29`） |
| 路由 | `novel/:novelId`（`routes.dart:511`，子路由 `comments`、`comments/:rootCommentId` L519-546） | `local-novels/:localId`（`routes.dart:574-582`），库页 `local-novels` L570 |
| 打开方式 | `openNovel` L1185；`openNovelComments` L1368 | `openLocalNovelReader` L1205；库入口 `openLocalNovels` L1201，从设置页进入（`settings_page.dart:162`） |
| 数据源 | `_novelDetailProvider` L45-53 → `fetchDetail` 合并 `/v2/novel/detail` + `/webview/v2/novel`（`novel_repository.dart:126-174`） | `_localNovelContentProvider` L16-24 → `repository.get(id)` + `File(path).readAsString()` 整读 |

## 2. 舞台（stage）差异

| 维度 | 在线 `_NovelReaderStage`（novel_page.dart:147-849） | 本地 `_LocalNovelReaderBody`（local_novel_reader_page.dart:54-114） |
|---|---|---|
| 骨架 | 全屏 `Stack`：正文 + 页脚 tip + 顶/底 `_ChromeBar` 滑入滑出（L229-275） | 普通 `Scaffold` + 常驻 `AppBar` + `ListTile` 信息行 + `NovelReader`（L37-81） |
| AppBar 标题 | chrome 内显示作品标题（L355-364） | AppBar 固定显示页面名 `localNovelsTitle`（L38），作品标题在信息行 |
| chrome 显隐 | 中区点按 `_toggleChrome`（L193，经 `onCenterTap` L296）；默认隐藏 | 无 chrome 概念；中心点按 `onCenterTap==null` 无任何效果 |
| 返回 | `PopScope(canPop: !_chromeVisible)` L229-233：系统返回先关 chrome；**显式返回按钮 L352 `maybePop()` 同样被 canPop 拦截 → 先关 chrome 不离页（W1 已确认缺陷）** | 无 PopScope，系统/显式返回都直接 pop |
| 页脚 tip | 常驻 `标题 · 页/总页 · %`（L243-261，垫 `viewPadding.bottom`） | 无 |
| 状态页 | `_NovelStatusScaffold` 覆盖 loading/error/restricted/contentUnavailable（L74-109, L119-141） | `async.when` loading/error/data（L39-49），无 restricted 概念 |
| 历史记录 | `HistoryVisibility` 包裹（L311-329），snapshot 携带 anchor（`history_snapshot.dart:14`） | 无（本地书不入浏览历史） |
| SafeArea | 正文 `SafeArea(bottom:false)` L288-289；chrome `Material` 铺到屏边、`SafeArea` 只垫控件（L346-348, L403-405，spec 明确要求） | AppBar/正文走 Scaffold 默认 |

## 3. 阅读设置

| 维度 | 在线 | 本地 |
|---|---|---|
| 设置模型 | `NovelReaderSettings`（fontSize 17、lineHeight 1.7、theme；clamp 12–30 / 1.3–2.4，`reader_settings.dart:53-67`） | 不传 settings → `const NovelReaderSettings()` 默认（reader L219 默认值） |
| 持久化 | `NovelReaderSettingsStore`（SharedPreferencesAsync，键 `pixivfunc.novel.reader_settings.v1`，设备级不随账号，L110-134） | 无 |
| 设置弹层 | `_showReaderSettings` L456-518：`showAppBottomSheet` + `StatefulBuilder`，字号 slider 12–30、行距 slider **1.1–2.2**（与 clamp 1.3–2.4 不一致）、4 个主题 `ChoiceChip`；改动即 `_applySettings` 写穿（L199-202，immediate 语义） | 无入口 |
| 主题调色 | `novelReaderPalette` 4 预设（system/paper/sepia/night，L30-48），染色舞台底+顶底栏 | 无 |

## 4. 进度与恢复

| 维度 | 在线 | 本地 |
|---|---|---|
| 进度显示 | 底栏只读文本 `page/pageCount · percent`（L424-434）+ 页脚 tip | 无 |
| 进度跳转 | **无可操作控件**；`NovelReaderHandle.goToPage` 已存在（reader L211、L277-284 animateToPage）但无调用方 | 无 |
| 持久化写入 | `_persistAnchor` L204-219 → `NovelProgressStore.write`（键 `<accountId>:<novelId>`，payload `{p,o}`，LRU 200，reader_settings.dart:139-195）；**`onAnchorChanged` 在首次布局落定即触发 → 打开即写一条进度** | `_persistCursor` L86-95：anchor→字符偏移（前段 `len+1` 累加 + offset）→ `updateReadOffset`（repository L126-134，SQLite `read_offset` 列，`local_novel_database.dart:86`）；同样首次布局即写 |
| 恢复 | `_loadPrefs` L173-191 读 `{p,o}` → `NovelAnchor` → `initialAnchor`（reader 首次布局用 `pageIndexForAnchor` 恢复，L432-437/471） | **`readOffset` 读了不用**：`_entityFor`/`NovelReader` 均未消费（W1 条目：转换为有效 initialAnchor+边界/失效处理）。内核已有反向工具 `NovelLayout.pageIndexForCharacter`（layout L245-252，当前无调用方） |
| 进度粒度 | 段落 anchor（段落 id + UTF-16 偏移），布局无关 | 字符偏移，布局无关 |

## 5. 增量能力

| 能力 | 在线 | 本地 |
|---|---|---|
| 收藏 | `BookmarkSwitchButton(isNovel:true)` 顶栏（L370-374） | — |
| 分享 | 顶栏 share → `SharePayload.novel`（L365-369, L520-534） | — |
| 作者 | 信息弹层 `AuthorSummary` → `openUser`（L551-560） | — |
| 评论 | 信息弹层 TextButton → `openNovelComments`（L578-591） | — |
| 作品信息 | `_showNovelInfo` L536-596：`showAppBottomSheet`+`DraggableScrollableSheet`；标题/作者/caption（CaptionRichText）/TagChip（**无 onTap，W1 已确认：可操作或非交互化** L570-573）/评论入口 | 仅常驻 `ListTile`（标题+字数 L65-73） |
| 系列 | `_NovelSeriesBar` L770-848（fetchSeries、上/下篇、`WatchlistToggle`、`markSeen` 游标 L804-813）或 `_NovelAdjacentBar` L733-768（webview seriesNavigation prev/next）；均推新 `NovelPage` 路由 | — |
| 文件信息 | — | 只有 ListTile 一行；`LocalNovel` 另有 path/encoding/importedAt/author（author 导入时恒 null，repository L95） |
| 目录/章节 | markup `[[chapter:]]` → `NovelChapterBlock` → `isChapterHeading`/`pageBreakBefore`（entity L391-402、layout `_paragraphsForDocument` L434-484）；`NovelLayoutPage.chapterTitle` 已产出，**无 TOC UI** | 无章节概念（TXT 无标记） |

## 6. 两端共有的缺口

- 可操作进度（slider/跳转）：两端皆无，`handle.goToPage` 是现成出口。
- 键盘/鼠标等价路径：无 `Focus/onKeyEvent`；`detail_image_pager.dart:44-79` 有 ←/→ 翻页先例。
- 宽屏行长：`_maxWidth = viewport - 48`（layout L878-882），无上限 → 宽屏单行可超百字。
- 嵌入式插图：`pixivimage`/`uploadedimage` token 合法时 `displayText=''`（entity L199/227），`embeddedImages`/`embeddedIllustThumbs` 已解析但 `_NovelPage` 只渲染文本行（reader L524-539）——两端同为纯文本，渲染插图不在 parity 范围。
- 超大文本：`NovelLayoutBudget.maxTextUnits=4M`（layout L40-59）超限抛 `NovelLayoutBudgetExceeded`；`_relayout` 只捕 `ApiCancelled`（L485-487）→ 未捕获异常路径两端相同。
- 音量键/翻页模式（覆盖/滚动）：均无，Shaft/legado 有；本包范围外注意别被扩张进来。
