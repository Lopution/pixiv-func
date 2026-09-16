# 开源竞品审计落地：功能补齐、基础设施与动效升级

## Goal

将 2026-09-16 开源 Pixiv 客户端源码审计（PixEz、Pixiv-Shaft、Pixeval、pixes、pansy、freepiv、skana_pix，
源码留存于 `/root/pixiv-audit/`）中判定"值得吸收"的条目落地，同时修复用户报告的小说页运行期损坏，
并补齐当前过度集中于 hero 图片转场的动效层。

本任务的功能定位：不追赶 Shaft 的"超级应用"路线（自有后端、商业化），而是把日常高频功能缺口补齐，
继续保持工程质量与正确性优势（架构测试、事务化分页、签名更新）。

## 范围边界

**做**：审计清单第 1–16 项 + 小说页修复 + 动效层补齐。

**不做**（用户 2026-09-16 明确排除，对应清单 17–19）：

- FANBOX 与 pixiv COMIC（付费内容合规 + 大工程）
- 设备端 AI：超分 / RIFE 插帧 / 去背景 / MangaOCR（模型分发成本与包体积）
- iOS / macOS 平台移植（证书与平台适配投入，另行评审）

## 需求集（按子任务分组）

| # | 子任务 | 需求（源自审计条目） | 验收要点 |
|---|---|---|---|
| 1 | `novel-domain-repair` | 定位并修复小说页/阅读器运行期损坏（用户报告当前不可用）；小说收藏 add/delete；小说评论（`/v3/novel/comments` + 回复）；小说排行（`/v1/novel/ranking`）；`/v1/trending-tags/novel` | 小说页可打开、阅读器可翻页阅读；收藏/评论/排行真实走通 API；沿用 CommentStore/PagedFeedController 契约 |
| 2 | `card-quick-actions` | 信息流卡片长按菜单（收藏/下载/稍后再看/屏蔽此作品/屏蔽作者/分享）；watch-later 本地暂存域 + 页面 | 长按动作不破坏现有 tap/Hero 手势；watch-later 纯本地、可增删列表、从卡片与页面双向操作 |
| 3 | `mute-system` | tag/user/单作品三态屏蔽；被屏蔽卡片模糊揭示（点击可临时查看）而非消失；屏蔽管理页；评估并接入官方 `/v1/mute/list|edit` 服务端同步 | 三态屏蔽全链路生效于所有 feed；管理页可增删查；服务端同步失败有本地兜底与可见错误 |
| 4 | `search-filter-v2` | 收藏数范围、AI 三态、宽高比/分辨率、性别向热度排序（参考 Shaft search v3 分组 sheet） | 新过滤器进入 SearchQuery wire 契约并走既有 cursor 校验；与既有 duration/日期互斥规则不回归 |
| 5 | `bookmark-tags` | `/v2/illust/bookmark/detail` 收藏详情、`/v1/user/bookmark-tags/illust` 标签列表、收藏时标签选择、标签管理页 | 收藏动作可携带/编辑标签；标签管理页可增删；与 BookmarkStore revision 协议兼容 |
| 6 | `content-expansion` | illust series 页（`/v1/illust/series` + `/v1/illust-series/illust`）；pixivision 文章列表与富文本详情（`/v1/spotlight/articles`） | 详情页/用户页可进入系列；pixivision 文章在应用内渲染图文并跳转站内作品 |
| 7 | `network-account-settings` | 图片镜像站一键预设（pixiv.cat/re/nl/自定义反代）；服务端 AI 显示设置（`/v1/user/ai-show-settings*`）与 R18 restricted-mode 同步；设置/历史数据导出导入 | 镜像预设接入既有 host override 底层；服务端设置读写双向一致；导出文件可重新导入 |
| 8 | `reverse-search-engines` | 在 SauceNAO 之外接入 TinEye/IQDB/Ascii2D（web 端点） | 引擎可选、结果可跳转；失败引擎不阻塞其余 |
| 9 | `download-v2` | HTTP Range 字节级断点续传；批量下载（整页/整作者作品）；caption 导出；命名模板变量扩展 | 复用 DownloadManager recovery/ownership 契约；中断恢复按字节续传；批量任务可管理 |
| 10 | `feed-resilience` | 上次会话 feed 快照首帧直出（参考 Shaft RoomFeedCacheBackend）；收藏/关注等变更离线持久化队列与重放（参考 actionqueue） | 冷启动可见上次内容且后台静默刷新；离线操作不丢失、上线重放与 revision 协议兼容 |
| 11 | `watchlist-local-library` | watchlist 追更（`/v1/watchlist/manga|novel` + add/delete）；本地 txt 小说导入与阅读；小说导出 TXT | 依赖 novel-domain-repair 先落地；本地库与线上阅读器共用排版能力 |
| 12 | `motion-layer` | 动效层补齐：页面转场、列表项进场、反馈动效、sheet/菜单/长按动效的统一语法，全部走 `app/motion/` token | 新动效有 token 单一来源、可被设置项降级（减少动态效果）；不引入逐帧自绘路径依赖 |
| 13 | `tablet-desktop-layout` | 平板/桌面宽屏适配深化：详情 two-pane、侧栏导航、窗口化体验 | 宽断点下详情双栏可读；不回归手机单列行为与既有手势 |

## 跨子任务验收标准（parent 级）

- [ ] `test/architecture/layering_test.dart` 继续全绿：所有新功能遵守 app/features/core 分层与路由门面契约
- [ ] 新 feed 一律复用 `PagedFeedController` + `FeedRequestContext`/`FeedCommitGate`；新变更一律复用 mutation revision 协议
- [ ] 新本地存储域（watch-later、mute、快照、离线队列、本地小说库）全部按账号边界隔离并有恢复语义
- [ ] 所有新 UI 文案走 gen-l10n ARB 四语言（en/ja/ru/zh）
- [ ] 每个子任务：`flutter analyze` 0 问题 + `dart format` 干净 + 相关测试通过；子任务 PR 合入前 `git diff --check` 干净
- [ ] 小说页修复后给出根因记录（写什么坏、为什么坏、如何防回归）

## 任务地图

建议顺序（可调整，依赖见各子任务 PRD）：

- Wave 1：`novel-domain-repair`（用户报告的损坏优先）→ `motion-layer`（先立动效语法，后续子任务消费）
- Wave 2：`card-quick-actions` → `mute-system`（长按菜单是屏蔽入口）→ `search-filter-v2`
- Wave 3：`bookmark-tags` → `content-expansion` → `network-account-settings` → `reverse-search-engines`
- Wave 4：`download-v2` → `feed-resilience` → `watchlist-local-library`（依赖 1）
- Wave 5：`tablet-desktop-layout`（功能面稳定后做布局深化）

## Notes

- 参考实现源码在 `/root/pixiv-audit/`：Shaft 对应实现目录见各子任务 PRD 的 References 段。
- 父任务无分支、无实现提交；各子任务按 AGENTS.md 各自 `task/09-16-<slug>` 分支 + PR。
- 审计报告本体保留在会话中，如需落盘可另立 research 文档（未请求则不入库）。
