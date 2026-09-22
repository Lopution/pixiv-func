# Implementation Draft — W7 comment-input-state

对应 design.md §4.7 六条 + §5.3/5.6/5.7 契约 + implement.md §6 W6/W7 门禁。
基线 `main@8067b2d`。所有行号见 codebase-*.md。

## 0. Owning files（预估）

- 改：`lib/features/comments/comments_page.dart`、`comment_input.dart`、`comment_item.dart`（仅语义包装，如需要）、`test/comments_replies_test.dart`、`lib/l10n/app_*.arb` + 重新生成 l10n（若加键）。
- 不改：`lib/core/comments/*`（store/actions/repository 契约已满足非乐观发送；失败保留草稿已在 UI 层）、路由表、共享组件。
- 消费（不创建）：W4 的 `lib/app/` 触觉封装、`showAppSnackBar`（已在用）、`AppBreakpoints`。
- 相邻不动：评论翻译 overlay、删除确认 dialog。

## 1. §4.7「根评论与回复同一滚动语义 / 可折叠摘要」

**建议方案：根评论+标题并入回复 ListView**（而非可折叠摘要——摘要在长根评论下收益相同但多一层交互，且「回复对象由引用条持续表达」已由 chip 承担）。

- `_CommentFeedView` 增加 `Widget? header` 参数（comments_page.dart:272-284）：`itemCount = comments.length + 1 + (header == null ? 0 : 1)`；index0=header（`Column`[根 `CommentItem` + 「Replies」标题行]），末尾仍 `FeedTail`。
- `CommentRepliesPage.build`（:172-214）去掉外层 `Column` 中的固定 `CommentItem`/`Padding`，把两者包进 header 传给 `_CommentFeedView`；根评论的 `onReply`/`onDelete` 原样传入。
- 根评论滚出视口后回复上下文靠 composer 引用条表达（§4.7 原意，Astra #19 后半句）。
- 注意：`NotificationListener` 的 extentAfter 触发与 `PullToRefresh` 不受影响；根评论参与下拉刷新/回顶属自然结果。
- 测试：长根评论（多行内容）下回复列表首帧仍可见 ≥1 条回复；header 计入 itemCount 不破坏 `FeedTail` 索引。

## 2. §4.7「回复目标由引用条持续表达，点击回复主动聚焦」

- 引用条升级为 chip 语义：`Semantics(container: true)` + 文本 + 关闭按钮（保留现有 tooltip）。回复页 `_replyTarget ?? root` 的默认行为保留（:166）——回复页引用条恒显示，取消时回到 root（:206 现状合理，即「回复根评论」是默认态）。
- **主动聚焦**：页面持有 `GlobalKey<CommentComposerState>`（State 类改 public）或注入一个 `VoidCallback` 注册；推荐前者，改动最小：
  - `onReply: (c) { setState(() => _replyTarget = c); _composerKey.currentState?.focusForReply(); }`
  - `CommentComposerState.focusForReply()`：`_inputState = keyboard; _focusNode.requestFocus()`（内部处理面板→键盘切换，见 §3）。
- 备选：向 composer 传 `replyRequestId`（int，每次点击自增），`didUpdateWidget` 检测到递增即聚焦。无需公开 State，但多一个计数器语义。两案皆可，建议 GlobalKey（命令式、可测试、无假状态）。

## 3. §4.7「四态互斥 none/keyboard/emoji/stamp，返回先关面板」

### 状态载体

```dart
enum CommentComposerInputState { none, keyboard, emoji, stamp }
// _CommentComposerState 内：
CommentComposerInputState _inputState = CommentComposerInputState.none;
double _cachedKeyboardHeight = 0; // viewInsets 峰值缓存
```

替换现有 `_CommentComposerPanel? _panel`（:6,:35）。`_busy`/`sending` 属发送态（§5.3 action），不并入此枚举——四态只管输入面。

### 转换规则（对齐 Shaft BottomPanelCoordinator 语义）

| 事件 | 转换 |
|---|---|
| 点 emoji 钮 | emoji↔none；进入 emoji：`unfocus()` |
| 点 stamp 钮 | stamp↔none；进入 stamp：`unfocus()`；emoji→stamp 直接换内容（同区域同高） |
| `_focusNode` 获焦（listener） | 面板态 → keyboard；none → keyboard |
| `_focusNode` 失焦且未进面板 | keyboard → none |
| `focusForReply()` | → keyboard + requestFocus |
| 系统/页面 back | `PopScope(canPop: !_isPanelOpen)`；拦截时 `_inputState = none`（不 unfocus 已由面板态保证无焦点）；keyboard 态**不拦截**（IME/系统先处理，与 Shaft `backCallback` 仅 PANEL 时 enable 一致） |
| 插入表情 | 见 §5 决策点 |
| 发送 stamp 成功 | → none（现状 :228 保留） |

互斥本质是「focusNode listener 收敛键盘态 + 面板枚举排他」，避免再引入第二个真假源（quality-guidelines「A second state machine beside a framework one」——focusNode 是框架权威，`_inputState` 是 UI 唯一事实源，listener 只做单向同步）。

### insets/高度处理（对齐仓库现有约定）

- 两页 `Scaffold` 设 `resizeToAvoidBottomInset: false`（:50、:170；先例 home_page.dart:160、search_page.dart:34）。
- composer 底部区域统一为：`bottomExtent = max(viewInsets.bottom, panelVisible ? panelHeight : 0)`，面板挂在 input row 下方的 `SizedBox(height: bottomExtent, child: panel)`：
  - keyboard 态：`viewInsets.bottom` 逐帧驱动高度 → 列表被键盘自然顶起（视觉等价 resize，但不压缩页面布局层级；因 body 不 resize，需给**列表**底部 padding `8 + bottomExtent`，先例 search_page.dart:552）。
  - 键盘→面板：先以当前 `viewInsets.bottom` 作面板高摆上屏，再 unfocus → 无跳变；`viewInsets` 回落后面板用 `_cachedKeyboardHeight`。
  - 面板→键盘：`requestFocus` 后面板保留渲染直到 `viewInsets.bottom >= panelHeight` 再卸载（Shaft `finishSwitchToKeyboard` 等价物）——也可用 `bottomExtent = max(...)` 的写法让键盘覆盖面板，面板在键盘完全升起后按状态卸载。
  - `_cachedKeyboardHeight`：build 中 `if (insets > _cachedKeyboardHeight) _cachedKeyboardHeight = insets`；未采样到时回退常量（Shaft 270dp / chat_bottom_container 300dp；本仓库建议 280 + `SafeArea` 底 padding 由面板自己承担）。
- 面板高度语义：emoji/stamp 面板统一 = `max(_cachedKeyboardHeight, fallback)`，不再用 210/250 写死。
- OEM 注意（Shaft 注释教训）：判键盘可见以 `viewInsets.bottom > 0` 配合 focusNode.hasFocus，而非纯高度阈值；`MediaQuery.viewInsetsOf` 只在需要的子树读取避免整页订阅逐帧重建（本仓库多处注释提及此约定）。

## 4. §4.7「发送成功后 reply target 行为固定」

**建议：清除**（Shaft `clearReplyIfUnchanged` 先例；Astra #19 要求明确规则）。

- 根页：`_send` 成功后 `_replyTarget = null` → 引用条消失，后续评论回顶层。
- 回复页：成功后 `_replyTarget = null` → `replyTarget ?? root` 回落为根评论，引用条仍显示（回复页默认态=回复根评论）。
- 守卫：发送期间用户改了目标则不覆盖——`_send` 入口快照 `target`，成功后 `if (identical(_replyTarget, target) || _replyTarget?.id == target?.id)` 才清。文本草稿同理由 composer `clear()` 只在成功路径执行（现状已对，测试固定）。
- 失败：`failSend` 后 mutation 留 failed 态（store 已有），UI 走 SnackBar；草稿与 reply target 全部保留 ✅。
- 测试固定：成功→目标清除+文本清空；失败→文本与目标保留；发送中改目标→成功后不清新目标。

## 5. §4.7「网格列数 + 插入后焦点」

- 列数：`LayoutBuilder` 包 GridView，`crossAxisCount = (constraints.maxWidth / minExtent).floor()`，emoji `minExtent ≈ 48`（clamp 3–10），stamp `minExtent ≈ 96`（clamp 2–5，图片需可辨识）；spacing 保持 8。320dp→emoji≈6 列（现状 10 列 cell≈26dp 不达标），840+→10 列封顶。
- **插入后焦点（决策点）**：两案——
  - A. 插入即关面板 + `requestFocus()` → keyboard（贴合 §4.7「恢复正确焦点」字面）；
  - B. 保持 emoji 面板开着继续连发（PixEz 行为），光标已正确推进。
  - **建议 A**：spec 措辞明确「插入后恢复正确焦点」，且现状的缺陷正是「关面板+丢焦点」的组合；连发场景靠重开 emoji 钮一键可达。实现：`_insertEmoji` 末尾 `_showKeyboard()`（state=keyboard + requestFocus），移除 `setState(_panel = null)` 裸关。
- 无障碍：每个 cell 包 `Semantics(button: true, label: name /* 或 'stamp $id' */)`，图片 `excludeFromSemantics`；面板容器 `Semantics(container: true)`。

## 6. 发送态（§4.7「发送中可见」+ §5.3 action）

- send IconButton 内容在 `_disabled`（即 busy）时换成 `SizedBox(18×18, CircularProgressIndicator(strokeWidth:2))`——先例：bookmark pending 用 CupertinoActivityIndicator 24px（state-management.md:74），评论按钮槽位更小用 18px Material 指示器即可；`Semantics(label: l10n 新增 'sending')`。
- `sending`（store pending）与 `_busy`（await 期间）重叠保留，不合并——store key 含 target id，换目标后 pending 即消失是预期行为。
- 无显式「重试」控件：失败后用户重按发送=重试同一操作（§5.7「重试是重试同一操作」由按钮天然承担）；SnackBar 文案沿用 `commentSendFailed`/`commentPermissionDenied`，回复页补齐 permission 分支（:232-235 现状不区分）。

## 7. 触觉消费点（§5.6 门禁）

- 消费点：发送成功终态（文本与 stamp 同）→ W4 封装的「明确震动」档（参考 PixEz `mediumImpact`）。位置：页面 `_send`/`_sendStamp` await 无异常返回后（与「清除 reply target」同点），不在 composer——保持 composer 无业务副作用。
- **W4 未合入前不得先建封装**（§4.11/§5.6「不另建来源」「禁止先落地未使用的公共封装」）。W7 实现分支基于含 W4 的 main rebase；若 W4 接口名未定，以 W4 合入物为准，禁止在 W7 内新建 `lib/app/*haptic*`。
- 冗余通道：触觉关/不支持时，成功已由「文本清空+新评论 prepend 上屏+目标清除」视觉可辨 ✅。

## 8. 屏幕阅读器/键盘可达路径（§7 矩阵：评论输入器是优先路径）

- 焦点顺序：列表项（avatar→reply/translate/delete/replies pill）→ composer 输入框 → emoji 钮 → stamp 钮 → 发送钮 →（面板开时）网格 cell。需在真机 TalkBack/Narrator 验证并记录；未验证就写未验证（§7）。
- 网格 cell 补 label（§5）；回复 chip 整体可朗读 + 关闭按钮独立可达。
- 键盘等价：TextField 保留 newline；面板/emoji 网格 cell 用 `InkResponse`（可被 Tab 聚焦）✅；建议补 `FocusTraversalGroup` 默认顺序即可，不引入自定义 policy（YAGNI）。
- Escape/back：面板态 Escape 关面板属桌面增强，可并入 `CallbackShortcuts` 于 composer 顶层——可选项，标注为增强。

## 9. 测试清单（聚焦）

- widget：四态互斥矩阵（emoji→tap field→keyboard、stamp→back→none 等）、320/390/600dp 下列数、插入表情后 focusNode.hasFocus && 面板关、发送中 spinner、失败草稿保留、成功 reply target 清除（两页各自语义）、回复点按后聚焦。
- 更新既有：comments_replies_test.dart:563-599 固定 10/5 列断言改为宽度驱动断言；spec `state-management.md:302-303`「10/5 columns」表述同步修订（质量门：spec 与代码同 commit）。
- 静态：`flutter analyze --no-pub`、`dart format`、`git diff --check`、`task.py validate`。
- 运行时矩阵记录：320/390/600/840/1200dp、中文+长翻译（ru 文案最长）、TalkBack 一条路径、reduced motion、触觉开/关。

## 10. 术语核对（§5.7）

- 「发送」=一次性持久化动作（action 类），无「保存/应用」歧义 ✅。
- chip 关闭按钮语义是「取消回复」——终止当前操作不离页，符合「取消只终止当前操作」。
- 「删除评论」不可恢复，沿用 `commentDelete` ✅；删除确认 dialog 不动。
