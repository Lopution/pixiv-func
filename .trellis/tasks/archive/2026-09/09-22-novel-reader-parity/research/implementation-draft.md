# W5 实现草案：在线/本地小说同一阅读舞台

对照父任务 `design.md` §4.5 逐条展开。基线 HEAD@8067b2d（产品代码 == origin/main@3c6c1bd）。**前置：W1 合入后再动手**——PopScope 语义与本地 `read_offset→initialAnchor` 转换属 W1 owning（§4.1、§4.11），W5 在其契约上重构，不在本包内重做。

## A. 同一舞台（§4.5 条 1）

**做法**：把 `_NovelReaderStage`（novel_page.dart:147-849）抽成共享 widget（如 `NovelReaderStage`，留在 `lib/features/novel/`——两消费者同目录，不满足 ≥3 消费者的 `lib/app/widgets/` 准入，见 directory-structure.md）。页面壳只负责数据获取：

```
NovelPage            → fetchDetail → NovelReaderStage(spec: online)
LocalNovelReaderPage → get+readAsString → _entityFor → NovelReaderStage(spec: local)
```

舞台持有的统一状态机（从现状搬出）：

- `_prefsReady` 闸：settings + 恢复位置就绪后才挂 `NovelReader`（novel_page.dart:283-287）。
- chrome：`visible` bool + `_ChromeBar` 滑入滑出（653-729）；中区点按切换；隐藏即零命中零语义。
- `PopScope`：chrome 开时系统返回先关 chrome（W1 修正后的契约：显式返回离页）。
- 页脚 tip：常驻 `标题 · 页/总页 · %`（243-261）。
- 顶栏：返回 + 标题 + 动作槽；底栏：目录（可为空态）+ 可操作进度 + 设置。

**数据源缝（建议签名，最终以 implement.md 为准）**：

```dart
class NovelReaderStageSpec {
  final NovelEntity novel;
  final List<Widget> topActions;          // 在线: share/bookmark/info；本地: fileInfo
  final Widget? seriesBar;                // 仅在线：_NovelSeriesBar/_NovelAdjacentBar
  final ReaderProgressBinding progress;   // load()/save(anchor) —— 两种持久化适配
  final Widget Function(BuildContext)? extrasWrapper; // 在线: HistoryVisibility
}
abstract interface class ReaderProgressBinding {
  Future<NovelAnchor?> load();
  Future<void> save(NovelAnchor anchor);
}
```

- 在线适配器：`NovelProgressStore.read/write`（{p,o} 直存 anchor，reader_settings.dart:167-194）。
- 本地适配器：anchor→charOffset 用 `_persistCursor` 现算法（local_novel_reader_page.dart:86-95）；charOffset→anchor 反向走段落累加 `len+1` 定位，**越界/负值/超总长返回 null 或 clamp**（这正是 W1「边界/失效处理」条目的同一代码面——建议 W5 把它做成 binding 的 load 实现，W1 若先落了简化版则以 W1 为准再扩）。
- 本地 `extrasWrapper` = null（不入浏览历史：history 是账号域概念，本地书无账号键；保持现状）。

## B. 宽屏行长限宽（§4.5 条 3 后半 + §5.5）

**建议常量**：字号相对上限 `maxTextWidth = fontSize × 40`（WCAG 1.4.8：CJK ≤40 字/行、拉丁 ≤80；≈34em 惯例同量级）。默认 17 → 680dp；上限字号 30 → 1200dp。断点不新增常量——这是 reader 角色的任务级规则，§5.5 允许；限宽天然只在宽屏生效（手机 390 宽下 680>390 不触发）。

**接法（推荐）**：`NovelReader` 增加 `maxContentWidth`（或 stage 层计算后传入）。`_relayout` 的布局 viewport 用 `Size(min(w, cap), h)`（layout 缓存键含 viewport，天然一致）；`_NovelPage` 的 Column 包 `Center+ConstrainedBox(maxWidth: capped)` 居中；`GestureDetector`/`zoneForTap` 仍用 `constraints.maxWidth` 全宽 —— **手势区全宽、文字列居中**，页边点击照常翻页，无死区。

替代方案（更简单但有取舍）：stage 层 `Center+ConstrainedBox` 包整个 `NovelReader`——零内核改动，但页边留白变成无响应死区（中心 chrome 切换也失效）。不推荐。

## C. 设置项统一与弹层迁移（§4.5 条 3、条 6 + §5.3/§5.5）

- 共享设置模型 `NovelReaderSettings` + 设备级 `NovelReaderSettingsStore` 两端同用（设置是设备舒适度偏好，不分账号——现状注释已述，保留）。
- `_showReaderSettings`（456-518）+ `_SettingsSliderRow`（601-646）随舞台共享；已走 `showAppBottomSheet`（commit f3e41b9 迁过），M3 默认 modal sheet `maxWidth:640`（Flutter bottom_sheet.dart:1511）——宽屏自动限宽，无需自建。
- **FormState = immediate**：改即生效+立即持久化（`_applySettings` 写穿，199-202）；符合 §5.7「immediate 不显示应用/保存」。失败保留：SharedPreferencesAsync 写失败目前 `unawaited` 静默 —— 可加 `.then` 失败 snackbar 或至少注释承认；门§6「异步动作保持现有可观察错误」。
- 信息顺序：字号 → 行距 → 主题 chips 保持；**修范围不一致**：sheet slider 1.1–2.2 vs `NovelReaderSettings` clamp 1.3–2.4（L485-488 vs settings L66-67）——统一到 clamp 常量（slider 直接读 `minLineHeight/maxLineHeight`，字号同理读 `minFontSize/maxFontSize`）。
- 「背景」= 主题 palette 4 预设，两端同享；本地 AppBar/ListTile 随舞台消失。
- 「翻页」= 内核横向 PageView + 30/40/30 点按区，两端本就同一内核——parity 后本地获得中心点按开 chrome 的行为（此前死区）。
- 目录（底栏）：在线可由 `novel.markup.blocks` 取 `NovelChapterBlock` 列表 + `seriesId` 系列条目构成目录 sheet（`showAppBottomSheet`，条目点击→`handle.goToPage(pageIndexForAnchor)`/`openNovel`）；本地 TXT 无章节标记 → **按钮隐藏或禁用态**，不做 TXT 章节启发式（扩范围）。

## D. 可操作进度 + 可取消（§4.5 条 4）

现状：底栏进度是只读文本（424-434）；`handle.goToPage` 已存在未被消费（reader L211/277-284）。

建议形态（对齐 legado/Shaft 底栏惯例）：

- 底栏进度文本变可操作：tap 打开进度 sheet（`showAppBottomSheet`），内含 Slider `0..pageCount-1` + 当前 `页/总页 · %` + （在线且 markup 有章节时）章节跳转。
- **可取消语义（§5.4 cancel 只取消当前操作）**：sheet 打开时捕获当前页；拖动 slider 只更新预览标签（**不驱动 PageView**——预览不落页就自然不会写锚点）；释放/确认才 `handle.goToPage(target)`；「取消」/下滑关闭 = 原位不动。若选「实时跟手预览」方案，则取消时必须 `jumpToPage` 回捕获页——实现更重，不推荐。
- `handle.goToPage` 现固定 `animateToPage`（L279-282）：跨大页距跳转会跑长动画。扩展 handle：`goToPage(page, {bool animate})`，进度跳转用 `jumpToPage`（即时）或短动画——内核小扩，属本包自有文件。
- 持久化时机：仅 `onPageChanged`（用户真实落页）触发的 `onAnchorChanged` 写进度；sheet 预览不碰。

## E. 「打开信息不记已读」的状态边界（§4.5 条 4 后半）

当前实际：

1. **打开即写进度**：首次布局落定 → `_notifyAnchor` → `_persistAnchor`/`_persistCursor` —— 在线写 `{p0,0}` 进 `NovelProgressStore`（占 LRU 200 一格），本地写 `read_offset=0`。这与 W6「本地书库主操作是继续阅读」冲突：`readOffset` null↔0 无法区分「未开过」与「打开没读」。
2. 历史：`HistoryVisibility` 在 sheet 打开时经 `didPushNext` 暂停（modal sheet 是路由），时长不计入 —— 已正确，保持。
3. 信息弹层本身不产生 anchor 变化 —— 已正确，保持。

**建议边界**（须写进 implement.md 的显式决策）：

- 进度持久化仅在「用户发起的页变更」时发生：即 anchor 与 `initialAnchor`/上次已存值不同，且来源是翻页/跳转而非首次布局落定。实现：`onAnchorChanged` 回调里跳过「等于恢复锚点的首次回调」；或内核加 `onUserPageChanged` 区分。打开信息 sheet、打开又关闭、看完第一页未翻页 → 都不写。
- 本地 `readOffset` 保持 null 直到首次真实翻页 → W6「继续阅读」可正确区分未读/在读。
- 决策点：若产品要「打开过就进继续阅读列表」，则反向——记 0 偏移并在 UI 侧把 0 当「未开始」。**倾向前者**（不打开不写），因为 §4.5 明文「不把打开信息记作已阅读」。

## F. 增量能力（§4.5 条 2）

- 在线：收藏 `BookmarkSwitchButton(isNovel:true)`、分享（ShareService）、作者（信息 sheet `AuthorSummary`→`openUser`）、评论（信息 sheet→`openNovelComments`）—— 全部现有，迁移进舞台动作槽即可。信息 sheet 的 `TagChip` 动作化/非交互化属 **W1 条目**，W5 只搬不修。
- 本地：文件信息 sheet——标题、`localNovelsChars(charCount)`、`author`（可为空则省）、`encoding`、导入日期；`path` 是技术值，按 §5.7/W8「技术 URI 下沉」原则放次级或不放。无需新 l10n 的概念复用 `localNovels*` 键；新增「文件信息」标签键。
- 系列 bar/adjacent bar 仅在线 → spec.seriesBar slot；本地 null。

## G. 其他修复随舞台顺带落地

- 本地页 AppBar 标题 `localNovelsTitle` → 舞台顶栏显示书名（一致性顺手解决）。
- 本地页常驻 ListTile 信息行被页脚 tip + chrome 取代。
- `_NovelStatusScaffold` 的状态页结构两端可共享同一模式（本地已有 async.when 三态）。

## H. 明确不做（防范围蔓延）

- 正文搜索、朗读、自动翻页、音量键翻页、翻页动画模式、纵向滚动模式、简繁转换、TXT 章节启发式、内嵌插图渲染（`pixivimage`/`uploadedimage` 当前 displayText='' 不渲染）、本地书进浏览历史。
- `local_novels_page.dart` 库页改造（继续阅读主操作、删除入更多）归 **W6**；本包只动 `local_novel_reader_page.dart`。
- W1 owning：PopScope 显式返回语义、TagChip 交互、本地 initialAnchor 转换的兜底——W5 只消费。
