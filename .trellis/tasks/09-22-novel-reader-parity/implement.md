# 执行计划：在线与本地小说阅读器一致（W5）

需求见 `prd.md`，技术设计见 `design.md`，逐文件精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实；
**实现期以 W1 合入后 rebase 的行号为准**）。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
`python3 tool/gen_l10n_lookup.py`。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test test/novel_reader_test.dart test/novel_reader_chrome_test.dart \
  test/novel_reader_settings_test.dart test/local_novel_page_test.dart \
  test/novel_markup_hardening_test.dart
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-novel-reader-parity
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：
> 先按 spec 判定噪声再排查。本地书测试沿用 `_pumpUntil` 真实异步等待与
> `databasePath` 注入模式。

本机不可验项（宽屏实机行长、TalkBack/Narrator、桌面焦点让渡、大 TXT 重排
性能、超大文件内存、CRLF 渲染）在 PR body 标"未验证"。

## 阶段 0：Rebaseline 门禁（W1 硬依赖，无产品 commit）

- [ ] **前置确认**：W1（`task/09-22-interaction-outcome-correctness`）已合入
      main；记录其 merge SHA 到本任务 notes。W1 未合入前不得创建实现分支——
      PopScope 显式返回、`readOffset→initialAnchor` 转换、信息 sheet `TagChip`
      与本包同文件且归 W1 owning。
      验证：`git log main --oneline` 含 W1 merge commit。
- [ ] **分支与复核**：`git switch -c task/09-22-novel-reader-parity`（自 W1 后
      的 main）→ `task.py start`；逐条复核 research 行号对新基线：
      `novel_page.dart` 显式返回已是 `pop()`、`local_novel_reader_page.dart`
      已消费 `initialAnchor` 转换函数（记录其实际位置/返回类型——core 层
      record 还是 features 层 `NovelAnchor`）、信息 sheet `TagChip` 终态。
      若 W1 落地形态与 research 假设不符（如转换函数签名差异），回规划更新
      本文件再继续。
- [ ] **基线测试**：跑上面聚焦测试清单记录真实基线；确认
      `novel_reader_chrome_test.dart` 当前断言语义（W1 版双路径 back）。

## 阶段 1：内核扩展（`novel_reader.dart`；`novel_layout.dart` 预期零改动）

- [ ] **K1 goToPage 可跳页**：`NovelReaderHandle.goToPage` 扩为
      `void Function(int page, {bool animate})?`（默认 true 保持现行为）；
      `animate:false` → `_pageController.jumpToPage`。
      测试：`novel_reader_test.dart` 用例——`goToPage(animate:false)` 后单帧
      pump 即到位（无 `MotionTokens.fast` 动画序列）。
      提交：`feat(novel): 阅读器跳转接口支持无动画跳页`
- [ ] **K2 锚点通知区分来源**：内核标记锚点通知来源（恢复/commit postFrame
      回显 vs `onPageChanged` 用户翻页；恢复那次程序性 `jumpToPage` 的
      `onPageChanged` 须标为非用户）。签名实现期定（`onAnchorChanged` 加
      source 参数或新增 `onUserAnchorChanged`），契约=舞台只为用户翻页写盘。
      测试：首次布局回显标非用户、tap 翻页标用户、settings 变更重排回显标非用户。
      提交：`feat(novel): 锚点回调区分恢复回显与用户翻页`
- [ ] **K3 排版失败可见**：`NovelReader` 加可选 `layoutEngine`/`budget` 注入缝；
      `_relayout` 捕 `NovelLayoutBudgetExceeded`（及排版确定性异常）→
      `_layoutError` 态 → `FeedError(title/error/retryLabel/onRetry)`，
      retry = `_scheduleLayout(force: true)`。
      测试：注入小 budget 触发超限 → `FeedError` 渲染 + retry 再调引擎。
      提交：`fix(novel): 排版预算超限渲染错误态而非未捕获异常`
- [ ] **K4 章节列表暴露**：handle（或布局回调）暴露只读章节表
      `List<({String title, int pageIndex})>`——来自已 commit layout 的
      `pages` 中 `chapterTitle != null` 项；无章节/无布局 → 空表。
      测试：含 `[[chapter:]]` markup 的实体布局后返回有序 (title,pageIndex)。
      提交：`feat(novel): 阅读器命令面暴露章节列表`
- [ ] **K5 行长限宽**：`LayoutBuilder` 内排版 viewport 宽 =
      `min(constraints.maxWidth, settings.fontSize * 40 + 2 * horizontalPadding)`
      传入 `_scheduleLayout`（`NovelLayoutKey.viewport` 随之收窄，缓存键一致）；
      `_NovelPage` 文字列 `Center` 居中（列宽=限宽值）；`zoneForTap`/
      `GestureDetector` 仍按 `constraints.maxWidth` 全宽分派。
      测试：1200 宽下 `key.viewport.width` == 上限、文字列居中、
      页边（x<30%区在限宽列外的留白处）tap 仍翻页；390 宽下 viewport 不变。
      提交：`feat(novel): 宽屏正文按字号相对限宽居中`

## 阶段 2：共享舞台抽取 + 在线等价迁移（`novel_page.dart` → `novel_reader_stage.dart`）

- [ ] **S1 舞台抽取**：新建 `lib/features/novel/novel_reader_stage.dart`，搬入
      `_NovelReaderStage` 全部状态机（`_prefsReady` 闸、`_ChromeBar`、PopScope、
      页脚 tip、顶/底栏、`_SettingsSliderRow`、`_NovelSeriesBar`/
      `_NovelAdjacentBar`、信息/设置弹层）+ `NovelReaderStageSpec` +
      `ReaderProgressBinding`（design.md §二签名）。`novel_page.dart` 壳只留
      `_novelDetailProvider` + `_NovelStatusScaffold` + spec 装配：
      `topActions=[share, bookmark]`、`infoSheet=作品信息`、`bodyWrapper=
      HistoryVisibility`、`progress=NovelProgressStore 适配器`。
      可观察行为零变化：`novel_reader_chrome_test.dart` 不改断言仍绿
      （W1 语义版）；`layering_test` 无新边。
      提交：`refactor(novel): 抽取共享阅读舞台，在线页改为 spec 装配`
- [ ] **S2 进度写时机门（D1）**：舞台 `onAnchorChanged` 仅对用户翻页来源调
      `binding.save`；恢复回显、设置重排回显、sheet 开关均不写。
      测试：打开（无记录）→ store 无写入；打开（有记录）→ 无写入；
      tap 翻页 → 写一次新锚点；开/关信息 sheet → 无写入。
      提交：`fix(novel): 打开阅读器不再写入未阅读的进度`
- [ ] **S3 设置 sheet 语义统一**：随舞台共享；slider 范围改读
      `NovelReaderSettings.minFontSize/maxFontSize/minLineHeight/maxLineHeight`
      （行距 1.1–2.2 → 1.3–2.4，消死区）；信息顺序字号→行距→主题不变；
      `_applySettings` 写穿失败 → `showAppSnackBar` 可见（替 `unawaited` 静默）。
      测试：slider min/max 断言 == 常量；save 抛错的 prefs fake → snackbar
      出现且设置内存态已应用。
      提交：`fix(novel): 阅读设置范围对齐模型约束，持久化失败可见`

## 阶段 3：本地接入（`local_novel_reader_page.dart`）

- [ ] **L1 本地 spec 装配**：删常驻 `AppBar`/`ListTile`；`body` →
      `NovelReaderStage(spec)`：`novel=_entityFor(novel,text)`（沿用合成实体）、
      `topActions=[]`、`infoSheet=文件信息 builder`、`bodyWrapper=null`、
      `progress=本地 binding`（`load` = `readOffset` → W1 转换函数 →
      `NovelAnchor?`；`save` = `_persistCursor` 现算法 anchor→charOffset →
      `updateReadOffset`）。chrome/设置/页脚/返回优先级即刻两端一致。
      测试：`local_novel_page_test.dart` 改写——无 `localNovelsTitle` AppBar、
      书名在 chrome 顶栏、中区 tap 开 chrome、设置 sheet 可达；
      **导入后 `readOffset==null`、打开不翻页退出仍 null（D1/W6 契约）、
      翻页后非 null 且=页首偏移**；有记录重开恢复到上次位置（W1 接线验证）。
      提交：`feat(localnovel): 本地阅读器接入共享舞台`
- [ ] **L2 文件信息 sheet**：`showAppBottomSheet` + 标题/字数
      （`localNovelsChars`)/编码/导入日期；`author` 恒 null 不渲染该行；
      `path` 技术值放次级或不放。新增 l10n 键（文件信息、编码、导入时间等，
      四语 + gen-l10n + lookup）。
      测试：sheet 渲染各字段；author null 时无作者行。
      提交：`feat(localnovel): 本地小说文件信息弹层`

## 阶段 4：新增能力与宽屏（stage 层，两端同得）

- [ ] **N1 可操作进度 + 目录 sheet**：底栏进度文本 → `InkWell`/`TextButton`
      （a11y 沿用 `novelReadingProgress`）→ 进度 sheet
      （`showAppBottomSheet`）：Slider `0..pageCount-1` 只更新预览标签
      （`页/总页 · %`）、确认 → `handle.goToPage(target, animate:false)`、
      取消/下滑原位不动；`handle` 章节表非空时同 sheet 列章节条目
      （点击直接跳页）——本地无章节 → 目录区不渲染（D4）。
      新增 l10n 键（目录/进度跳转等）。
      测试：拖 slider 时 `handle.currentPage` 不变；确认后跳到目标页且
      进度写一次（落页即用户翻页语义）；取消后页码不变；章节条目点击跳
      对应 `pageIndex`。
      提交：`feat(novel): 可操作进度跳转与目录弹层`
- [ ] **N2 键盘翻页**：舞台层 `Focus(autofocus)` + `onKeyEvent`
      `←/→` → 前/后页（`detail_image_pager.dart:44-79` 先例）；sheet 打开时
      焦点自然让渡给路由，不另加守卫。
      测试：`sendKeyEvent(LogicalKeyboardKey.arrowRight/Left)` 翻页断言。
      提交：`feat(novel): 方向键翻页`

## 收尾

- [ ] l10n 四语键全部就位（gen-l10n + `gen_l10n_lookup.py` 产物已提交）；
      `flutter analyze --no-pub`、聚焦测试与全量 `flutter test`（噪声按 spec
      判定）、`git diff --check`、`task.py validate` 全绿。
- [ ] 宽屏/键盘/屏幕阅读器/大 TXT 等本机不可验项在 PR body 逐项标"未验证"，
      不以 widget test 推断通过。
- [ ] PR：`gh pr create --fill`；CI 绿后 `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后提交）。

## 边界（不做）

- TXT 章节启发式、正文搜索/朗读/自动翻页/音量键/翻页动画模式/纵向滚动/
  简繁转换、嵌入图渲染、本地书入浏览历史（D6）。
- `local_novels_page.dart` 库页与"继续阅读"列表项改造归 **W6**；本包只为
  其固定 `read_offset` null↔0 数据源语义（R6）。
- PopScope 显式返回、`readOffset→initialAnchor` 转换函数、信息 sheet
  `TagChip` 归 **W1**，本包只消费。
- `lib/app/` 任何新基件；`NovelReaderCommitGate` 行为不改；routes、
  `local_novels` schema、网络层不动。
