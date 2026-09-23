# External: 键盘/面板互斥与评论输入器对标

## 1. Pixiv-Shaft（本仓库 V3 复刻基准；本地检出于 /root/pixiv-audit/Pixiv-Shaft）

**结论：Shaft 已有该问题的完整参考实现**，位于独立 Gradle 模块 `panel/`：

- `panel/.../PanelState.kt`：三态枚举 `NONE / KEYBOARD / PANEL`——emoji 与 stamp 合并在同一个 PANEL 里（ViewPager2 两页 + tab 切换，`CommentComposerController.kt:212-216`、KAOMOJI_PAGE=0/STAMP_PAGE=1 :281-282），不是两个互斥面板。W7 的「none/keyboard/emoji/stamp 四选一」比 Shaft 粒度更细，可沿用同一骨架把 PANEL 拆成两个值。
- `BottomPanelCoordinator.kt` 关键机制：
  - `savedKeyboardHeight`（:66）在 `WindowInsetsAnimationCompat.Callback.onStart` 里采样 `bounds.upperBound`（:278-281），面板高度 = 键盘高度 - navBar，未采样到回退 270dp（:58、:524-525）。**Flutter 等价物：`MediaQuery.viewInsetsOf(context).bottom` 在键盘动画期间逐帧变化，需缓存历史最大值/末值**。
  - `setupToggleButton`（:348-355）：输入框被点且 state==PANEL → `switchToKeyboard()`（对称互斥的实现点）。
  - `setupTapToDismiss`（:357-401）：内容区 DOWN 时若面板/键盘开着则 trackingTap=true，UP 时 `dismiss()` 并**消费该次点击**——防止「关面板的点击穿透触发被遮住的条目」。GestureArena 细节：以 touchSlop 区分滚动与点按。
  - 返回键：`OnBackPressedCallback` 仅 `state == PANEL` 时 enable（:91-99、:407）——面板开时 back 关面板；**KEYBOARD 态不拦截**（系统自己先收键盘）。Flutter 等价：`PopScope(canPop: _state == none)` 或 `onPopInvokedWithResult` 中先 `dismiss()`。
  - OEM 坑（注释 :209-216、:310-317）：HyperOS 键盘弹起时会把 navigationBars inset 并入 ime 报告——只在键盘不可见时采样 navBar 高度；判键盘可见用 `isVisible(ime)` 而非纯尺寸启发式。
  - `dismiss()`（:188-198）按当前态分发：KEYBOARD→hideKeyboard+clearFocus；PANEL→动画收起。
- `CommentsComposerViewModel.kt`：
  - **发送成功清除回复目标**：`clearReplyIfUnchanged(reply.revision)`（:144-150）在 `sendComment`/`sendStamp` 成功路径调用；revision 守卫——若发送期间用户改了回复目标/草稿则不动（`draftRevision`/`replyRevision`，:88-89、:107-111）。这是「发送成功后 reply target 清除 vs 保留」的现成决策先例：**清除**。
  - `sendMutex.tryLock()` 防重复发送（:84）；stamp 选中即单发、与文字互斥（:118-133 注释「对齐官方 App 抓包行为」）。
  - `snapshotReply`（:77-80）：显式 parentCommentId 优先，退化为 replyToComment.id，再退化 0=顶层。
- `CommentsFragment.kt:150-168`：reply banner 用 Slide transition 显隐 + 关闭按钮 `cancelReply()`；文案 `"回复 @${name}"`。
- `view_comment_composer.xml`：输入栏 = emoji toggle + EditText(maxLines:4) + `ProgressImageButton` 发送（发送中带进度态）；面板高度首帧占位、由 coordinator 换成实测键盘高（XML 注释 :57-58）。
- 发送成功无 SnackBar、无触觉调用（comments/panel 目录 grep `haptic` 0 命中）。

## 2. PixEz Flutter（/root/pixiv-audit/pixez-flutter，参考而非基准）

`lib/page/comment/comment_page.dart`：
- 互斥方式不同：`MediaQuery.of(context).viewInsets.bottom == 0 && _emojiPanelShow` 才渲染面板（:460-462）——**面板与键盘共存但视觉上被键盘遮住**；点 emoji 按钮 `unfocus()` 收键盘后面板自然出现（:383-390）；点输入框弹键盘时 `_emojiPanelShow` 仍为 true，键盘收起后面板复现。这是「可见性按 insets 推导」的简化方案，不如四态互斥明确，但证明了 `viewInsets == 0` 判据可用。
- emoji 面板：`Container(height: 200)` + `GridView.count(crossAxisCount: 5)`（:105-141），插入到 selection 或 append 末尾；**插完不关面板**。
- **无 stamp 发送 UI**（stamp 只渲染 `comment.stamp`，:283-292）；无回复页内嵌根评论——回复页是同一 `CommentPage(isReplay:true, pId:...)`，目标名在 TextField label（`reply_to ${name ?? "illust"}`，:415），书本图标重置回顶层（:371-380）。
- 发送成功 `HapticUtil.medium()`（:446）+ `_editController.clear()` + `_store.fetch()` 重拉——**触觉反馈先例存在**，但其 HapticUtil 带 60ms 节流与用户开关（utils/haptic_util.dart）。失败 catch 只 print + 关 loading toast，草稿保留（不 clear）。
- 顶层 `Listener` 在任何 pointerDown/Move 时 unfocus（:160-169）——激进的全局收键盘手段。

## 3. 通用 Flutter 模式（web）

- flutter/flutter issue #32583：键盘↔面板无缝切换是长期已知难题；公认结构是 `Scaffold(resizeToAvoidBottomInset: false)` + 底部 panel 容器自己消费 insets。
- `chat_bottom_container`（pub.dev，LinXunFeng/GitLqr）：`PanelType{none,keyboard,emoji,tool}` 枚举 + `ChatBottomPanelContainerController` + inputFocusNode 联动；提供 `changeKeyboardPanelHeight` 回调与键盘高度缓存（默认 300 回退）——与本任务四态模型同构，可作为设计词汇参考；**不建议引入依赖**（仓库无此依赖，60 行内可自实现，且需消费本仓库 mutation/store 契约）。
- flutter/flutter issue #121087：Flutter 无「键盘最终高度」API，实践就是 `viewInsets.bottom` 采样缓存；禁用动画/硬键盘/OEM IME 可能不触发完整动画序列——状态终态必须能从普通 build/insets 路径收敛（Shaft 注释里同样的教训）。
- SO 共识：面板高度 = 缓存的键盘高度（首启回退 ~270-300dp），`resizeToAvoidBottomInset: false`，keyboard→panel 切换时先把面板以当前键盘高摆上屏再 unfocus，避免跟随键盘下落。

## 4. 对 W7 的可迁移结论

1. 状态机直接对齐 Shaft `PanelState`，扩为 `{none, keyboard, emoji, stamp}`（emoji/stamp 是否合并为一个 PANEL+tab 是 W7 决策点——Astra #20 要求四选一，Shaft 结构天然满足「同一时刻一种输入面」）。
2. 互斥三原则（Shaft + Astra 一致）：面板开→unfocus；输入框获焦→关面板；系统 back→先关面板再退页（keyboard 态不拦，框架自己处理）。
3. 面板高度：缓存 `viewInsets.bottom` 峰值，回退常量；`resizeToAvoidBottomInset:false` + 手动底 padding（本仓库 search_page 已是该模式）。
4. 回复目标：发送成功后清除（Shaft 先例），带 revision 守卫防覆盖发送期间的新选择；本仓库可先做简单版（成功且 target 未变 → 清）。
5. 发送中：`ProgressImageButton` 式按钮内进度 + disabled（§5.3 action busy）。
6. 触觉：发送成功用「明确震动」级（§5.6）；PixEz 用 mediumImpact 先例；本仓库必须等 W4 的 `lib/app/` 单一封装。
