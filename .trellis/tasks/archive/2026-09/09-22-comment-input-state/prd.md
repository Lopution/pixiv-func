# 评论输入状态机与回复页滚动收敛（Roadmap W7）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图）。
规划基线：`main@8067b2d`。全部代码断言已由本 leaf `research/codebase-*.md`
在当前 HEAD 逐条复核；外部对标（Pixiv-Shaft `panel/` 模块 `BottomPanelCoordinator`、
PixEz、`chat_bottom_container`）见 `research/external-panel-keyboard-patterns.md`；
Astra 归属条目为追踪矩阵 #19（回复页）与 #20（评论输入器）。

## Goal

收敛回复页滚动结构与共享评论输入器：根评论并入回复滚动语义；回复目标由引用条
持续表达且点击回复主动聚焦；发送成功后回复目标固定为清除；输入面收敛为
none/keyboard/emoji/stamp 四态互斥且返回先关面板；网格按可用宽度与最小点击区
决定列数；发送中可见、失败保留草稿；发送终态消费 W4 触觉封装。

本 leaf 只拥有 `lib/features/comments/`（`comments_page.dart`、`comment_input.dart`、
`comment_item.dart` 的语义包装）与 `test/comments_replies_test.dart`、l10n 键；
不改 `lib/core/comments/`（store/actions/repository 的非乐观发送契约已满足）、
不改路由表、不建公共 bottom-panel 基件（父 design.md §4.11/§5.5，消费者不足 3 个）；
触觉只消费 W4 已合入的 `lib/app/` 薄封装（§5.6），本包不自建第二来源。

前置硬依赖：W1（§5.4 返回契约基线）与 W4（§5.6 触觉 owner）必须先合入 `main`，
门禁见 `implement.md` 阶段 0。

## Requirements

### R1. 根评论并入回复滚动语义

现状：`CommentRepliesPage` 的根评论与「Replies」标题固定在滚动区外
（comments_page.dart:177-192），长根评论 + 键盘顶起时回复列表被挤到很小——
两页 Scaffold 默认 `resizeToAvoidBottomInset: true`（:50、:170）是放大器。

- `_CommentFeedView`（:272-365）新增 `Widget? header` 参数：header 占 itemBuilder
  前置槽位，`itemCount = comments.length + 1 + (header != null ? 1 : 0)`；
  末位 `FeedTail` 槽位与 `extentAfter < 1.2` 视口的 loadMore 触发（:322-330）不变。
- 回复页把根 `CommentItem` + `commentReplies` 标题行包成 header 传入，
  删除外层固定块；根评论的 `onReply`/`onDelete` 原样接入。
- `PullToRefresh` → `NotificationListener` → `SmoothWheelScroll` 包裹链不变；
  根评论随列表参与下拉刷新与回顶属自然结果。
- 根评论滚出视口后回复上下文由 composer 引用条持续表达（R2）；不另做可折叠
  摘要（§4.7 两选一，取并入滚动——摘要在长根评论下收益相同但多一层交互）。

### R2. 回复目标持续表达 + 点击回复主动聚焦

现状：`onReply` 只 `setState` 目标（comments_page.dart:57、:196-197），composer
`_focusNode` 无从感知（Astra #19「点击回复主动聚焦」未满足）；引用条是纯文本行，
无语义包装（comment_input.dart:57-78）。

- 引用条保留并补 `Semantics(container: true)`；关闭按钮保留 tooltip、独立可达；
  回复页默认 `replyTarget ?? root`（:166）与取消回落 root（:206）不变，
  回复页引用条恒显示是既定默认态。
- 两页持有 `GlobalKey<CommentComposerState>`；`onReply` 回调在 `setState` 目标后
  调 `_composerKey.currentState?.focusForReply()`（内部：`_inputState = keyboard`
  + `_focusNode.requestFocus()`，面板开着时由状态机完成面板→键盘切换）。
- State 类改 public `CommentComposerState`；不采用 `replyRequestId` 计数器
  备选（决策 D4）。

### R3. 发送成功后回复目标固定为「清除」

现状：`_sendText` 只清文本（comment_input.dart:211-215），两页 `_replyTarget`
发送后保留，行为未定义（Astra #19 后半句要求明确规则）。

- 采用 Shaft `clearReplyIfUnchanged` 先例：发送成功且目标未变 → 清除。
  根页 `_replyTarget = null`（引用条消失，后续评论回顶层）；回复页
  `_replyTarget = null` → `?? root` 回落默认根评论目标，引用条仍显示。
- 守卫：`_send` 入口快照 `target`，成功后仅当 `_replyTarget` 与快照同一
  （`identical` 或 `id` 相等）才清除——发送期间用户改了目标不覆盖新选择。
- 失败路径不动：`failSend` 后 mutation 留 failed 态、SnackBar 提示、
  草稿与目标全保留（现状已对，测试固定）。
- **已知不修项（记录）**：mutationKey 含 target id（comments_page.dart:47-48、
  :167-168），发送中换目标会让 `sending` 立即变 false 而旧请求仍在飞——
  这是 store key 契约的既有行为，本包不改 key 结构；busy 指示由 composer
  `_busy`（await 窗口）兜底覆盖主要窗口，「目标 A 的发送仍在进行」的额外
  UI 表达不在本包范围。

### R4. 输入面四态互斥 + 返回先关面板

现状：`_CommentComposerPanel? _panel` 三态 + 隐式焦点（comment_input.dart:6、:35）；
开面板 unfocus 收键盘，但输入框重新获焦不关面板——键盘+面板并存（Astra #20 根因）；
无 `PopScope`，面板开着系统返回直接退页；面板高度写死 210/250（:154）与键盘无关。

- `enum CommentComposerInputState { none, keyboard, emoji, stamp }` 为 UI 唯一
  事实源，替换 `_panel`；`_busy`/`sending` 属发送态（§5.3 action）不并入枚举。
- `_focusNode` listener 单向收敛键盘态：获焦 → keyboard（面板态被收敛关闭）；
  失焦且未进面板 → none。点 emoji/stamp 钮：对应面板 ↔ none 切换并
  `unfocus()`；emoji→stamp 同区域直接换内容。
- `PopScope` 仅拦截面板态（`canPop` = 非 emoji/stamp），拦截即回 none 不离页；
  keyboard 态不拦（系统先收键盘，对齐 Shaft `backCallback` 仅 PANEL enable）。
- 两页 `resizeToAvoidBottomInset: false`（先例 search_page.dart:34、
  home_page.dart:160；component-guidelines 明禁依赖默认 resize）；composer 底部
  区域 `bottomExtent = max(viewInsets.bottom, panelVisible ? panelHeight : 0)`；
  列表底部 padding 跟随 `8 + bottomExtent`（search_page.dart:552 先例）。
- 面板高度统一 `max(_cachedKeyboardHeight, ~280dp 回退)`，废弃 210/250；
  `_cachedKeyboardHeight` 只在 `_focusNode.hasFocus` 期间采样 `viewInsets.bottom`
  峰值——规避 OEM IME 把 navBar inset 并入 ime 的污染（Shaft 注释教训）。

### R5. 网格按可用宽度定列数 + 插入后恢复焦点

现状：emoji 固定 10 列（320dp 下 cell≈26dp，低于 44-48dp 最小点击区）、stamp
固定 5 列（comment_input.dart:159）；`_insertEmoji` 关面板但不恢复焦点（:203），
用户须再点输入框（Astra #20 确认缺陷）。

- `LayoutBuilder` 驱动列数：`crossAxisCount = (maxWidth / minExtent).floor()`
  加 clamp——emoji minExtent≈48 clamp 3–10，stamp minExtent≈96 clamp 2–5
  （图片需可辨识）；spacing 保持 8。320dp→emoji≈6 列，840+→10 列封顶。
- 插入表情：token 插入 + 光标推进后**关面板 + `requestFocus()`** → keyboard
  （贴 §4.7「插入后恢复正确焦点」字面，决策 D2）；连发场景靠重开 emoji 钮
  一键可达。
- 连带契约（同一 commit 落地）：`test/comments_replies_test.dart:563-599` 固定
  10/5 断言重写为宽度驱动断言；`.trellis/spec/frontend/state-management.md` 的
  「10 columns / 5 columns」（:302-303）与「10/5 grids」（:338）表述同步修订为
  响应式契约——spec 与代码同 commit，不留被证伪的契约。

### R6. 发送中可见 + 失败保留草稿

现状：发送中仅全按钮 disabled，无进度指示（Astra/§4.7「发送中可见」未满足）；
失败保留草稿已对（comment_input.dart:211-219），须保留。

- 发送按钮在 `_disabled`（`sending || _busy`）期间显示
  `SizedBox(18×18, CircularProgressIndicator(strokeWidth: 2))` + semantics label
  （新 l10n key）；其余按钮 disabled 行为不变。
- `sending`（store pending）与 `_busy`（await 窗口）重叠保留不合并——R3 记录的
  key 提前失效坑由 `_busy` 覆盖主要窗口。
- 失败重试 = 重按发送钮重试同一操作（§5.7「重试是重试同一操作」），不加显式
  重试控件；回复页 `_showMutationError` 补 `CommentPermissionException` →
  `commentPermissionDenied` 分支（:232-235 现状不区分，对齐根页 :97-99）。

### R7. 发送终态触觉 + 屏幕阅读器/键盘可达路径

- 触觉唯一消费点：页面 `_send`/`_sendStamp` await 成功返回后（与清目标同点），
  调 W4 合入的 `lib/app/` 触觉封装「明确震动」档（§5.6 分级；参考 PixEz
  `mediumImpact` 先例）；文本与 stamp 发送同点同档；composer 内不加业务副作用。
  W4 接口名/路径以合入物为准（预期 `lib/app/haptics/`），W4 未合入本包不动工。
  触觉关/不支持时，成功由「文本清空 + 新评论 prepend 上屏 + 目标清除」视觉可辨
  ——触觉是冗余通道（§5.6）。
- emoji/stamp cell：`Semantics(button: true, label: emoji 名 / stamp id)`，
  图片 `excludeFromSemantics`；面板容器 `Semantics(container: true)`；
  stamp cell 点击即发送的语义须可辨。
- 焦点顺序可解释：列表项（avatar→reply/translate/delete/replies pill）→ 输入框
  → emoji 钮 → stamp 钮 → 发送钮 →（面板开时）网格 cell；`FocusTraversalGroup`
  默认顺序即可，不引自定义 policy（YAGNI）。
- 键盘等价：TextField `newline` action 保留；`InkResponse` cell 可被 Tab 聚焦；
  面板态 Escape 关面板为可选桌面增强（`CallbackShortcuts` 于 composer 顶层，
  标注为增强项）。

## Acceptance Criteria

- [ ] 四态转换矩阵 widget 测试：emoji→点输入框→keyboard、面板态 simulated back
      →none 不离页、keyboard 态不拦截、emoji→stamp 直换；互斥回归用例先对旧实现
      跑红留证据（quality-guidelines 回归证明条款）。
- [ ] header 并入后：长根评论下回复列表首帧 ≥1 条回复可见；`FeedTail`/loadMore
      槽位索引不破坏；根评论 onReply/onDelete 原样工作。
- [ ] 点击回复 pill → composer `focusNode.hasFocus` 断言；引用条语义可定位；
      回复页初始引用条 = 根评论名。
- [ ] 发送成功 → 目标清除（根页 null / 回复页回落 root）+ 文本清空；失败 →
      文本与目标保留；await 期间改目标 → 成功后不清新目标。两页各覆盖关键分支。
- [ ] 320/390/600dp 下列数宽度驱动断言（320→emoji≈6、宽屏封顶 10；stamp
      clamp 2–5）；测试重写与 `state-management.md` 修订同 commit。
- [ ] 插入表情后 `focusNode.hasFocus && 面板关`；面板高度 =
      `max(缓存键盘高, 280 回退)` 断言（MediaQuery 注入 insets）。
- [ ] 发送中 spinner + semantics label 断言；emoji/stamp cell button/label
      语义断言；回复页 permission 分支断言。
- [ ] 触觉调用点在发送成功终态且走 W4 封装（grep 无第二 `HapticFeedback` 来源）；
      W4 未合入则本包不提交产品代码。
- [ ] `flutter analyze --no-pub`、聚焦 + 全量 `flutter test`（loopback 噪声按
      spec 判定）、`dart format`、`git diff --check`、`task.py validate` 全绿；
      每 commit 对应 implement.md 一个勾选框；分支 `task/09-22-comment-input-state`。
- [ ] 运行时缺口在 PR body 标「未验证」：键盘↔面板动画切换顺滑度（widget test
      不可复现真实 IME 动画）、TalkBack/Narrator 焦点路径、OEM IME 高度采样、
      真机触觉开/关、1.3x 大字体、横屏与 reduced motion。

## 行为差异（旧 → 新，迁移说明）

| 场景 | 旧行为 | 新行为 |
|---|---|---|
| 键盘弹出（回复页） | 整列压缩，回复列表被根评论+键盘挤小 | 根评论随列表滚出，insets 自管不压缩层级 |
| 面板开时点输入框 | 键盘+面板并存 | 面板关、进 keyboard（对称互斥） |
| 面板开时系统返回 | 直接退页 | 先关面板，再按才退页 |
| 插入表情 | 面板关、焦点丢 | 面板关、回键盘继续输入 |
| 发送成功 | 回复目标保留 | 目标清除（回复页回落根评论） |
| 窄屏/宽屏网格 | 固定 10/5 列，320dp cell 过小 | 宽度驱动列数，保证最小点击区 |
| 发送中 | 仅按钮 disabled | 发送钮内 spinner + 语义 |

以上均属修复明确缺陷/规格化未定义行为；返回键变化遵循 §5.4「系统返回优先
关闭临时层」既定契约，无隐藏模式类肌肉记忆迁移。

## Decisions（规划定案）

- D1 发送成功清除 reply target（回复页回落 root），带未变更守卫——Shaft
  `clearReplyIfUnchanged` 先例；不选「保留」方案。
- D2 插入表情后关面板 + `requestFocus()`（方案 A），不做 PixEz 式面板保持
  连发（方案 B）——§4.7「插入后恢复正确焦点」字面。
- D3 emoji/stamp 为四态枚举两个值，不合并为单 PANEL+tab（贴 §4.7 枚举措辞；
  Shaft 的 ViewPager2 tab 结构不照搬）。
- D4 主动聚焦用 `GlobalKey<CommentComposerState>.focusForReply()`（命令式、
  可测试）；`replyRequestId` 计数器 prop 备选不采用。
- D5 面板高度 = `max(缓存键盘高, ~280dp 回退)`，废弃 210/250 写死值。
- D6 keyboard 态不拦 back，`PopScope` 仅拦面板态（Shaft `backCallback` 同义）。
- D7 mutationKey 含 target id 导致 `sending` 提前 false 为既有契约行为，
  本包不修、`_busy` 兜底（R3 已记录）。
- D8 不建公共 bottom-panel 组件（唯一消费者，不满足 §5.5 ≥3 门槛）；
  不引入 `chat_bottom_container` 依赖。

## Out of scope

- `lib/core/comments/` 数据层与 mutation key 结构；`sending` 提前 false 的
  额外 UI 表达。
- 公共 bottom-panel/键盘管理基件；第三方面板依赖。
- 评论翻译 overlay、删除确认 dialog、举报/屏蔽（W6/W8 范围）。
- 评论列表 `restorationId`/`PageStorageKey`——恢复等级按 §5.1 声明为内存级
  （现状即无持久恢复，本包不升级）。
- 回复页删除根评论成功后 pop 的既有路径保留不动（:262-264）。
