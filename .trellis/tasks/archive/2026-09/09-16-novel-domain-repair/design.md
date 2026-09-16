# 设计：小说域修复与功能闭环

## 1. 根因（静态定位，实现前先复现确认）

`NovelRepository.fetchDetail` 只调 `/v2/novel/detail`，`NovelEntity.fromJson` 读 `content`/`novel_text` 字段。
该端点不返回正文 → `contentAvailable=false` → `fetchDetail` 抛 `_NovelContentUnavailableException` →
`NovelPage` 落 FeedError → 用户看到"打不开"。

**证据链**（`/root/pixiv-audit/` 源码交叉验证，四家一致）：

| 客户端 | 正文来源 |
|---|---|
| PixEz | `apiClient.webviewNovel` → `GET app-api.pixiv.net/webview/v2/novel?id=` → HTML 内嵌 JSON → `novel.text` |
| Shaft | `API.getNovelText("/webview/v2/novel")` → `WebNovelParser` 从 `Object.defineProperty(window,'pixiv',{value:{…}})` 提取 |
| pixes | `apiGetPlain("/webview/v2/novel?id=…")` → 扫描 `novel:` 花括号配平 → `text` |
| skana_pix | `apiGetPlain("/webview/v2/novel?id=$id")` 同法 |

`/v2/novel/detail` 只提供元数据（text_length 等），正文一律走 webview HTML 内嵌 JSON。
func 的 markup 管线（`NovelMarkupParser`：newpage/chapter/jumpuri/rb/pixivimage/uploadedimage 全套 token + budget + 可取消解析）与 reader 布局是完整的——缺的就是正文输入。

## 2. 修复设计

### 2.1 webview 正文管线（`core/novel/`）

- `NovelRepository.fetchWebText(novelId)` → `pixivHttpClient.get(appApiBase.replace(path:'/webview/v2/novel', queryParameters:{id, viewer_version:'20221031_ai'}))` → HTML 字符串
- `_extractNovelJson(html)`：定位 `Object.defineProperty(window, 'pixiv'` 脚本 → `value: {` 起花括号配平截取 → 去尾逗号（`,(?=\s*[}\]])`)→ `jsonDecode` → 取 `novel` 对象。解析失败抛 `ApiParseError`，不做静默降级。
- webview JSON 结构（Shaft `WebNovel`/PixEz `NovelWebResponse`）：`text`、`seriesNavigation{prev,next}`、`marker`、`images`（上传图 id→URL）、`illusts`（pixivimage id→插画引用）。
- `fetchDetail` 改为 detail + webText 并发：detail 失败→错误传播；webText 失败→`ApiHttpError` 传播（FeedError+retry）；均成功但无 text → 保留 `_NovelContentUnavailableException`（页面 contentAvailable 空态分支兜底）。
- `NovelEntity` 增加 `embeddedImages`（identifier→resolved URL，来源 images+illusts map）与 `seriesPrevId/seriesNextId`（seriesNavigation）；`contentVersion` seed 用正文原文。

### 2.2 内嵌图片渲染

markup 已解析 `pixivimage`/`uploadedimage` token 但 block 层无 image block，reader 不渲染。
本期：解析 `images`/`illusts` map 到 `embeddedImages`，reader 遇到 image token 渲染 `PixivImage` 块（宽度铺满、保持纵横比、可点入 illust 详情 pixivimage 情形）。若渲染链路改动过大，可降级为占位卡 + "查看插图"链接，design 实现时裁定并在 PRD 备注。

### 2.3 功能闭环（本 child 后半段）

| 能力 | 端点 | 归属 |
|---|---|---|
| 小说收藏 add/delete | `POST /v2/novel/bookmark/add`（novel_id, restrict）、`POST /v1/novel/bookmark/delete` | `BookmarkRepository` 扩展；`BookmarkEntityType.novel` 已存在，接 actions |
| 小说评论 | `/v3/novel/comments`、`/v2/novel/comment/replies`、`/v1/novel/comment/add`、`/v1/novel/comment/delete` | `CommentStore` 的 `illustId` 维度泛化为 `(workKind, workId)`；key 前缀 `illust:`/`novel:` |
| 小说排行 | `/v1/novel/ranking` | `core/novel/` feed controller + `features/ranking` 增加 novel tab 或独立入口 |
| novel 热词 | `/v1/trending-tags/novel` | `search_trending_controller` 加 kind 参数 |
| 系列导航 | seriesNavigation prev/next | 详情页 `_NovelSeriesBar` 已有按 seriesId 实现；webview 的 prev/next 作为无 seriesId 时的兜底 |

### 2.4 入口审计

当前小说只能从 搜索结果/个人页 feed 进入。本子任务在 ranking 页加小说 tab 后，小说域有了稳定入口。

## 3. 测试策略

- `test/fixtures/` 新增 webview HTML fixture（自制最小 HTML 含 defineProperty 结构）→ 提取器测试：正常/尾逗号/缺 script/截断
- `fetchDetail` 合并测试：detail+text 双 fake；text 失败→HttpError 传播；无 text→contentAvailable 空态
- Bookmark novel 复用 illust 测试矩阵；Comment 泛化后原有 illust 测试必须全绿 + 新增 novel 维度
- widget 测试：NovelPage 详情态/阅读器首段渲染/系列 bar

## 4. 不做

本地 txt 库、TXT 导出、watchlist → `watchlist-local-library`；AI 过滤等 → 不动。
