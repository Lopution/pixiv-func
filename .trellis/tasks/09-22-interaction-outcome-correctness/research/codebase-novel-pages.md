# Codebase — 在线/本地小说阅读页（§4.1 第 1–3 项）

核查基线：HEAD `8067b2d6764a2b6ceb22628f3b2c23b86795592b`（静态审查基线 `8215fa8` 之后已有 488e854、d4c5f26、f3e41b9 三次小说相关提交，traceability 文档已覆盖）。

## 1. 在线阅读器「显式返回离页 / 系统返回先关 chrome」

### 现状代码

`lib/features/novel/novel_page.dart`：

- L133 `_NovelStatusScaffold`（加载/失败骨架）：`BackButton(onPressed: () => Navigator.of(context).maybePop())`。该骨架没有 PopScope，`maybePop` 可正常弹栈（除非此页是分支首路由，见风险）。
- L229–233 正文 `PopScope`：
  ```dart
  PopScope(
    canPop: !_chromeVisible,
    onPopInvokedWithResult: (didPop, result) {
      if (!didPop) _hideChrome();
    },
    child: ...
  )
  ```
  语义：chrome 可见时系统返回只关 chrome；chrome 隐藏时系统返回离页。静态审查后的语义本身正确。
- L350–353 chrome `AppBar` 显式返回按钮：
  ```dart
  IconButton(
    tooltip: MaterialLocalizations.of(context).backButtonTooltip,
    onPressed: () => Navigator.of(context).maybePop(),
  )
  ```
  **问题**：chrome 可见时 `canPop == false` → `maybePop()` 返回 `doNotPop` → 走 `onPopInvokedWithResult(didPop: false)` → 只调用 `_hideChrome()`。点击显式返回按钮的效果 = 隐藏承载该按钮的 chrome，永远不弹栈。按钮标签是 backButtonTooltip，结果却是「收起工具栏」——名称与后果不一致，且 `_hideChrome` 后按钮随 chrome 消失，用户需要第二次系统返回才能离页。

### 框架语义（已在 SDK 源码核实，flutter 3.47.2）

- `Navigator.maybePop` 会征询 `route.popDisposition`，`canPop:false` 时走 `RoutePopDisposition.doNotPop`，最终回调 `onPopInvokedWithResult(didPop: false)`（`navigator.dart` L5605–5613 附近）。
- `Navigator.pop`（命令式）不征询 `popDisposition`：go_router `MaterialApp.router` 走 `onPopPage` → `route.didPop` → 仍以 `didPop: true` 触发 `onPopInvokedWithResult`（flutter/flutter#163052 同理）。因此 `pop()` 会绕过拦截且不会重复触发 `_hideChrome`（`!didPop` 守卫）。
- `routes.dart` 全库无 `onExit`，`context.pop()`/`Navigator.pop` 不会被 GoRoute.onExit 拦截。

### 结论

Astra 断言「显式返回与系统返回同语义」**仍然成立**：显式按钮在 chrome 可见时是死动作（只关 chrome）。修复应把 L352 改为命令式 `Navigator.of(context).pop()`（与 `profile_edit_page.dart` L140/165 一致），并保留 PopScope 处理系统返回。`_NovelStatusScaffold` 的 `maybePop` 可保留或同样改 `pop`（该处无拦截）。

### 测试

- `test/novel_reader_chrome_test.dart` L124–170「chrome toggles, back closes chrome first」：用 `tester.binding.handlePopRoute()` 模拟系统返回——已钉住「系统返回先关 chrome」，修复后仍须通过。需新增「显式返回按钮直接离页」用例（在 ReaderHandle chrome 可见时 `tester.tap(find.byTooltip(backButtonTooltip))` 后断言路由栈）。
- 测试基建：`appRootNavigatorKey`/`appRootRouteObserver` 注入、`ReaderHandle`/`NovelReaderHandle` 调试句柄已就绪（同文件 L45–110）。

## 2. 小说标签可操作或非交互

### 现状代码

`lib/features/novel/novel_page.dart` L567–573（`NovelInfoSheet` 内）：

```dart
for (final tag in novel.tags)
  TagChip(label: tag.name, translated: tag.translatedName),
```

`lib/app/widgets/tag_chips.dart` L7–24：`onTap` 为 `VoidCallback?`；`onTap == null` 时 `InkWell` 为 null，但 `Card` 仍带 `outline`/`surfaceTint` 边框和填充，视觉上与可点击 TagChip 完全一致。

对比已可操作的对照：
- `lib/app/widgets/info_block.dart` L52–60（插画详情标签）：`onTap: () => openTagSearch(context, tag.name)`。
- `lib/features/novel/novel_page.dart` L557–559（作者 chip）：`Sheet.of(context).close()` 后 `openUserPage`。
- `lib/features/search/search_page.dart` L181：`NovelSearchQuery(keyword: tag.name)` 已存在且 `_TrendingTagTile` L132 在用。

**Astra 断言成立**：「不可点但带边框」同时违反 §5.7 可操作性与 §5.2 视觉层级（非交互元素不应有边框轮廓）。

### 修复取向

两个合法选项：
- a) `onTap: () { Sheet.of(context).close(); openSearchResults(context, NovelSearchQuery(keyword: tag.name)); }` —— 与 TrendingTagTile/illust info 一致，推荐。
- b) 去掉 `Card` 包装改成纯 `Text` —— 放弃导航价值。

### 测试

`test/novel_reader_chrome_test.dart` L172–206 的 info sheet 测试目前断言 `find.text('#tag1 t1')`；改为可操作后可断言 `openSearchResults` 路由被 push（需 router 或 `context.push` 桩）。

## 3. 本地小说持久化位置 → `initialAnchor`

### 现状代码

`lib/features/novel/local_novel_reader_page.dart`：

- L63–81 `Column[ListTile + Expanded(NovelReader(...))]`：传入 `novel`/`novelRepo`/`onAnchorChanged`/`readerHandle`，**未传 `initialAnchor`**。
- L86–95 `_persistCursor`：把 `NovelAnchor` 转回字符偏移 `readOffset = Σ(前序段落 length + 1) + anchor.offset`，经 `LocalNovelRepository.updateReadOffset` 落库（`local_novel_repository.dart` L126–134；列定义 `local_novel_database.dart` L86 `read_offset INTEGER`）。
- L97–114 `_entityFor`：`text.split('\n')` → `NovelParagraph('p$i', text)`；`contentVersion = 'local:${id}:${text.length}'`。

`lib/features/novel/novel_reader.dart`：

- L433–437：`oldLayout == null` 时才消费 `widget.initialAnchor`（经 `pageIndexForAnchor`），首次布局即恢复页。
- L471：`initialAnchor` 经 `commitGate.forCall(...)` → `_notifyAnchor` 写回进度（恢复页会立即以相同偏移回写，幂等）。

`lib/features/novel/novel_layout.dart`：

- L104–122 `NovelAnchor(paragraphId, offset)`。
- L231–243 `pageIndexForAnchor`：页内区间匹配；未知 `paragraphId` 由 L260–268 `_compareAnchors` 归一化为 index 0（落在首页），不会崩但会静默回起点。
- L245–252 `pageIndexForCharacter(characterOffset)` 已存在（仅供测试/其他调用方），`initialAnchor` 入口只接受 `NovelAnchor`。
- L415–440 `_format` 每段落起始 offset 记录含 +1 换行，与 `_persistCursor` 的算法同构。

**Astra 断言成立**：`readOffset` 被持续写入但从不被读取用于恢复，「位置可恢复」是死语义。

### 逆映射方案（W1 内可实现，纯函数）

`readOffset`（字符偏移）→ `NovelAnchor`：按 `text.split('\n')` 逐行累计 `len + 1`，找到 `offset` 所在行 i → `NovelAnchor('p$i', offset - 行起始)`。边界：

- `readOffset == null` → `initialAnchor: null`（现状语义）。
- `readOffset <= 0` → `null` 或 `('p0', 0)`（等价首页）。
- `0 < offset < text.length` → 所属段落锚点。
- `offset >= text.length`（文件被改短/陈旧）→ 夹取到末段末尾 `('p${n-1}', lastLine.length)` → 落在末页；或判失效回 `null`。建议夹取（用户读过的尾部位置仍最接近），但需在 leaf design 中写明。
- `offset` 落在换行符边界上：`offset` 恰等于行累计值（含 +1）时应归下一行 `('p$i', 0)`，与 `_persistCursor` 的正映射互逆（持久化的锚点 offset ≤ 段长，因为 `page.startAnchor` 只落在段首，不会落在换行处——验证用例覆盖往返即可）。

`updateReadOffset` 不做校验（L127–130），负值/超大值都可能落库 → 读取端必须做全部边界判断。

### 测试基建

`test/local_novel_page_test.dart` L28–72：`localNovelDatabaseProvider` 可用 `LocalNovelDatabase.inMemory()` + sqflite ffi 覆盖，页面级 widget 测试可写库后打开 reader。建议把「offset→anchor」抽成可单测的纯函数（可放 `local_novel_reader_page.dart` 私有或 `core/localnovel/` 下公开 helper——跨包测试需要公开，见 implementation-draft）。
