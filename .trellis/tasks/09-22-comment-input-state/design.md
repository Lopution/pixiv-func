# 技术设计：评论输入状态机与回复页滚动收敛（W7）

设计基线：`main@8067b2d`。逐文件精确改动方案与代码草案见
`research/implementation-draft.md`；现状核实（行号）见 `research/codebase-*.md`；
外部对标（Shaft `BottomPanelCoordinator`、PixEz、`chat_bottom_container`）见
`research/external-panel-keyboard-patterns.md`；决策点与证据缺口见
`research/risks.md`。本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

默认单 stage（`task/09-22-comment-input-state`）。前置硬依赖（父 §4.11 冻结表
「W7 必须等待 W1、W4」）：

- **W1 已合入**：本包 `PopScope`/back 语义建立在 §5.4 契约基线上；
- **W4 已合入**：`lib/app/` 唯一触觉薄封装存在（预期 `lib/app/haptics/`），
  W7 只消费、不自建；接口名/等级以 W4 合入物为准。W4 未合入则本任务
  不得开始产品代码提交。

阶段 0 = 等两包合入 + rebase 到新 main + 复核 research 行号。阶段内提交分组：

| 组 | 条目 | 耦合面 | 风险 |
|---|---|---|---|
| A | 四态状态机 + insets/底部区域（R4） | comment_input.dart + 两页 Scaffold | 高（viewInsets 逐帧、OEM IME） |
| B | header 并入滚动 + 主动聚焦 + 目标清除（R1/R2/R3） | comments_page.dart | 中（FeedTail/loadMore 槽位） |
| C | 响应式列数 + cell/引用条语义（R5/R7 部分） | comment_input.dart + 测试 + spec | 低 |
| D | 发送 spinner + permission 文案 + 触觉（R6/R7） | 两页 + composer + W4 封装 | 低（API 名待 W4 定） |

同文件（comment_input.dart）被 A、C 先后触碰：同一分支内串行提交，无并行冲突。

## 二、关键契约

### 输入状态机（父 §4.7 落点）

```dart
enum CommentComposerInputState { none, keyboard, emoji, stamp }
```

转换表（对齐 Shaft `BottomPanelCoordinator` 语义）：

| 事件 | 转换 |
|---|---|
| 点 emoji 钮 | emoji ↔ none；进入 emoji：`unfocus()` |
| 点 stamp 钮 | stamp ↔ none；emoji→stamp 同区域直接换内容 |
| `_focusNode` 获焦（listener） | 任何面板态 → keyboard；none → keyboard |
| `_focusNode` 失焦且未进面板 | keyboard → none |
| `focusForReply()` | → keyboard + `requestFocus()` |
| 系统/页面 back | `PopScope` 仅拦面板态 → none；keyboard 态不拦 |
| 插入表情 | 关面板 + `requestFocus()` → keyboard |
| 发送 stamp 成功 | → none（现状保留） |

单一事实源：`_focusNode` 是框架权威，listener 只做「焦点 → 状态枚举」单向
同步；`_inputState` 是 UI 唯一事实源，不得在第二处再推导「面板是否开着」。
这满足 quality-guidelines「不在框架状态机旁建第二个状态机」——listener
是镜像收敛而非平行判定。`_busy`/`widget.sending` 属发送态，不进枚举。

### insets / 底部区域契约（Shaft 同构）

- 两页 `Scaffold` 设 `resizeToAvoidBottomInset: false`（先例
  search_page.dart:34、home_page.dart:160；component-guidelines「Common
  Mistakes」明禁 autofocus 页依赖默认 resize）。
- composer 底部区域统一：`bottomExtent = max(viewInsets.bottom,
  panelVisible ? panelHeight : 0)`；面板挂在 input row 下方
  `SizedBox(height: bottomExtent, child: panel)`。键盘升起时 SizedBox 逐帧
  跟随 → input row 始终贴键盘上沿；键盘→面板先以当前 insets 摆上屏再
  unfocus 无跳变；面板→键盘由 `max(...)` 让键盘覆盖面板直到状态卸载。
- 列表底部 padding `8 + bottomExtent`（search_page.dart:552 先例），保证
  键盘/面板开着时末位与 `FeedTail` 重试钮可滚出可点。
- `viewInsets` 只读 `MediaQuery.viewInsetsOf` 权威值：composer 底部区域与
  列表 padding 读同一物理量允许，但不得各自缓存；`_cachedKeyboardHeight`
  是唯一缓存点，采样窗口 = `focusNode.hasFocus` 期间的 `viewInsets.bottom`
  峰值（OEM navBar-in-ime 污染规避），未采样回退 ~280dp。
- `MediaQuery.viewInsetsOf` 只在需要的子树读取，避免整页订阅逐帧重建
  （search_page.dart:31 注释先例）。
- 判键盘可见 = `viewInsets.bottom > 0` 配合 `focusNode.hasFocus`，不用纯
  高度阈值（Shaft OEM 注释教训）。

### FormState（父 §5.3 落点）

- 发送 = **action**（idle/busy/success/error）：busy 由 `_busy || sending`
  表达且 spinner 可辨；error 走 `showAppSnackBar`（#48 共享入口，不新增
  平行反馈通道）。
- 草稿文本 = **draft** 语义：失败保留、成功清除（现状已对，测试固定）。
- 回复目标清除是成功副作用的一部分，带「目标未变更」守卫；不重按即重试
  同一操作（§5.7）。

### BackAndCancel（父 §5.4 落点）

- 系统返回优先级：面板开 → 关面板不离页；键盘开 → 不拦截（系统先收键盘）；
  none → 正常退页。`PopScope` 挂 composer 顶层，`canPop = 非面板态`。
- 引用条关闭按钮 = 「取消回复」：只终止当前回复目标，不离页（§5.7「取消
  只终止当前操作」）。
- 回复页删除根评论成功后 `pop`（:262-264）是代码路径 pop，不经手势拦截，
  与面板 `PopScope` 无嵌套冲突——保留不动。

### 术语（父 §5.7 落点）

- 「发送」= 一次性持久化 action；「取消」仅用于引用条关闭（终止当前操作）；
  「重试」= 重按发送钮，不设独立控件；「删除评论」沿用 `commentDelete`。
- 新增 l10n key（四语 `app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
  `python3 tool/gen_l10n_lookup.py`）：发送中语义（如 `commentSending`）、
  stamp cell label（带 id 占位）。

### 触觉（父 §5.6 落点）

- 唯一消费点：两页 `_send`/`_sendStamp` await 成功返回后（与清目标同点），
  调 W4 封装的「明确震动」档；composer 无业务副作用。
- 冗余通道：触觉关/不支持时成功仍视觉可辨（清空 + prepend + 目标清除）。
- 禁止在 W7 内新建 `lib/app/*haptic*`（§4.11/§5.6「不另建来源、禁止先落地
  未使用的公共封装」）。

## 三、复用与禁止

- 复用：`_CommentFeedView`（扩 `header` 参数）、`PullToRefresh` /
  `NotificationListener` / `SmoothWheelScroll` / `FeedTail` 链路、
  `showAppSnackBar`、`commentEmojiAsset`/`commentStampAsset`、
  `FocusTraversalGroup` 默认序、W4 `lib/app/` 触觉封装、`AppBreakpoints`。
- 禁止：新建 `lib/app/` 下任何 haptic/bottom-panel 公共基件；改
  `lib/core/comments/` 与 mutation key 结构；新增路由/provider；emoji/stamp
  合并为单面板+tab；为「目标 A 发送中」做额外 UI（R3 记录不修）。

## 四、测试策略

- widget 测试为主：四态转换矩阵（枚举 + widget 树断言，不断言动画）、
  header 槽位与 FeedTail/loadMore 索引、点击回复聚焦、插入表情焦点、
  宽度驱动列数（320/390/600/840 各 pump）、spinner/semantics、成功清目标 /
  失败保留 / 发送中改目标三分支、回复页 permission 分支。
- viewInsets 用 `MediaQuery` 包裹或 `tester.binding` 注入模拟；动画顺滑
  不可断言，只断言终态（quality-guidelines「assert terminal state」）。
- 互斥与插入焦点的回归用例先对旧实现跑红留证据（同 spec 回归证明条款）。
- 既有 `comments_replies_test.dart:563-599` 与 `state-management.md` 表述
  同 commit 重写——删/改固化旧行为的断言不留 `skip:`。
- SnackBar 断言须 import `material_ui`（quality-guidelines shadows 条款）。

## 五、运行时证据缺口（PR 标「未验证」）

- 键盘↔面板切换动画顺滑度（真实 IME 动画 widget test 不可复现）；
- TalkBack/Narrator 焦点顺序与 cell 朗读；
- OEM IME（HyperOS navBar 并入 ime）高度采样；
- 真机触觉开/关两档；1.3x 大字体 composer 溢出；横屏 320dp 高；
- reduced motion 下面板切换表现；俄语长文案（最长翻译）。
