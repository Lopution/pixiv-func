# 实现推荐作品流与可靠分页

## Goal

用真实 Pixiv API 打通登录后的 Recommended Illust 首页，并建立后续 Ranking/Search 共用的实体与分页基础。

## Confirmed Facts

- 当前 Home Recommended tab 仅显示文字占位，没有模型、API、图片或分页状态。
- beta56 的 lib/pages/recommended/illust/source.dart 使用 nextUrl 分页，lib/components/illust_previewer/illust_previewer.dart 定义卡片 badges 与收藏入口。
- 父 PRD 要求 Feed 保存作品 ID，IllustStore 保存实体，分页必须区分 initial/refresh/load-more 及其错误。

## Dependencies

- 08-26-pixiv-network-token-refresh 完成。
- 08-26-restore-icon-font、secure-account-store、oauth-pkce-webview-login 完成，并可从真实账号进入 Home。

## Requirements

- R1: 定义稳定 IllustEntity、作者/图片/tag 等必要 value objects 和 API mapper；解析异常返回可观察 ApiError。
- R2: IllustStore 以作品 ID 为键合并实体，保留较新/更完整字段；Feed state 只保存有序 ID 和分页状态。
- R3: 实现 RecommendedIllustRepository 和 PagedFeedController，区分 initial loading、refresh、load more、initial error、load-more error、empty 和 exhausted。
- R4: next_url 必须通过网络层 allowlist parser；每页和跨页按作品 ID 去重，refresh 不遗留旧 next cursor。
- R5: 复刻 beta56 Illust preview 的信息密度、R18/AI/page count badges、作者/标题与图片比例；收藏显示读取共享状态，修改行为留给 bookmark task。
- R6: 图片请求支持取消、占位、失败重试和有界缓存；离开页面不继续无意义 load-more。
- R7: Home tab 返回时保留 ID 列表、scroll position、分页 cursor 和错误状态；账号切换清除账号隔离数据并重新加载。
- R8: 本任务只实现 Recommended Illust；其他 Recommended 类型不得以假数据或空成功路径冒充完成。

## Acceptance Criteria

- [ ] 真实账号登录后首页加载非 mock Recommended Illust，并可刷新、连续分页及到达 exhausted。
- [ ] 重复 ID 只显示一次；恶意 next URL 被拒绝且页面显示明确错误，不发起未知 host 请求。
- [ ] initial error 与 load-more error UI/重试互不覆盖，刷新失败时既有内容策略明确。
- [ ] 切换 Home tab 后返回，feed 内容、scroll position 和分页状态保持；账号切换不串数据。
- [ ] 卡片的 R18、AI、page count 和图片/作者基本信息与 beta56 源码行为一致。
- [ ] controller/repository/widget 测试、analyze、全量 test、debug build 和真实 API 受控验证通过。

## Out of Scope

- Illust Detail 内容。
- Bookmark mutation。
- Recommended Novel/User/Live。
- Ranking、New、Search。

## Risks and Deferred Items

- 推荐 endpoint/schema 会变化；mapper 必须容忍已知可选字段但不能把结构错误静默转换为空列表。

## Source Anchors

- beta56 lib/pages/recommended/recommended.dart、lib/pages/recommended/illust/source.dart、lib/components/illust_previewer/illust_previewer.dart
- beta56 lib/data_content/data_source_base.dart；当前 lib/features/home/home_page.dart

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
