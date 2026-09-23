# Risks & Decision Points — W3

## 决策点（需在 PRD/design 阶段拍板）

1. **收起态 follow 的呈现形式**：溢出菜单项（文字）vs 工具栏图标。FollowSwitchButton 是 96/116dp 宽按钮，塞不进 48dp toolbar slot；菜单项复用 `followActionsProvider`/`FollowStore` 状态即可，但「已关注」态菜单文案需新 l10n（现有 `followed`='Following' 语义是状态不是动作）。
2. **re-tap 回顶的实现层级**：PrimaryScrollController（零参数改动）vs 显式 ScrollController 注入四个 feed。前者依赖 NestedScrollView 自动安装内层 controller 的行为，需要 widget 测试钉住。
3. **统计导航目标映射**：illust/manga/novel 计数 → work tab+section 切换（`animateTo`+`setState(_workSection)` 两步，注意 Tab 契约：非 TabBar 回调的 animateTo 允许）。/me 无 about tab，统计是其唯一 totals 入口。
4. **`UserEntity` 字段缺口**：无 `totalIllustSeries` 等 series 计数——series 统计项要么省略要么解析新增字段（数据层改动需单独说明）。
5. **本地过滤的范围**：§4.3「增加本地过滤」若指标签 feed 内关键字过滤，只能作用于已加载页（PixEz PR#1155 的 bookmark_search 同样只能过滤已加载内容+持续请求补偿）。PRD 必须写清语义（过滤已加载 vs 持续拉取补偿），否则验收会卡在歧义上。
6. **收藏 sheet 失败后草稿恢复**：当前 confirm 即 pop+fire-and-forget，失败时草稿已弃。若 §5.3 要求「失败保留草稿」，需要延迟 pop 到 mutation 确认或支持重开恢复——`addWithRestrict` 目前不返回 Future 结果给 UI 等结果（bookmarkActionsProvider 调用未被 await 结果检查），改动牵涉 BookmarkStore 交互，建议 PRD 明确「关闭即提交、失败经 snackbar 可见、重开需重填」是否可接受。
7. **枚举三胞胎**：UserRestrict/BookmarkRestrict/FollowRestrict 三个平行枚举的映射点收敛在 routes 门面还是各调用点。合并枚举超出 W3 范围（属 core 重构），只约定映射点。

## 风险

- **Hero scope 冲突**：`profile_illust_feed.dart` L101-104 heroScope 不含 `bookmarkTag`——/me 收藏流与某标签收藏流（BookmarkTagFeedPage 复用同一 widget）若同时挂载同一作品 ID，tag 相同会跨 surface Hero。scope 字符串需加 `bookmarkTag` 维度。
- **feedKey→widget key 重建**：`_ProfileTabBody` 的 `ValueKey(feedKey)`（user_page.dart L338-340）在 restrict/section 变化时整棵重建——改「持续可见 section 选择器」后切换 section 仍重建（现状如此），但新增「范围从标签页回传」类交互时注意不要触发意外重建丢滚动位置。
- **`material_ui` 阴影**：测试要 import `material_ui` 而非 `flutter/material`，snackbar 断言走 `showAppSnackBar` 通道（quality-guidelines L220-224）。
- **IgnorePointer 改动回归面**：header crossfade 行为已被 `user_profile_test.dart` 多组 widget 测试钉住（L358「expanded details crossfade」、L371 inset、L377 背景独立淡出、L456 收起 chrome 位置、L498 标题不与动作重叠、L594 返回键存活、L650）；把 collapsed 渲染阈值改为 isFullyCollapsed 会改变 0.55–1.0 区间的可见性测试预期。
- **Tab 缩字策略**：`ReplicaProfileTabsDelegate` 的 0.55 下限 + FittedBox 双重缩字（L764-813）；门禁要求不无限缩字——0.55 已是有界，但 1.3x 大字号 + 长翻译场景要实测是否可读，可能需要 overflow/两行 fallback。
- **W1 未合入**：`_attemptPop` 契约（无改动直接返回）尚未实现；W3 若先行，保存入口上移后测试需覆盖 dirty/clean 两条路径，且不能假设 W1 已落地——以 rebase 后基线为准。
- **parallel worktree**：AGENTS.md 要求一 agent 一 worktree；W3 与 W2/W4 等并行叶子同改 `l10n` arb 文件是高冲突区（新增 key 都进同一批 arb+lookup.dart），按 owner 合入顺序串行，lookup.dart 的 key 表冲突需手工合并。
- **运行时验证缺口**：NestedScrollView 内层 controller 回顶、sheet 键盘顶起、宽屏 dialog/sheet 宽度、reduced-motion 下 `showAppBottomSheet` noAnimation 路径——均需真机/桌面矩阵，不能只用 widget 测试推断。
