# 复刻 New 内容流

## Goal

复刻 beta56 New 页的 Following、Everyone、My Pixiv 三类入口及 Illust/Novel 内容选择和状态保持。

## Confirmed Facts

- beta56 NewPage 有 Following、Everyone、My Pixiv 三个 tabs；再次点击当前 tab 展开 type selector。
- 每个分类分别存在 Illust/Novel source，并使用 AutoKeep 保持页面状态。
- Illust 和 Novel 已有/将有共享实体与 route，本任务不能创建伪 Novel card。

## Dependencies

- 08-26-bookmark-state-sync 完成。
- 08-26-novel-reader 完成 Novel entity/route；08-26-user-profile-follow 提供作者 route 与共享 relationship state。

## Requirements

- R1: 保持 Following、Everyone、My Pixiv 顺序、可滚动 tab 样式和再次点击当前 tab 展开/收起内容类型选择器。
- R2: 每个 tab/type 组合使用独立 lazy paging、cursor、scroll、error 和账号隔离状态。
- R3: Illust 内容复用 IllustStore/Card/Bookmark；Novel 内容复用 NovelEntity/preview/reader route。
- R4: Follow/My Pixiv 为空或账号无权限时显示真实 empty/error，不回退 Everyone 数据。
- R5: 切 tab/type 保持已加载状态，refresh 只影响当前组合；账号切换清理旧账号 feed。
- R6: 当前 API 不再支持某 source 时在 research 中确认并明确阻塞/不可用，不以替代 endpoint 改变含义。

## Acceptance Criteria

- [ ] 三 tabs 与 type selector 手势/展开状态符合 beta56。
- [ ] Illust/Novel 各组合加载真实数据、分页、刷新、错误及 scroll 状态独立。
- [ ] 卡片进入共享 Detail/Novel Reader，bookmark/entity 状态跨入口一致。
- [ ] 空 Follow/My Pixiv、取消、错误、账号切换不串用 Everyone 或旧账号数据。
- [ ] analyze、全量 test、debug build、状态 Widget 测试和真实 API 抽验通过。

## Out of Scope

- 新增自定义订阅筛选。
- 改变 tabs 为独立 Navigator。
- 在 API 不支持时伪造 My Pixiv/Follow 数据。

## Risks and Deferred Items

- My Pixiv endpoint 可能已变更或停用；开始实现时必须真实核验，失败保持显式 blocker。

## Source Anchors

- beta56 lib/pages/new/new.dart、controller.dart、everyone/follow/my_pixiv 各 source
- 共享 Illust/Novel stores 与 paging contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
