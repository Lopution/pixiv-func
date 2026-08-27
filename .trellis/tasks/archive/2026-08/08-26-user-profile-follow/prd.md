# 复刻用户主页与关注关系

## Goal

复刻 beta56 UserPage/MePage 的 header、tabs、分页内容和关注交互，并用共享 UserStore/FollowStore 保证跨页面一致。

## Confirmed Facts

- beta56 UserPage 展示背景、头像、名称、统计、分享、Follow 以及 Work/Bookmarked/Follow/About tabs。
- MePage 的 expanded action 是 Settings；collapsed 时 bookmark/follow相关 tab 显示 public/private selector。
- 父 PRD要求自写 ReplicaProfileHeaderDelegate，标题只有完全折叠时居中出现，不继续 fork extended_sliver。

## Dependencies

- 08-26-bookmark-state-sync 完成。
- 08-26-pixiv-network-token-refresh 完成；本任务定义共享 typed user route，供后续 Search/Comments/Live 复用。

## Requirements

- R1: 定义 UserEntity/UserStore，按账号隔离合并 detail、preview、follow 等字段；同一 user ID 不由页面复制状态。
- R2: 实现 User detail、背景/头像/名称/统计/分享和明确 loading/error/not-found/blocked 状态。
- R3: 自写 ReplicaProfileHeaderDelegate，复刻 expanded/collapsed 视觉和 action；标题仅完全折叠时居中出现。
- R4: User tabs保持 Work、Bookmarked、Follow、About；Me tabs/顺序按 beta56源码，所有 tab/type/restrict分页状态和 scroll位置保持。
- R5: 当前 tab再次点击展开 type selector；Me在相关 collapsed tab显示 public/private selector，expanded显示 Settings action。
- R6: FollowStore按 account+user ID管理 confirmed/pending/error；关注/取消成功后所有卡片/Profile/Live同步，失败恢复。
- R7: 作品/Novel/Bookmark/Following/Fans/MyPixiv分页复用共享实体与paging，不存页面私有对象。
- R8: 账号切换、查看本人、分享URL、404/受限和深链导航均明确处理。

## Acceptance Criteria

- [ ] User/Me header在 expanded、过渡、fully collapsed时布局和actions符合beta56，标题出现条件正确。
- [ ] User和Me各tabs、type/restrict selector、分页/错误/scroll保持通过Widget与真机滚动测试。
- [ ] 同一用户在Search、Comments、Profile、Live中的follow状态同步；pending不伪成功，失败恢复。
- [ ] 作品/Novel卡片进入共享Detail/Reader并保持bookmark状态。
- [ ] 404/受限/账号切换/本人路由/分享URL不崩溃、不串数据。
- [ ] analyze、全量test、debug build和真实Profile/follow API验证通过。

## Out of Scope

- 资料编辑提交。
- 私信/聊天。
- 发布作品。
- 自定义Profile redesign。

## Risks and Deferred Items

- 复杂nested scroll容易产生header抖动和scroll丢失；必须使用自有delegate和多尺寸真机测试。

## Source Anchors

- beta56 lib/pages/user/user.dart、me.dart、controller.dart、work/bookmark/following/fans/my_pixiv/about
- beta56 components/follow_switch_button/*；shared entity/paging stores

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
