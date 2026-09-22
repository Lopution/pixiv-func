# 技术设计：确定性动作后果与返回行为修复（W1）

设计基线：`main@8067b2d`。九项修复的逐文件精确改动方案、代码草案与测试点见
`research/implementation-draft.md`；逐项现状核实（行号）见 `research/codebase-*.md`；
决策点与运行时证据缺口见 `research/risks.md`。本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

默认单 stage（`task/09-22-interaction-outcome-correctness`）。触发拆分条件时按
`research/risks.md` §stage 拆分建议切成 `…-s1`/`…-s2`：stage1={R1,R2,R3,R5}（novel+profile），
stage2={R4,R6,R7,R8,R9}（search+settings+login）。

| 组 | 条目 | 耦合面 | 风险 |
|---|---|---|---|
| A | R1 显式返回 / R2 标签可操作 / R3 本地恢复 / R5 资料编辑返回 | novel + profile，互不重叠 | 中（PopScope SDK 行为依赖） |
| B | R4 热门标签 / R6 测试改名 / R7 凭据清除 | search + settings，纯局部 | 低 |
| C | R8 反向搜图取消 / R9 WebView 动作 | controller + 两端 WebView | 中（状态机/桌面不可验） |

## 二、关键契约

### BackAndCancel（父 §5.4 在本 leaf 的落点）

- 系统返回关闭临时层（chrome/sheet）优先于离页；显式页面 back 直接离页——两条路径
  在 novel_page 必须同时成立：`PopScope` 管系统返回，显式按钮走 `Navigator.pop()`。
- 取消只终止当前操作（R8 `stopSearch` 原地停搜）；`reload`/`重新登录`/`返回` 三动作
  语义互异（R9：reload=重载当前页、重新登录=beginSession 新会话、返回=pop）。
- 脏草稿才提示丢弃（R5 `hasUnsavedChanges` 短路）。

### 术语（消费父 §5.7 种子）

- R6 "应用并测试"、R9 "重新加载"/"重新登录"、R8 ready 态"取消"保留；
  l10n 新 key 流程：`app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` → `python3 tool/gen_l10n_lookup.py`。

### 状态机边界

- R8 `stopSearch` 是 `searching|preparing|picking → ready|idle` 的合法转换；`canceled`
  态仍归全量 `cancel()`；generation 单调递增保证交错调用安全。
- R7 三态互斥由 `_saving` 保证：spinner(in-progress)/primary(success)/error 色(failure)。

### 锚点逆映射（R3）

`readOffset → NovelAnchor('p<i>', col)`：逐段累加，`offset <= 段末` 归该段；
恰落换行处归下一段段首（与 `_persistCursor` 正映射互逆）；`>= text.length` 夹取末段末页。
`initialAnchor` 仅在首次布局消费（novel_reader.dart L433），同步计算无异步竞态。

## 三、复用与禁止

- 复用：`TagChip`、`Sheet.of(context).close()`、`openSearchResults`/`NovelSearchQuery`、
  `showAppDialog`、`profileSessionProvider.hasUnsavedChanges`、现有 l10n 工具链。
- 禁止：新 provider、新路由、overlay/断点/触觉等公共基件、网络层改动、
  `ResumeAnchor` 复用于阅读进度。

## 四、测试策略

- 纯函数单测（R3 锚点、R8 generation/stopSearch）+ widget 测试（R1/R2/R5/R6/R7/R9）。
- 可 override 点齐备：`translationCredentialStoreProvider`、`pixivNetworkFactoryProvider`、
  `oauthServiceProvider`、`platform.pickImage`；桌面 WebView 不建 fake（D7）。
- 既有按文案定位的测试（'Cancel' tooltip、'取消'）须同步更新。

## 五、运行时证据缺口（PR 标"未验证"）

Android predictive-back 真实手势链、Windows WebView2 reload/重启、TalkBack/Narrator
朗读、1.3x 大字体错误卡双按钮溢出、真机 picker 中 stopSearch。
