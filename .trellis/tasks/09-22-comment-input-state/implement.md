# 执行计划：评论输入状态机与回复页滚动收敛（W7）

需求见 `prd.md`，技术设计见 `design.md`，逐文件精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实）。

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
flutter test test/comments_replies_test.dart   # 聚焦
flutter test                                   # 收尾跑全量
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-comment-input-state
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：看似不相关的
> 测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec 判定噪声再排查。

本机不可验项（真实 IME 动画切换、TalkBack/Narrator、OEM IME、真机触觉、1.3x
大字体、横屏、reduced motion、俄语长文案）在 PR body 标"未验证"，不得由
widget test 推断通过。

## 阶段 0：依赖门禁（阻塞性，无产品提交）

- [x] 确认 W1 `09-22-interaction-outcome-correctness` 已合入 `main`——
      §5.4 返回契约基线；本包 `PopScope`/back 语义建立在其上。
      **核实记录（2026-09-23）**：W1 以 PR #56 与 #57 合入
      （merge `2ebf2b8`、`b8eafb3`），早于本分支基线。
- [x] 确认 W4 `09-22-artwork-viewer-series-flow` 已合入 `main` 且其 `lib/app/`
      触觉薄封装存在（预期 `lib/app/haptics/`）——记录实际 API 名与「明确
      震动」档调用签名，以 W4 合入物为准。**W4 未合入则本任务到此停止**；
      禁止在 W7 内新建 `lib/app/*haptic*`（§4.11/§5.6）。
      **核实记录**：W4 以 PR #60 合入（merge `8e8e1ed`，即本分支基线）。
      实际封装 = `lib/app/haptics/app_haptics.dart`
      `abstract final class AppHaptics`，按动作角色暴露
      `select()`/`confirm()`/`success()`/`error()` 四个静态方法。
      「发送终态·明确震动」档调用签名 = **`AppHaptics.success()`**
      （内部 `HapticFeedback.mediumImpact`，80ms 节流；
      W4 design §2.1 契约面写死「保存/发送成功 → success」）。
      测试观察缝：无注入式 observer，但 `HapticFeedback.vibrate` 走
      `SystemChannels.platform`，可在 widget test 用
      `defaultBinaryMessenger.setMockMethodCallHandler` 计数；
      `AppHaptics.debugReset()` 清节流。全库 `HapticFeedback` 仅
      `app_haptics.dart` 一处命中（`configure` 挂在 `app.dart:125`）。
- [x] Rebaseline：记录当时 `main` SHA；按 `research/codebase-*.md` 复核
      owning files 行号（W1/W4 若动了相邻文件需更新引用）；确认追踪矩阵
      #19/#20 未被其他已合入任务吸收；运行
      `flutter test test/comments_replies_test.dart` 记录真实基线。
      **核实记录**：基线 `main@8e8e1ed`（research 标称 `8067b2d`）。
      `git diff 8067b2d..8e8e1ed` 显示 `lib/features/comments/`、
      `test/comments_replies_test.dart`、`state-management.md` 零改动
      ——research 行号对本 HEAD 无漂移；漂移仅在 l10n 生成物与新增
      `app_haptics.dart`。追踪矩阵 #19/#20 无其他合入任务认领。
      基线测试：`flutter test test/comments_replies_test.dart`
      **12/12 全绿**（含既有固定 10/5 列断言 :563-599）。
      设计勘误（评审定案优先于本文件）：A2「列表底部 padding
      `8 + bottomExtent`」被 design.md「避让责任唯一」条款取代——
      composer 参与布局即全部避让机制，列表底部只留常数 `8`。
- [x] `git switch -c task/09-22-comment-input-state` 后
      `python3 ./.trellis/scripts/task.py start .trellis/tasks/09-22-comment-input-state`。
      **核实记录**：分支 `task/09-22-comment-input-state`，启动记账
      `3494e1a`，worktree `/root/Pixiv-func-w7`。

## 阶段 1：组 A — composer 四态状态机与 insets（R4）

- [ ] **A1**：`lib/features/comments/comment_input.dart`——引入
      `enum CommentComposerInputState { none, keyboard, emoji, stamp }` 替换
      `_CommentComposerPanel? _panel`（:6、:35）；`initState` 挂 `_focusNode`
      listener 单向收敛键盘态（获焦→keyboard、失焦未进面板→none）；
      `_togglePanel` 对称互斥（进面板 `unfocus()`、emoji↔stamp 直换内容）；
      `_insertEmoji` 末尾改为关面板 + `requestFocus()`（:203 裸
      `setState(_panel = null)` 移除）；State 类改 public
      `CommentComposerState` 并暴露 `focusForReply()`（`_inputState =
      keyboard` + `requestFocus`）；composer 顶层 `PopScope(canPop:
      非面板态)`，拦截时 `_inputState = none`，keyboard 态不拦。
      测试：四态转换矩阵（emoji→点输入框→keyboard；面板态 simulated back
      →none 不离页；keyboard 态不拦截；emoji→stamp 直换）；插入表情后
      `focusNode.hasFocus && 面板关`。互斥回归用例先对旧实现跑红留证据。
      提交：`feat(comments): 输入器收敛为 none/keyboard/emoji/stamp 四态互斥`
- [ ] **A2**：两页 Scaffold `resizeToAvoidBottomInset: false`
      （comments_page.dart:50、:170）；composer 底部区域 `bottomExtent =
      max(viewInsets.bottom, panelVisible ? panelHeight : 0)`，面板挂 input
      row 下 `SizedBox(height: bottomExtent)`；列表底部 padding
      `8 + bottomExtent`（search_page.dart:552 先例）；`_cachedKeyboardHeight`
      只在 `focusNode.hasFocus` 期间采样 `viewInsets.bottom` 峰值，回退
      ~280dp；面板高度统一该值（废弃 :154 的 210/250）；`viewInsetsOf` 只在
      需要子树读取。
      测试：MediaQuery 注入 insets——键盘高 300 → 面板高 300、无采样 → 280
      回退、键盘态 bottomExtent 跟随。
      提交：`feat(comments): 评论页改手动 insets，面板高度对齐键盘缓存`

## 阶段 2：组 B — 回复页结构与回复目标（R1/R2/R3）

- [x] **B1**：`_CommentFeedView` 新增 `Widget? header`（comments_page.dart
      :272-284）：`itemCount = comments.length + 1 + (header != null ? 1 : 0)`，
      index0=header，末位仍 `FeedTail`；`CommentRepliesPage.build`（:172-214）
      拆除外层 Column 中固定 `CommentItem`/`Padding`，包成 header（根
      `CommentItem` + `commentReplies` 标题行）传入，根 onReply/onDelete
      原样接入；`PullToRefresh`/`NotificationListener`/`SmoothWheelScroll`
      链不动。
      测试：长根评论下回复列表首帧 ≥1 条回复可见；header 计入 itemCount
      不破坏 `FeedTail` 索引与 loadMore 触发。
      提交：`feat(comments): 根评论并入回复列表滚动`
- [x] **B2**：两页持有 `GlobalKey<CommentComposerState>`；`onReply` 回调
      `setState` 目标后调 `_composerKey.currentState?.focusForReply()`；
      引用条补 `Semantics(container: true)`（关闭按钮 tooltip 与
      `onCancelReply` 不变；回复页 `replyTarget ?? root` 默认态保留）。
      测试：点 `CommentItem` reply pill → composer `focusNode.hasFocus`；
      引用条语义可定位；回复页初始引用条 = 根评论名。
      提交：`feat(comments): 点击回复主动聚焦输入框，引用条补语义`
- [x] **B3**：发送成功清除 reply target——两页 `_send` 入口快照 `target`，
      await 成功后仅当 `identical(_replyTarget, target) ||
      _replyTarget?.id == target?.id` 才 `setState(_replyTarget = null)`；
      根页→回顶层，回复页→回落 `?? root`。
      回复页 `_showMutationError` 补 `CommentPermissionException` →
      `commentPermissionDenied` 分支（:232-235 对齐根页 :97-99）。
      测试：成功→目标清+文本清；失败→文本与目标保留；await 期间改目标→
      成功后不清新目标（三用例，两页各覆盖关键分支）。
      提交：`feat(comments): 发送成功清除回复目标并区分权限错误`

## 阶段 3：组 C — 网格响应式与可达性（R5/R7 网格部分）

- [x] **C1**（同一 commit 三处落地）：`comment_input.dart` 面板 GridView 外包
      `LayoutBuilder`，`crossAxisCount = (constraints.maxWidth / minExtent)
      .floor()` 加 clamp——emoji minExtent≈48 clamp 3–10，stamp minExtent≈96
      clamp 2–5，spacing 保持 8；**同 commit** 重写
      `test/comments_replies_test.dart:563-599` 固定 10/5 断言为宽度驱动断言
      （320→emoji≈6、390、840→10 封顶；stamp 边界）；**同 commit** 修订
      `.trellis/spec/frontend/state-management.md` 的「10 columns / 5 columns」
      （:302-303）与「10/5 grids」（:338）为响应式契约表述。
      提交：`feat(comments): 表情贴图网格按宽度定列数`
- [x] **C2**：emoji/stamp cell 包 `Semantics(button: true, label: emoji 名 /
      stamp id)` + 图片 `excludeFromSemantics`；面板容器 `Semantics(container:
      true)`；新增 l10n key（stamp cell label 带 id 占位，如 `commentStamp`）
      四语 arb + `flutter gen-l10n` + `tool/gen_l10n_lookup.py`。
      测试：cell button/label 语义 finder 断言。
      提交：`feat(comments): 补输入器网格语义标签`

## 阶段 4：组 D — 发送态与终态反馈（R6/R7）

- [x] **D1**：发送 IconButton 在 `_disabled` 期间换
      `SizedBox(18×18, CircularProgressIndicator(strokeWidth: 2))` +
      semantics label（新 l10n key `commentSending`，四语）；`_busy` 与
      `sending` 重叠保留不合并（mutationKey 含 target id 的提前 false 为
      既有契约，`_busy` 兜底，PRD R3 已记录不修）。
      测试：发送中 spinner + semantics 断言；`sending` false 但 `_busy` true
      窗口仍 busy。
      提交：`feat(comments): 发送中按钮内显示进度`
- [ ] **D2**：触觉消费——两页 `_send`/`_sendStamp` await 成功返回后（与清
      目标同点）调 W4 合入封装的「明确震动」档（接口名以 W4 合入物为准，
      参考 PixEz `mediumImpact`）；文本与 stamp 同点同档；composer 内不加
      业务副作用；grep 确认全库无第二 `HapticFeedback` 调用点。
      测试：以 W4 封装可注入/可观察的接缝断言「成功路径调用一次、失败路径
      不调用」；若封装不可注入则静态断言 + PR 标真机未验证。
      提交：`feat(comments): 发送成功接入共享触觉反馈`

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec
      判定）；`dart format lib test`、`git diff --check` 干净；
      `task.py validate` 通过。
- [ ] PR：`gh pr create --fill`——body 显式列「未验证」清单（键盘↔面板
      动画、TalkBack/Narrator、OEM IME、真机触觉、1.3x 大字体、横屏、
      reduced motion、俄语长文案）；CI 绿后 `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后提交）。

## 边界（不做）

- `lib/core/comments/` 数据层与 mutation key 结构；「目标 A 发送中」额外
  UI 表达（PRD R3 记录不修）。
- 公共 bottom-panel/键盘管理基件（消费者不足 3，§5.5）；`lib/app/*haptic*`
  或任何第二反馈通道；`chat_bottom_container` 依赖。
- 评论翻译 overlay、删除确认 dialog、举报/屏蔽（W6/W8）。
- 评论列表 `restorationId`/`PageStorageKey`（恢复等级声明=内存级）。
- 回复页删除根评论后 pop 的既有路径（:262-264）。
- Escape 关面板桌面增强为可选项（`CallbackShortcuts` 于 composer 顶层，
  若做随 C 组提交并标注增强）。
