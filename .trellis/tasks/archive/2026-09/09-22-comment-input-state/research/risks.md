# Risks & Decisions — W7 comment-input-state

## 阻塞性风险

1. **W4 触觉 owner 尚未存在**。`lib/` 全库 `HapticFeedback` 0 命中；`09-22-artwork-viewer-series-flow` 仍是 planning（task.json 确认）。W7 必须等 W4 合入后消费其 `lib/app/` 封装；接口名/等级由 W4 定义，本调研无法钉死调用点签名——draft §7 只给消费位置（发送成功终态），不给 API 名。若 W7 先行，会违反「禁止先落地未使用的公共封装/不另建来源」。

2. **`resizeToAvoidBottomInset` 改 false 的连锁面**。两页 Scaffold 从默认 true 改 false 后，feed 底部必须自管 `viewInsets` padding（search_page 先例），否则键盘/面板遮挡列表尾部；同时 `FeedTail` 重试按钮在键盘开时需仍可点——属「同一物理量两处测量」风险区（quality-guidelines 禁忌），实现时 bottomExtent 只能由一个源头（composer 底部区域或列表 padding 计算式）读取 viewInsets，两处读取同一权威值是允许的，但不得各自缓存。

## 设计决策点（实现前需拍板）

| # | 决策 | 建议 | 依据 |
|---|---|---|---|
| D1 | 发送成功后 reply target 清除 or 保留 | **清除**（回复页回落默认=根评论） | Shaft `clearReplyIfUnchanged` 先例；Astra #19 要求明确规则；mutation key 含 target，清除后回到顶层语义干净 |
| D2 | 插入表情后：关面板+回键盘（A）or 面板保持开（B） | **A** | §4.7「插入后恢复正确焦点」字面；PixEz 的 B 也可辩护，若选 B 需在 PRD 写明 |
| D3 | emoji/stamp 是两枚举还是单面板+tab | **两枚举**（四态直译） | §4.7 明写四选一；Shaft 是单 PANEL+ViewPager2 tab，结构更优雅但偏离 Astra 枚举措辞；选 tab 方案需回写设计 |
| D4 | 主动聚焦的接口形态 | GlobalKey<CommentComposerState>.focusForReply() | 最小改动；备选用 `replyRequestId` 计数器 prop 免公开 State |
| D5 | 面板高度来源 | `max(缓存键盘高, ~280dp 回退)` | Shaft/chat_bottom_container 共识；210/250 写死值废弃 |
| D6 | keyboard 态是否拦 back | **不拦** | Shaft 只在 PANEL 时 enable backCallback；拦键盘 back 会吞掉系统收键盘手势 |

## 测试/契约连带

- `test/comments_replies_test.dart:563-599` 硬编码 10/5 列断言，改响应式列数必须同 commit 重写；`state-management.md:302-303` 的「10 columns / 5 columns」是 spec 文本——质量门要求 spec 与代码同 commit 修订，不能留过时契约。
- `mutationKey` 含 reply target id（comments_page.dart:47-48、:167-168）：发送中换目标会让 `sending` 立即变 false（key 变了），但旧请求仍在飞——这是现状固有行为，若要在 UI 上表达「目标 A 的发送仍在进行」需要额外展示，W7 范围外，但实现时要意识到 busy 指示可能中途消失。缓解：composer `_busy`（await 期内）覆盖大部分窗口。
- 回复页删除根评论成功会 pop（:262-264）：与「back 先关面板」叠加时注意 PopScope 嵌套顺序——composer 内 PopScope 只拦截自身路由的 pop，删除 pop 是代码路径不受影响。
- `SmoothWheelScroll`/`NotificationListener`/`PullToRefresh` 三层包裹中插入 header：header 只是 itemBuilder 前置槽位，无契约冲突；但 header 内 `CommentItem` 不能再用列表 ValueKey 冲突（根 id 不会出现在 replies 列表，安全）。

## 环境/验证风险

- 键盘动画在 widget test 中不可复现（`viewInsets` 需 `tester.binding.window` 注入或 `MediaQuery` 包裹模拟）——四态互斥测试只能断言状态枚举与 widget 树，不能断言动画顺滑；运行时矩阵需真机/模拟器覆盖键盘↔面板切换，缺失即标「未验证」。
- OEM IME（Shaft 注释：HyperOS 把 navBar inset 并入 ime）在 Flutter 侧表现为 viewInsets 峰值污染键盘高度缓存——缓存策略应取「focusNode.hasFocus 期间的 viewInsets.bottom」而非无脑 max。
- l10n 新增键需走生成流程（app_*.arb → gen），4 语言（en/zh/ja/ru）同步；俄语长文案需进运行时矩阵。
- `material_ui` 阴影包：widget test 断言 SnackBar 时必须 import `material_ui`（quality-guidelines 已载）。

## 不做清单（防范围蔓延）

- 不建公共 bottom-panel 组件（仅评论一处消费，不满足 ≥3 消费者门槛——design §5.5）。
- 不动评论翻译、删除确认、举报/屏蔽（屏蔽属 W6/管理页范围）。
- 不为列表加 restorationId/PageStorageKey（现状没有；§5.1 恢复等级声明=内存级，可在 PRD 一句话声明）。
