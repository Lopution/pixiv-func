# 在线与本地小说阅读器一致（Roadmap W5）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图，
§4.5 为本包范围定义）。
规划基线：`main@8067b2d`（产品代码 == `origin/main@3c6c1bd`）；实现基线 = **W1
（interaction-outcome-correctness）合入后的 main**。全部代码断言已由本 leaf
`research/codebase-*.md` 在基线 HEAD 逐条复核。

## Goal

把在线小说（`NovelPage`）与本地 TXT 小说（`LocalNovelReaderPage`）收敛到同一个
沉浸式阅读舞台：一套 chrome/设置/进度/恢复状态机，数据源差异只通过显式适配缝
表达。复用现有 `NovelReader` 内核，不建平行阅读器；不改网络/数据层语义。

## Requirements

### R1. 同一阅读舞台（父 §4.5 条 1、条 5）

现状：在线端 `_NovelReaderStage`（`lib/features/novel/novel_page.dart:147-849`）
持有 chrome 显隐、`_prefsReady` 闸、PopScope、页脚 tip、顶/底栏、设置弹层全部
状态机；本地端 `_LocalNovelReaderBody`（`local_novel_reader_page.dart:54-114`）
只是 `Scaffold + AppBar + ListTile + NovelReader`，无 chrome、无设置、无页脚。

- 抽取共享 `NovelReaderStage`（新文件 `lib/features/novel/novel_reader_stage.dart`；
  两个消费者同在 `features/novel/`，不满足 `lib/app/widgets/` ≥3 消费者准入，
  见 directory-structure.md 与决策 D8）。
- 页面壳只负责数据获取与 spec 装配：`NovelPage` → `fetchDetail`；
  `LocalNovelReaderPage` → `repository.get` + `File.readAsString` + `_entityFor`。
- 舞台统一持有：`_prefsReady` 闸（settings + 恢复位置就绪后才挂 `NovelReader`，
  避免默认排版后二次重排）、chrome 滑入滑出（隐藏即零命中零语义）、中区点按切换、
  页脚 `标题 · 页/总页 · %`、顶栏（返回 + 标题 + 动作槽）、底栏
  （目录 + 可操作进度 + 设置）、PopScope。
- PopScope/显式返回语义直接消费 W1 修后契约（显式 back 离页、系统返回先关
  chrome），本包不重做。

### R2. 数据源缝：`NovelReaderStageSpec` + `ReaderProgressBinding`

内核已数据源无关（本地端以合成 `NovelEntity` 适配证明，`local_novel_reader_page
.dart:97-114`）。差异收敛为一个 spec 对象与两方法持久化绑定（签名以 design.md 为准）：

- `novel`、`topActions`（信息按钮前的附加动作）、`infoSheet`（信息弹层 builder）、
  `progress`（`ReaderProgressBinding.load()/save(anchor)`）、`bodyWrapper`
  （在线的 `HistoryVisibility` 包裹）。
- 在线适配器：`NovelProgressStore.read/write`（`<accountId>:<novelId>` → `{p,o}`，
  `reader_settings.dart:139-195`）。
- 本地适配器：`save` = 现有 `_persistCursor` 正向算法（anchor→字符偏移，
  `local_novel_reader_page.dart:86-95`）；`load` = 消费 W1 落地的
  `readOffset→anchor` 转换（含边界/失效处理，W1 owning，本包只接线不重写）。
- 系列/前后篇 bar 由舞台按实体字段（`seriesId`/`seriesPrevId`/`seriesNextId`）
  自渲染——本地合成实体这些字段恒 null，天然无入口，spec 不暴露该槽。

### R3. 本地阅读器接入舞台

- 删除本地页常驻 `AppBar`（标题误显示 `localNovelsTitle` 页面名）与 `ListTile`
  信息行；书名进舞台顶栏与页脚 tip。
- 本地端获得：中心点按开 chrome（此前中区是死区）、阅读设置、可操作进度、
  页脚、系统返回先关 chrome。
- 状态页保留 `async.when` loading/error/data 三态（无 restricted 概念）；
  文件读失败走既有 provider error 分支。

### R4. 阅读设置统一（父 §4.5 条 3、条 6；§5.3 FormState）

- `NovelReaderSettings` + 设备级 `NovelReaderSettingsStore` 两端同用（设备舒适度
  偏好不分账号，现状注释已述，保留）。
- 设置弹层已走 `showAppBottomSheet`（`novel_page.dart:456-518`）——本包做的是
  **语义对齐而非入口替换**：信息顺序保持 字号 → 行距 → 主题 chips 不变，
  immediate 语义（改即生效 + `_applySettings` 写穿）不变。
- **修范围不一致**：sheet 行距 slider 现 `1.1–2.2`（L485-488）vs 模型 clamp
  `1.3–2.4`（`reader_settings.dart:66-67`）——下段 1.1–1.3 是拖动死区、上段
  2.2–2.4 不可达。slider 直接读 `NovelReaderSettings.minFontSize/maxFontSize/
  minLineHeight/maxLineHeight` 常量，一处定义两端同守（决策 D7）。
- immediate 失败可观察：`unawaited(...save())` 静默失败（`novel_page.dart:201`）
  改为失败时经 `showAppSnackBar` 可见（父 §6"异步动作保持现有可观察错误"），
  不落盘的值不留假象。

### R5. 可操作进度与目录（父 §4.5 条 4 前半；§5.4 cancel）

现状：底栏进度是只读文本（`novel_page.dart:424-434`）；`handle.goToPage` 已存在
但无调用方（`novel_reader.dart:211, 277-284`）。

- 底栏进度文本升级为可操作控件：tap 打开进度 sheet（`showAppBottomSheet`），
  内含 Slider `0..pageCount-1` + 当前 `页/总页 · %` 标签。
- **可取消语义**：sheet 打开时捕获当前页；拖动 slider 只更新预览标签、**不驱动
  PageView**（预览不落页自然不会写锚点）；确认才 `handle.goToPage(target)`；
  「取消」/下滑关闭 = 原位不动（决策 D2，对齐 legado/Shaft 底栏惯例）。
- `handle.goToPage` 扩为 `goToPage(page, {bool animate})`：进度跳传用
  `jumpToPage`/短动画，避免跨数百页跑完整 `animateToPage` 帧序列
  （内核小扩，本包自有文件）。
- 目录：在线且 markup 含 `NovelChapterBlock` 时，进度 sheet 列章节表
  （条目 → `goToPage`；页码来自 layout pages 的 `chapterTitle`，内核需暴露
  只读章节列表）。本地 TXT 无章节 → **不渲染目录入口**（按内容条件隐藏，
  非禁用态；在线无章节作品同理；不做 TXT 章节启发式，决策 D4）。

### R6. 进度写入时机：只在真实翻页后落盘（父 §4.5 条 4 后半）

现状缺陷：首次布局落定 → `_notifyAnchor` → 在线写 `{p0,0}` 进
`NovelProgressStore`（占 LRU 200 一格），本地写 `read_offset=0`——
"打开即记已读"。

- 持久化仅在**用户发起的页变更**时发生：首次布局恢复回显、设置变更重排回显、
  打开/关闭信息弹层均不写。实现方向：内核锚点通知区分来源
  （恢复/commit 回显 vs 用户翻页 settle），舞台只对用户翻页调 `binding.save`
  （决策 D1；恢复跳转那次程序性 `jumpToPage` 的 `onPageChanged` 也须标为非用户）。
- **对 W6 的契约含义（显式写入）**：本地 `read_offset` 语义变为
  `null = 从未打开`，`非 null = 用户真实翻过页`（值 = 当前页首字符偏移）。
  W6 库页"继续阅读"主操作据此区分未读/在读（进度百分比 = offset/charCount，
  W6 research 已按此口径规划）。本包**不**采用"打开记 0 + UI 过滤"的反向方案。
- 在线侧同理：打开→关闭不再产生 `{p0,0}` 占位记录；重开从头读起（与现状
  可观察行为一致，只是不落盘）。
- 历史计时保持现状：`HistoryVisibility` 在 sheet 打开时经 `didPushNext` 暂停，
  已正确，回归测试钉住。

### R7. 宽屏行长限宽（父 §4.5 条 3 后半；§5.5 ResponsiveContent）

现状：`_maxWidth = viewport - 48`（`novel_layout.dart:878-882`）无上限，
宽屏单行可超百字。

- 上限 = `fontSize × 40` 文本宽（WCAG 1.4.8：CJK ≤40 字/行 ≈ 拉丁 ≤80 字符；
  默认 17 → ≈680dp，上限字号 30 → 1200dp）。字号相对上限随设置滑块自缩放，
  天然只在宽屏生效（390dp 手机下 680>390 不触发）；不新增断点常量、不用
  固定 dp + `AppBreakpoints` 门（决策 D3；reader 角色的任务级规则，§5.5 允许）。
- **接法（固定）**：缩的是**排版 viewport**——`_scheduleLayout` 收到的
  viewport 宽 = `min(约束宽, fontSize×40 + 2×horizontalPadding)`，使
  `NovelLayoutKey.viewport` 与缓存键天然一致；`_NovelPage` 文字列
  `Center` 居中；`GestureDetector`/`zoneForTap` 仍按 `constraints.maxWidth`
  全宽 30/40/30 分派——**手势区全宽、文字列居中**，页边留白照常翻页，无死区。
- 明确不采用：stage 外层 `Center+ConstrainedBox` 包整个 `NovelReader`
  （零内核改动但页边变无响应死区、中心点按失效）。

### R8. 排版/读文件异常可见（新行为，写进 PRD）

现状：`_relayout` 只吞 `ApiCancelled`（`novel_reader.dart:485-487`），
`NovelLayoutBudgetExceeded`（>4M 字符 TXT 可触发，`novel_layout.dart:40-59`
预算）冒泡为未捕获异步错误；本地可导入任意大 TXT。

- `NovelLayoutBudgetExceeded`（及排版管线其他确定性失败）接到 `FeedError`
  态：标题 + 错误 + `retryLabel`/`onRetry`（retry = force relayout；
  预算超限确定性再失败可接受，与 404 重试同理）。
- 为可测试，内核暴露 `NovelLayoutEngine`/budget 注入缝（可选参数，
  不改现默认行为）。
- `File.readAsString` 超大文件内存压力属引擎外风险，本包不承诺 OOM 兜底，
  记入"未验证/已知限制"。

### R9. 桌面键鼠等价路径（决策 D5 纳入）

- 键盘 `←/→` 翻页，两端经共享舞台一次获得（先例
  `detail_image_pager.dart:44-79` `Focus`/`onKeyEvent`；父 §1"手机手势有
  桌面可见控件、鼠标或键盘等价路径"）。
- 鼠标点击沿用现有三区分派，无需额外工作；**滚轮翻页不做**（滚动语义属
  翻页模式范畴，超范围）。

## Acceptance Criteria

- [ ] 同一契约测试对在线 spec 与本地 spec 各跑一遍（父 §6"W4/W5 门禁"：
      字号、主题、恢复、进度跳转同断言）；`novel_reader_chrome_test.dart`
      既有断言（tap 开 chrome、系统返回先关 chrome、inset 几何、W1 修后的
      显式返回）保持绿或按 W1 语义同步更新。
- [ ] 进度写时机：打开（有/无恢复记录）→ 不写；首次翻页 → 写一次；
      打开再关信息/设置 sheet → 不写；设置变更重排 → 不写；进度 sheet
      预览拖动 → 不写、确认落页 → 写。
- [ ] 本地 `read_offset`：导入后 null；打开不翻页退出仍 null；翻页后非 null
      且=页首偏移；越界/陈旧值恢复走 W1 转换（本包只验证接线，转换本身
      归 W1 测试）。
- [ ] 行距 slider 范围 == `NovelReaderSettings.minLineHeight/maxLineHeight`
      （1.3–2.4），字号 slider == 12–30；设置写失败路径有 snackbar 断言。
- [ ] 限宽：1200dp 宽下 `NovelLayoutKey.viewport.width` == 字号上限值，
      文字列居中，页边 tap 仍翻页（zoneForTap 按全宽分派）。
- [ ] `NovelLayoutBudgetExceeded` → `FeedError` 可见 + retry 触发重排。
- [ ] `goToPage(animate:false)` 远距离跳页即时到位（无动画帧序列）。
- [ ] 进度 sheet：拖动不落页（断言 `handle.currentPage` 不变）、确认跳页、
      取消原位；有章节时章节条目点击跳对应页；本地无目录入口。
- [ ] `←/→` 键翻页 widget 测试；移动端触控三区回归。
- [ ] 单页 / 多页 / 超长文档（预算内大 TXT）三档布局验证；手机 390 宽
      行为零变化（min() 不触发）。
- [ ] 每 commit 对应 implement.md 一个勾选框；分支
      `task/09-22-novel-reader-parity`；`flutter analyze --no-pub`、
      相关 `flutter test`、`git diff --check`、`task.py validate` 全绿；
      运行时矩阵不可验项在 PR 标"未验证"。

## Decisions（规划定案）

- **D1** 进度写时机 = 仅用户真实翻页（内核通知带来源标记，舞台按源过滤）。
  反向方案"记 0 + UI 过滤"不采用——父 §4.5 明文"不把打开信息记作已阅读"，
  且 null↔0 语义是 W6 继续阅读的数据源契约。
- **D2** 进度跳转 = 独立 sheet + 预览不落页；不做底栏内联 slider、不做
  实时跟手预览（取消须跳回捕获页，实现更重且收益不增）。
- **D3** 限宽 = `fontSize×40` 字号相对上限，缩排版 viewport 进 `NovelLayoutKey`；
  不用固定 dp/断点门，不用外层 ConstrainedBox。
- **D4** 目录入口按内容条件渲染（有 `NovelChapterBlock` 才出现）；本地 TXT
  天然无入口，不做章节启发式。
- **D5** 键盘 `←/→` 翻页纳入（共享舞台一次接入两端受益、有同库先例）；
  滚轮/翻页动画模式/纵向滚动明确不做。
- **D6** 本地书不入浏览历史维持现状（history 是账号域概念）；`bodyWrapper`
  本地传 null。W6 若做本地历史/继续阅读入口，数据源是 `local_novels` 表。
- **D7** slider 范围对齐模型 clamp 常量（行距 1.3–2.4、字号 12–30），
  不放宽 clamp 本身。
- **D8** 共享舞台落地为 feature 内文件 `lib/features/novel/novel_reader_stage.dart`；
  不进 `lib/app/widgets/`（2 消费者 < 3 准入线），不预建任何未来基件（§4.11）。
- **D9** 本地文件信息 sheet 字段：标题、`localNovelsChars` 字数、`encoding`、
  `importedAt`；`author` 导入时恒 null（repository L95）不渲染该行的承诺；
  `path` 是技术值，按 §5.7/W8"技术 URI 下沉"原则放次级或不放。

## Out of scope

- TXT 章节启发式；正文搜索、朗读、自动翻页、音量键翻页、翻页动画模式、
  纵向滚动模式、简繁转换；嵌入图渲染（`pixivimage`/`uploadedimage` 当前
  `displayText=''`，维持纯文本）；本地书进浏览历史。
- `local_novels_page.dart` 库页改造（继续阅读主操作、删除入更多、进度变体）
  归 **W6**；本包只动 `local_novel_reader_page.dart`。
- W1 owning：PopScope 显式返回语义、`read_offset→initialAnchor` 转换及其
  兜底、信息 sheet `TagChip` 交互——本包只消费其合入后的契约。
- 排行/对象条目组件（W6）、查看器（W4）、设置页表单（W8）、任何
  `lib/app/` 公共基件、网络/持久化 schema 变更。
