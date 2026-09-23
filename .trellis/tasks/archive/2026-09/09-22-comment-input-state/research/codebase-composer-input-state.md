# Codebase: CommentComposer 输入器现状（键盘/表情/贴图互斥）

文件：`lib/features/comments/comment_input.dart`（235 行），基线 `main@8067b2d`。

## 1. 状态载体现状

| 状态 | 载体 | 行号 |
|---|---|---|
| 文本 | `TextEditingController _controller` | :33 |
| 焦点 | `FocusNode _focusNode` | :34 |
| 面板 | `_CommentComposerPanel? _panel` ∈ {null, emoji, stamps} | :6、:35 |
| 本地忙 | `bool _busy` | :36 |
| 外部忙 | `widget.sending`（来自 store mutation key） | :25、:38 |
| `_disabled` | `widget.sending \|\| _busy` | :38 |

**没有「keyboard」这一显式状态**：键盘开 = `_focusNode.hasFocus` 的副作用，代码从不读取。四态（none/keyboard/emoji/stamp）当前实际是「三态面板 + 隐式焦点」，这是 Astra #20 的根因。

## 2. 互斥缺口（逐条核对）

- `_togglePanel`（:185-188）：`_focusNode.unfocus()` + 切换 `_panel`。开面板会收键盘 ✅；但**反向不对称**——面板开着时点输入框，`TextField` 正常获焦弹键盘，`_panel` 不清空：键盘上方同时挂着 emoji/stamp 网格（「打开表情会收键盘，但输入框重新获焦不会对称关闭面板」已在 traceability §4-20 确认）。
- `_insertEmoji`（:190-204）：插入 `(name)` token、光标移到 token 后，然后 `setState(() => _panel = null)` 关面板但**不恢复焦点**——键盘不会回来，用户必须再点输入框（Astra #20「插入表情后不恢复焦点」确认）。
- 无返回拦截：面板开着时系统返回直接 pop 路由（页面无 `PopScope`）。
- `ScrollViewKeyboardDismissBehavior.onDrag`（:157）只作用于面板内 GridView，不解决互斥。

## 3. 面板网格现状（Astra #20 列数问题）

`_buildPanel`（:151-183）：
- emoji：`SizedBox(height: 210)`，`SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 10, spacing 8)`，38 项（comment_assets.dart:3-42），cell = `InkResponse` + `Image.asset(commentEmojiAsset(name))`（:167-173）。
- stamp：`SizedBox(height: 250)`，`crossAxisCount: 5`，40 项（:44-85），cell = `InkResponse` + `Image.asset(commentStampAsset(id))`，**点击即发送**（`_sendStamp`，:177、:223-234）。
- 固定列数 → 320dp 下 emoji cell ≈ 26dp（远低于 44-48dp 最小点击区）；宽屏不增列。
- 现有 widget 测试固定断言 10/5 列（test/comments_replies_test.dart:563-599）——改响应式列数必须同 commit 更新该测试 + spec 中「10 columns / 5 columns」表述（state-management.md:302-303）。

## 4. 发送路径细节

- `_sendText`（:206-221）：`trim` 空串直接 return；`_busy` 置位 → `await widget.onSend(text)` → 成功 `clear()`；异常走 `onError`；`finally` 复位。
- `_sendStamp`（:223-234）：同构，成功关面板。
- 发送键（:134-140）：`IconButton(tooltip: commentSend)`，disabled 条件 `_disabled || text.trim().isEmpty`；stamp 键仅当文本为空才渲染（:121 `if (_controller.text.trim().isEmpty)`），`onChanged` setState 驱动可见性。
- `TextField`：`minLines:1 maxLines:5`、`textInputAction: newline`（:85-91）——发送不走 IME action，只走按钮。

## 5. 回复引用条现状

- `widget.replyTo != null` 时渲染一行 `Text('${commentReplyTo}: $replyTo')` + 关闭 `IconButton`（:57-78）；disabled 时关不了（:73 `onPressed: _disabled ? null : onCancelReply`）。
- 只是纯文本行，不是 chip；`Expanded`+ellipsis 处理长名 ✅。
- 屏幕阅读器：该行无 Semantics 包装；关闭按钮有 tooltip。

## 6. 无障碍/焦点现状

- emoji/stamp 网格 cell：`InkResponse`+裸 `Image.asset`，**无 label、无 button 语义**（Flutter `InkResponse` 只暴露 onTap/onLongPress 语义动作，不自动带 label——flutter ink_well.dart:1401 核对）。TalkBack 下读到的是无意义可点项。
- `IconButton` 都有 tooltip（emoji :110、stamps :123、send :135、cancel-reply :72）✅。
- `_ActionPill`（comment_item.dart:299-350）有 icon+label 文本，InkWell 提供 tap 语义，但无 `button` trait。
- `CommentItem` 无任何 `Semantics` 包装；焦点顺序 = 视觉顺序（列表逐条 → composer）。
- 键盘 Tab/Enter：TextField `newline` action，桌面端无 Enter-to-send；W7 运行矩阵要求键盘等价路径（design.md §7）。

## 7. 键盘高度/insets 现状

- 页面 Scaffold 默认 `resizeToAvoidBottomInset: true`；无键盘高度缓存，无 `MediaQuery.viewInsetsOf` 使用（全仓库仅 search_page.dart:552 一处手动 padding）。
- 面板高度写死 210/250，与键盘实际高度无关 → 键盘↔面板切换有明显跳动。

## 8. 相关既有资产

- `comment_assets.dart`：38 emoji 名、40 stamp id（资产顺序即清单顺序，注释警告勿乱序）；`commentEmojiAsset`/`commentStampAsset` 路径函数。
- `comment_text.dart`：渲染 `(name)` 为 WidgetSpan（:22-46），未知 token 保留原文。
- `material_ui` 是 pub 包（pubspec.yaml:40 `material_ui: ^1.2.0`），测试须 import 它而非 `flutter/material.dart`（quality-guidelines §material_ui shadows）。
