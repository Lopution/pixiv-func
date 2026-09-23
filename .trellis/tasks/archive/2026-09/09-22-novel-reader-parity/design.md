# 技术设计：在线与本地小说阅读器一致（W5）

设计基线：`main@8067b2d`（产品代码 == `origin/main@3c6c1bd`）；实现基线 =
W1 合入后的 main。逐文件精确改动方案与签名草案见 `research/implementation-draft
.md`；行号核实见 `research/codebase-*.md`；决策点定案见 prd.md Decisions 与
`research/risks.md`。本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

**硬依赖：W1 合入 main 后才允许起实现分支。** `novel_page.dart` 的 PopScope
显式返回、`local_novel_reader_page.dart` 的 `readOffset→initialAnchor` 转换、
信息 sheet 的 `TagChip` 均归 W1（父 §4.1/§4.11）且与本包同文件——并行必然双
分支改同一文件。rebaseline 列为 implement.md 阶段 0 门禁。

默认单 stage（`task/09-22-novel-reader-parity`）。文件所有权集中且互斥：

| 阶段 | 内容 | 主文件 | 风险 |
|---|---|---|---|
| 0 | Rebaseline 门禁（见 implement.md） | — | W1 排期不确定是最大进度风险 |
| 1 | 内核扩展 | `novel_reader.dart`（`novel_layout.dart` 预期零改动） | 中（commit gate 契约不能破） |
| 2 | 共享舞台抽取 + 在线等价迁移 | `novel_reader_stage.dart`（新）、`novel_page.dart` | 高（chrome 测试回归面） |
| 3 | 本地接入 | `local_novel_reader_page.dart` | 中 |
| 4 | 新增能力（进度 sheet/目录/键盘）+ l10n | stage 文件、`app_*.arb` | 中（l10n 高冲突，§8 串行） |

与 W2/W3/W4/W8/W9 文件不重叠，可并行；l10n `app_*.arb` + generated 为高冲突
文件，与并行叶子按 owner 合入顺序串行。

## 二、关键契约

### 数据源缝（舞台 spec）

```dart
class NovelReaderStageSpec {
  final NovelEntity novel;
  final List<Widget> topActions;    // 信息按钮之前的附加动作；在线: share/bookmark，本地: 空
  final WidgetBuilder infoSheet;    // 信息弹层内容；在线: 作品信息，本地: 文件信息
  final ReaderProgressBinding progress;
  final Widget Function(BuildContext, NovelAnchor?, Widget)? bodyWrapper; // 在线: HistoryVisibility；本地: null
}

abstract interface class ReaderProgressBinding {
  Future<NovelAnchor?> load();      // null = 无记录/记录失效 → 首页
  Future<void> save(NovelAnchor anchor);
}
```

- `bodyWrapper` 需要舞台态（当前 anchor）做 history snapshot → 签名携带
  `NovelAnchor?`，由舞台在 build 时喂入。本地恒 null（D6）。
- 系列/前后篇 bar 不进 spec：舞台按 `novel.seriesId/seriesPrevId/seriesNextId`
  自渲染；本地合成实体恒 null 自然无入口。
- `ReaderProgressBinding` 与 spec 类型放 `novel_reader_stage.dart`
  （feature 内）。W1 的 offset→anchor 转换若在 `lib/core/localnovel/` 返回
  记录型 primitives（`NovelAnchor` 是 features 层类型，core 不能 import），
  本地 binding 的 `load` 负责包装成 `NovelAnchor`——转换正确性归 W1 测试，
  本包只测接线。

### 进度写时机状态机（D1 落点）

内核 `_notifyAnchor` 现有两个触发源：`PageView.onPageChanged`（含用户翻页与
恢复那次程序性 `jumpToPage`）与 commit postFrame 显式回显
（`novel_reader.dart:373-377, 476-482`）。契约：

- 锚点通知携带来源（`layoutRestore/commitEcho` vs `userTurn`/`tocJump`/
  `progressJump`/`keyTurn` 等提交源）；恢复跳转的那次 `onPageChanged`
  用内核内标记压成非用户源。`onAnchorChanged` 签名扩展或新增
  `onUserAnchorChanged`——实现期二选一，对外契约=舞台只为**提交源**
  调 `binding.save`。
- **提交路径枚举（写进契约，防止漏判）**：真实手势翻页、目录跳转确认、
  进度 sheet 确认跳页、键盘翻页，都是用户提交位置 → 写；
  恢复回显、force relayout 回显、slider 预览、打开/取消 sheet，
  都不是提交 → 不写。首次提交写一次，之后每次 settle 写一次（同值去重
  沿用 `_anchor == anchor` 短路）。
- **W6 数据源契约**：`local_novels.read_offset` null=**没有已提交的
  阅读锚点**（不称"未打开"——打开本来就不写）；非 null=在读。

### FormState / BackAndCancel（§5.3/§5.4 落点）

- 阅读设置 sheet = **immediate**：改即生效+写穿，无"应用/保存"按钮。
  **持久化失败 = 恢复已确认值 + 可见失败提示**（全应用统一默认，与 W8
  「旧值保持+显示失败」同规则）：写库失败 → 内存态回滚到最后持久化的
  字号/主题、`showAppSnackBar` 报失败。不允许"屏幕保留新值但存储未污染"
  的静默分歧——若某设置确需"仅本次阅读有效"，必须显式命名该状态并写清
  退出条件，不能靠普通错误提示让用户猜。§5.3 的"draft 失败保留"不适用
  （无草稿态）。
- 进度 sheet = 一次性操作：打开捕获页码、slider 只动预览、确认跳页、
  取消/下滑原位不动——cancel 只取消当前操作，不离开阅读页。
- 返回优先级栈：modal sheet 是路由 → 系统返回先关 sheet；无 sheet 时先关
  chrome（PopScope，W1 契约）；chrome 已关才离页。显式顶栏 back 直接离页
  （W1）。三层顺序由 widget 测试钉住。

### 行长限宽（D3 落点）

- `NovelReader.build` 的 `LayoutBuilder` 内：`layoutWidth =
  min(constraints.maxWidth, settings.fontSize * 40 + 2 * style.horizontalPadding)`；
  `_scheduleLayout(viewport: Size(layoutWidth, maxHeight))` → `NovelLayoutKey
  .viewport` 吃到限宽值，缓存键一致；`zoneForTap`/`GestureDetector` 仍读
  `constraints.maxWidth`。
- `_NovelPage`：`Center` + 限宽约束居中文字列（列宽 = 排版 viewport 宽，
  padding 仍在列内）。390dp 下 `min` 不触发 → 移动端零行为差。
- 内核自算（`settings.fontSize` 已是输入），不新增公开参数。

### 异常与状态页边界

- `_relayout` 增加 `on NovelLayoutBudgetExceeded`（连同排版确定性失败）→
  reader 内 `_layoutError` 态 → `FeedError(title, error, retryLabel, onRetry:
  force relayout)`。注入缝：`NovelReader` 可选 `layoutEngine`/`budget` 参数
  供测试注入小预算。
- `_NovelStatusScaffold` 模式两端共享（本地 async.when 三态不变）。
- `HistoryVisibility` 暂停/恢复、`snapshotFromNovel(anchor)` 时序原样搬移；
  `_NovelSeriesBar.markSeen` postFrame 副作用保持原时序（risks #13）。

### 恢复等级声明（§5.1 口径）

- 阅读位置 = 进程死亡后恢复（durable：在线 SharedPreferences `NovelProgressStore`，
  本地 SQLite `read_offset`）——本包最强恢复层。
- chrome 可见性、进度 sheet 预览值 = view-local，不恢复；sheet 自身由路由栈管
  理（内存恢复层）。
- 路由 durable 值不变（`novel/:id`、`local-novels/:id`），不动 routes.dart。

## 三、复用与禁止

- 复用：`NovelReader`/`NovelLayoutEngine`/`NovelAnchor`/`pageIndexForAnchor`
  （`pageIndexForCharacter` 留作 W1/本包 binding 备用）、`showAppBottomSheet`
  （M3 modal 默认 `maxWidth:640`，宽屏无需自写）、`FeedLoading/FeedError/FeedEmpty`、
  `BookmarkSwitchButton(isNovel)`、`AuthorSummary`、`CaptionRichText`、
  `WatchlistToggle`、`HistoryVisibility`、`novelReaderPalette`、`MotionTokens`、
  现有 `novel*`/`localNovels*` l10n 键。
- 禁止：平行阅读器；`lib/app/widgets/` 新基件（<3 消费者）；routes/schema/
  网络层变更；TXT 章节启发式；绕过 `NovelReaderCommitGate` 自写排版缓存；
  第二份 anchor↔offset 换算（本地反向转换用 W1 落地的函数）。
- l10n 新增键（预估）：目录、进度跳转 sheet 标题/确认语义、文件信息及其字段
  标签、设置保存失败 snackbar；走 `app_{en,ja,ru,zh}.arb` → `flutter gen-l10n`
  → `tool/gen_l10n_lookup.py`，与并行叶子串行。

## 四、测试策略

- **契约双跑**：`test/novel_reader_stage_contract_test.dart`（新）把同一组断言
  （chrome 切换、设置滑块生效、恢复锚点、进度跳页、D1 写时机）对在线 spec
  （MockClient 详情，沿用 `novel_reader_chrome_test.dart` 脚手架）与本地 spec
  （sqflite ffi + 临时目录，沿用 `local_novel_page_test.dart` 脚手架）各跑一遍。
- 内核单测：`goToPage(animate:false)` 即时到位；anchor 来源标记
  （恢复回显 vs tap 翻页 vs commit postFrame）；限宽后 `key.viewport.width`；
  budget 注入 → `FeedError`；章节列表暴露。
- 舞台/widget：进度 sheet 预览不落页/确认/取消；目录条目跳转；`←/→` 键；
  设置 slider 范围 == clamp 常量；写失败 snackbar（`SharedPreferencesAsync`
  可注入抛错实现）；本地文件信息 sheet 字段（author null 不渲染）。
- 回归：`novel_reader_chrome_test.dart`（W1 语义版断言）、
  `novel_reader_test.dart`、`novel_reader_settings_test.dart`、
  `local_novel_page_test.dart`（AppBar/ListTile 断言需改写为 chrome 断言）、
  `test/architecture/layering_test.dart`（新文件零跨层边）。
- 本地测试沿用既有坑位：sqflite `databasePath` 注入、`_pumpUntil` 真实异步等待、
  全量跑的 loopback 噪声按 spec 判定。

## 五、运行时证据缺口（PR 标"未验证"）

宽屏（≥840/1200dp）实机行长观感与页边手势、TalkBack/Narrator 读 chrome 与
进度 sheet 焦点顺序、桌面键盘焦点与 sheet 打开时的焦点让渡、大 TXT（数十万
字+）设置滑块连续重排的 CPU 开销（`didUpdateWidget` force relayout，risks #9）、
`File.readAsString` 超大文件内存、CRLF 文本 `\r` 渲染（预期零宽但未实测，
risks #10）。
