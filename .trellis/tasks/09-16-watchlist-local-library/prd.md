# 追更与本地小说库

## Goal

PixEz 式系列追更（`/v1/watchlist/manga|novel`）+ Shaft 式本地 TXT 小说库 + 线上小说 TXT 导出，三块共用「系列/小说」域。

## Confirmed Facts

- PixEz 端点已确认：`GET /v1/watchlist/{manga|novel}` 列表；`POST /v1/watchlist/{manga|novel}/add|delete` form 字段 `series_id`。**追更对象是系列**——manga watchlist 追 illust series，novel watchlist 追 novel series。
- `series_models.dart` 已解析 `watchlist_added`/`isConcluded`/`latestContentId`（系列详情载荷携带）；`illust_series_page.dart` 与 novel series 入口在位。
- 小说阅读管线：`NovelWebPayload.text`（pixiv markup：`[newpage]`/`[[rb:]]`/`[pixivimage:]`）→ `novel_layout.dart` 排版分页；本地 TXT 走同一管线即纯文本段落。
- `file_selector` 已用于备份导入与桌面图片选择；`saveFile` 可用于导出。
- Shaft `TextDecoder`：UTF-8 → BOM 检测 → GB18030 兜底（简中老文本）；`TxtExporter`/`NovelExportManager` 提供导出格式参考。
- `watchlater.db` 先例：本地库可复用「一特性一 SQLite」模式。
- `novel-domain-repair` 已归档（依赖满足）。

## Requirements

- `WatchlistRepository`：manga/novel watchlist 的 list/add/delete，走 `pixiv_http_client` 类型化边界；`watchlistAdded` 状态进 series store。
- 追更入口：illust series 页 + novel series 页的「追更/取消追更」操作；系列详情页 `watchlist_added` 状态展示。
- 追更列表页：manga/novel 两段，显示系列名/作者/最新内容时间，**有新内容角标**（`latestContentId` 与本地已读游标比较）。
- 本地 TXT 导入：`file_selector` 选 `.txt` → 解码（UTF-8 → BOM → GB18030 兜底，与 Shaft 一致）→ **复制进应用文档目录 `local_novels/`** → `local_novels.db` 索引（title/author/路径/字数/imported_at）。
- 本地库列表页：书名/大小/导入时间；删除同时删库行与文件。
- 本地阅读复用线上 `novel_layout` 排版管线（本地文本包成等价 payload，不另写渲染器）。
- 线上小说 TXT 导出：阅读页操作 → 元信息头（标题/作者/作者ID/作品ID/链接/标签）+ 正文 → `saveFile`/SAF 落盘。
- 离线 watchlist add/delete：若 `feed-resilience` 的 `ActionQueue` 已落地则入队重放；否则同步失败可见（不静默吞）。

## Acceptance Criteria

- [ ] 系列页可追更/取消；追更列表展示 manga+novel 两段，新内容有角标。
- [ ] 离线追更变更在 ActionQueue 可用时入队重放（无队列时失败可见）。
- [ ] GBK/GB18030 编码 TXT 导入后中文不乱码；本地库列表可打开阅读、排版与线上一致。
- [ ] 导出 TXT 含元信息头 + 正文；重复导出覆盖同文件不产生乱码。
- [ ] 测试覆盖 repository 端点编解码、解码器三档、本地库 CRUD、导出格式。

## Out of Scope

- EPUB/PDF/Markdown 导出（Shaft 有，本 PRD 只要求 TXT）。
- 本地 TXT 的章节切分/目录识别（按整篇导入，分页由排版管线处理）。
- 追更的推送/后台轮询通知（进入列表时刷新即可见）。
- 本地库文件夹批量导入/扫描。
- watchlist 历史快照或已读进度同步。

## Dependencies

`novel-domain-repair`（已归档）。`feed-resilience` 的 `ActionQueue` 为软依赖：先落地则离线追更自动受益；未落地则同步失败。

## 默认决策（待用户确认）

| 项 | 默认 | 备选 |
|---|---|---|
| 本地文件存储 | 复制进应用文档目录 | 引用原路径（原文件删则失效）|
| 导出范围 | 仅 TXT 含元信息头 | 加 EPUB |
| 更新角标 | latestContentId vs 本地已读游标 | 仅显示 latestContentId 时间 |
| 本地阅读进度 | 存本地游标（复用 anchor 机制）| 不存进度 |
