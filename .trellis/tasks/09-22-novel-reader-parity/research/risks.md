# W5 风险与开放决策点

## 依赖/边界风险

1. **W1 同文件竞合**：`novel_page.dart` 的 PopScope 显式返回、`local_novel_reader_page.dart` 的 initialAnchor 转换、信息 sheet 的 TagChip 均归 W1（design §4.1/§4.11）。W5 在 W1 合入前动手会双分支改同一文件 → 必须串行：W1 先合 main，W5 再 rebase 后实现。W1 叶子当前只有 prd 骨架（planning），排期不确定是本包最大进度风险。
2. **W6 反向依赖**：「本地书库继续阅读」要读 `readOffset`；「打开即写 0」会让未读与在读不可分 → E 节边界决策影响 W6 的可用性，宜在本包 PRD 写明。
3. **共享舞台 <3 消费者**：两调用点同在 `features/novel/`，抽 `NovelReaderStage` 为 feature 内私有 widget 合规；**不要**提前放进 `lib/app/widgets/`（directory-structure.md 准入 ≥3）。
4. **l10n 高冲突文件**（§8）：目录/跳转/文件信息/取消等新键会碰 `app_*.arb` + generated——与并行叶子冲突时按 owner 合入顺序串行。
5. **chrome 测试回归**：`novel_reader_chrome_test.dart`（270 行）断言 tap 开 chrome、系统返回先关 chrome、inset 几何——舞台抽取必须保持这些可观察行为；W1 改 PopScope 后断言语义会随之更新。

## 技术风险

6. **`_relayout` 只吞 `ApiCancelled`**（novel_reader.dart:485-487）：`NovelLayoutBudgetExceeded`（>4M 字符 TXT 可触发，layout L40-59）及 `File.readAsString` 超大文件的内存/OOM 无兜底——本地书用户可导入任意大 txt；建议本包至少把预算异常接到 FeedError 态（新行为，需写入 PRD 范围）。
7. **行长限宽的 viewport 双轨**：若只缩 `_NovelPage` 渲染宽度而不缩排版 viewport，TextPainter 行切分仍按全宽——必须让 `NovelLayoutKey.viewport` 吃到限宽后的值（缓存键一致性），并让 `_NovelPage` 居中。只包外层 ConstrainedBox 会让页边手势死区。
8. **`handle.goToPage` 固定 animateToPage**：跨数百页跳转的长动画；需加 `animate` 参数或 `jumpToPage` 分支——内核签名小改，测试要跟上。
9. **`didUpdateWidget` 的 force relayout**（L293-301）：settings/主题每次变更整本重排——本地大 TXT 下设置弹层拖 slider 会连续触发重排（节流/防抖是否必要需实测；`chunkParagraphs` 让出+commit gate 已防错乱，但 CPU 开销真实存在）。
10. **CRLF 尾巴**：本地 `_entityFor` 用 `split('\n')`，Windows 文本每段尾带 `\r`——排版把 `\r` 当可见宽字符的风险（TextPainter 对 \r 渲染为零宽通常安全，但未验证）；offset 换算自洽不受影响。
11. **本地 author 恒 null**（repository L95 导入未解析文件名/元数据）——文件信息 sheet 别承诺显示作者。
12. **进度 sheet 的取消语义**：若实现成「slider 实时驱动 PageView」，取消必须跳回捕获页；推荐「预览不落页」方案规避整组问题。
13. **`_NovelSeriesBar.markSeen` 在 build 里写 watchlist 游标**（L804-813 postFrame）——迁移时保持原时序，别把 side effect 提前或丢掉（追更语义归此）。
14. **SharedPreferences 写失败静默**（`unawaited` L201/209-218）——设置/进度写失败用户无感；§6 要求异步错误可观察，至少 snackbar 或注释承认现状。

## 开放决策点（须进 PRD/design）

- **D1** 「打开信息不记已读」落地形态：跳过首次布局锚点写（推荐）vs 记 0 偏移+UI 侧过滤。影响 W6 继续阅读数据源。
- **D2** 进度跳转 UI 形态：底栏内联 slider vs 独立 sheet；推荐 sheet（可放章节列表、取消语义干净）。
- **D3** 行长限宽常量：`fontSize×40` 字号相对（推荐，随字号缩放、天然只咬宽屏）vs 固定 dp（如 680/720）+ AppBreakpoints 门。
- **D4** 本地目录按钮：隐藏 vs 禁用+空态文案；TXT 章节启发式不做。
- **D5** 桌面键鼠等价（←/→ 翻页、滚轮）：detail_image_pager.dart:44-79 有 Focus/onKeyEvent 先例；§4.5 未明文要求，建议列为可选增强或明确排除。
- **D6** 本地书是否入浏览历史：现状不入（history 账号域）；W6 若要给本地书做历史/继续阅读入口，数据源是 `local_novels` 表而非 history——本包不改。
- **D7** 阅读设置行距 slider 范围对齐 clamp（1.3–2.4）还是放宽 clamp——一处常量为准。
