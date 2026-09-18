# 设计：追更与本地小说库

## 1. 结构

```
lib/core/watchlist/
  watchlist_models.dart       — WatchlistEntry(seriesId/type/title/author/latestContentId/ts)
  watchlist_repository.dart   — GET/POST /v1/watchlist/{manga|novel} 类型化端点
  watchlist_controller.dart   — 两段列表 provider + add/delete mutation
lib/core/localnovel/
  local_novel_database.dart   — local_novels.db（一特性一库）
  local_novel_store.dart      — 索引 CRUD + 文件复制/删除
  text_decoder.dart           — UTF-8 → BOM → GB18030 解码
lib/features/watchlist/
  watchlist_page.dart         — manga/novel 两段列表 + 更新角标
lib/features/novel/local/
  local_library_page.dart     — 本地库列表 + 导入入口
  local_novel_reader_page.dart— 复用 novel_layout 的本地阅读壳
lib/features/novel/           — novel_reader 加「导出 TXT」操作
```

## 2. 追更

- `WatchlistRepository`：list 返回分页的 watchlist 条目（系列元数据 + `latestContentId`）；add/delete 走 `pixiv_http_client.post` form `series_id`。
- 入口接线：illust series 页（`illust_series_page.dart`）与 novel series 页加追更 toggle，状态源 `watchlist_added`（详情载荷已有字段解析）。
- 角标：watchlist 列表行对比 `entry.latestContentId` 与本地已读游标（轻量 prefs/`local_novels.db` 同款小表 `watchlist_read_marks`）；进入列表页时刷新一次，不做后台轮询。
- mutation 复用 `MutationLedger` pending/confirmed 语义；离线失败在 `ActionQueue` 存在时入队（`type='watchlist.manga.add'` 等，handler 回同一 repository 方法）。

## 3. 本地 TXT

- 导入：`openFile(.txt)` → `TextDecoder.decode(bytes)` → 复制到 `appDocs/local_novels/<uuid>.txt` → `local_novels` 表插行（title 取文件名去扩展，author 空可编辑）。
- 解码三档（Shaft 同款）：BOM → UTF-8（`allowMalformed:false` 探测）→ GB18030；Dart 侧 GB18030 走 `charset` 包或平台 channel——**决策点：若引新依赖 `fast_gbk`/`charset`，选 charset（纯 Dart、无原生）**；都失败则以 UTF-8 lossy 解并在 UI 标注可能乱码。
- 阅读：`local_novel_reader_page.dart` 把本地文本包成 `NovelWebPayload(text: decoded)` 等价物喂给 `novel_layout` 的排版/分页管线——不另写 renderer；阅读进度游标存本地表。
- 删除：库行 + 文件同删；文件缺失时行保留但打开报可见错误。

## 4. TXT 导出

- `NovelExporter`（`core/novel/`）：拼元信息头（Shaft `TxtExporter` 范式——标题/作者/作者ID/作品ID/链接/标签/简介）+ 正文 `payload.text`（markup 转纯文本段——`[[rb:]]` 保留汉字去注音、`[pixivimage:]` 转占位行）。
- 落点：桌面 `file_selector.saveFile`；Android SAF `createDocument("text/plain")`——复用下载域的 SAF channel；失败可见。

## 5. 测试

- repository：假 client 断言三端点 URL/method/form body；watchlist JSON 解析。
- `TextDecoder`：UTF-8/BOM/GB18030 三档 fixture；乱码兜底标注。
- `LocalNovelStore`：sqflite_ffi 库 + 临时目录文件操作。
- 导出：元信息头格式 + markup→纯文本转换 fixture。
- watchlist controller：mutation pending/confirmed、角标游标逻辑。

## 6. 不做的事

- 不做 EPUB/PDF；不做 txt 章节识别；不做追更推送；不做本地库批量导入。
