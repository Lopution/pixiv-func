# 搜索筛选会员适配与刷新图片交接修复

## Goal

修复两个用户报告的缺陷：

1. **搜索筛选无效**：筛选面板里发布时间（duration 预设/自定义日期）和人气排序怎么选都返回同样内容。
2. **下拉刷新图片形态怪**：刷新后整屏卡片在播"作品 A 溶解成作品 B"的跨作品交叉淡化。

## Background

### 筛选链路与根因（已定位）

本地链路端到端验证无误：sheet → `replaceSearchResults` 路由参数 → `_searchQuery` 还原 → `searchFeedProvider` family → `toQuery()` → `NextPageParser` 白名单 → 发出。问题在**服务端语义与客户端假设不匹配**：

- `sort=popular_desc`（及 `popular_male/female_desc`）是 Pixiv Premium 专属；非会员发出后被**静默忽略** → 与 `date_desc` 同结果。
- `duration`/`start_date`/`end_date` 属高级筛选权益，非会员同样被忽略。
- 本应用 `Account` 从未解析 `is_premium`，无法区分会员。
- 对照组：PixEz/pxview 非会员人气排序改走 `/v1/search/popular-preview/{illust,novel}` 端点；Pixeval 直接拦截+提示；pansy 非会员隐藏该 tab；Shaft 用借号池（需要自建后端，不采用）。**没有任何客户端发 `duration=within_last_*`**——预设档全部客户端换算成 `start_date`/`end_date`（app API 对 duration/start_date 的 honoring 已不稳定，见 gallery-dl #8844；且 duration 与日期区间语义互斥，我们目前允许同发）。
- Shaft 还有一个细节：默认档 `partial_match_for_tags` 时**不传 `search_target`**，让标题也能命中（issue #906），与官方 app 行为一致。

### 刷新图片根因（已定位）

`IllustFeedGrid` 的 `SliverChildBuilderDelegate` 无 item key，element 按 index 复用。刷新后坑位换成新作品 → 新 `transitionKey` → `_transitionHistory` 无记录 → `previousTransition == null` → `crossfade = true`（fadeIn 500ms + fadeOut 1000ms）。同时 `useOldImageOnUrlChange` 让 OctoImage 保留**旧作品**的帧作底图 → 观感 = 每张卡播"旧作品溶解成新作品"。所有同类客户端（Glide/RN Image/裸 Image/WinUI）在此场景都是旧帧驻留 + 瞬时替换，无跨作品淡化。

## Requirements

- R1 `Account`/`OAuthUserProfile` 解析并持久化 `is_premium`（登录与 token 刷新两条路径同步更新）。
- R2 非会员选 `popular_desc` 时，illust/novel 搜索自动路由到 `popular-preview` 端点（参数透传，去掉 sort）；会员保持 `/v1/search/{illust,novel}` + `sort=popular_desc`。
- R3 `duration` 预设档在 `toQuery` 时客户端换算成 `start_date`/`end_date`（today−N ~ today），不再发送 `duration` 参数。
- R4 筛选 UI 中 duration 与自定义日期互斥：选 chip 清空日期，选日期清空 duration；`start > end` 禁止应用。
- R5 默认档 `partial_match_for_tags` 不再发送 `search_target`（对齐官方召回范围）。
- R6 `PixivImage` 识别"同 element 换 URL"（坑位交接）→ 淡化置零瞬时替换；`transitionKey` 历史继续管 hero/档位交接；仅真冷加载播 crossfade。
- R7 feed 卡片按作品 id 带 key（`ValueKey(entity.id)`），重排时 element 跟随作品移动。

## Acceptance Criteria

- [ ] 非会员账号：`popular_desc` 实际请求打到 `/v1/search/popular-preview/illust`；`within_last_week` 实际发出 `start_date`/`end_date` 而非 `duration`；同一次搜索应用不同筛选后 provider/缓存键不同。
- [ ] 会员账号：`popular_desc` 直发 `/v1/search/illust` 且带 `sort=popular_desc`。
- [ ] `partial_match_for_tags` 请求中无 `search_target` 参数；其余档仍发送。
- [ ] widget 测试：同 element 换 URL 时 fadeIn/fadeOut = 0；冷加载仍播 crossfade。
- [ ] 全量测试 + analyze + `dart format --set-exit-if-changed` 全绿。

## Notes

- 非会员的 `start_date`/`end_date` 是否被服务端 honor 不确定（证据矛盾）；按所有同类客户端的通行做法照常发送。
- UI 不新增"会员专属"锁定提示，保持 sheet 现有形态；popular_desc 对非会员静默降级为 preview（pxview 模式）。
