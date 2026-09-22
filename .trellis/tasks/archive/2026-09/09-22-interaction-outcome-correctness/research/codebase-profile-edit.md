# Codebase — 资料编辑返回语义（§4.1 第 5 项）

`lib/features/profile/profile_edit_page.dart`：

- L137–167 `_attemptPop()`：
  ```dart
  final session = ref.read(profileSessionProvider).valueOrNull;
  if (session == null) { Navigator.of(context).pop(); return; }   // L139–141
  // L143–162：无 hasUnsavedChanges 判断，直接 showAppDialog「放弃更改？」
  ```
- L175–179 `PopScope(canPop: session == null || !hasUnsaved, onPopInvokedWithResult: (didPop,_) { if (!didPop) _attemptPop(); })`。
- L183–186 AppBar leading：`IconButton(tooltip: context.l10n.cancel, icon: arrow_back) → _attemptPop`。

## 静态事实

- **系统返回**：`canPop = session==null || !dirty` —— 干净时直接弹，脏时被拦 → `_attemptPop` → 弹确认框。系统返回路径**已经正确**。
- **显式返回按钮**：`_attemptPop` 在 `session != null` 时不看 `hasUnsavedChanges`，**无改动也弹「放弃更改？」对话框** —— 违反「无改动直接返回，有改动才确认」。这是 Astra 断言中仍成立的部分。
- **术语**：tooltip 是 `l10n.cancel`（"Cancel"/"取消"），图标是返回箭头，行为是「返回（可能带确认）」—— §5.7 要求无改动的返回不称「取消」。建议 tooltip 统一为 `MaterialLocalizations.backButtonTooltip`（与 novel_page L351 模式一致）或 `l10n.back`。
- `_attemptPop` 修复：`session == null || !session.hasUnsavedChanges` → `pop()`；否则对话框。注意 `session.hasUnsavedChanges` 是 profile controller 上的属性（确认 getter 名：页面 L177 读的是 `session.hasUnsavedChanges`，`valueOrNull` 空时 `hasUnsaved` 局部变量=false）。

## 测试

`test/profile_edit_test.dart` L471–480：现有用例「dirty → 点 Cancel tooltip → 弹框；系统返回 → 弹框」。修复后：
- 该用例的 dirty 分支不变；
- 需新增「干净表单点返回按钮 → 直接 pop、无对话框」用例（leading 按钮 finder 依赖 tooltip 文案，改 tooltip 后 finder 需同步——这是测试改动点）。
- 取消对话框内动作需核对 M3 措辞（`discard`/`cancel` 键已存在）。

## 风险/边界

- `session == null`（资料未加载/加载失败）时无表单可言，`pop()` 正确。
- 保存成功后 `hasUnsavedChanges` 归零 → 返回直接走 pop 路径，无需特判。
- 若实现给 `_attemptPop` 加 dirty 判断，确认 `hasUnsavedChanges` 涵盖全部字段（头像、昵称、简介……）——profile controller 是脏状态的唯一事实源，不要复制字段比较。
