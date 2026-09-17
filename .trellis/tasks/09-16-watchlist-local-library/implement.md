# 执行计划：追更与本地小说库

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(watchlist): /v1/watchlist/{manga,novel} repository 与追更状态`——三端点类型化 + WatchlistEntry 模型 + watchlistAdded 进 series store + 单测
- [ ] `feat(watchlist): 追更入口与列表页`——illust/novel series 页 toggle + watchlist 两段列表页 + 更新角标（latestContentId vs 已读游标）+ l10n + widget 测试
- [ ] `feat(localnovel): TXT 导入与本地库`——TextDecoder 三档 + local_novels.db + 文件复制入库 + 列表页 + 删除 + 单测
- [ ] `feat(localnovel): 本地阅读与线上 TXT 导出`——本地文本走 novel_layout 管线 + 阅读游标 + NovelExporter（元信息头+markup 转纯文本）+ saveFile/SAF 落点 + 单测
- [ ] `chore(09-16): watchlist-local-library journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test test/watchlist_test.dart test/local_novel_test.dart test/novel_export_test.dart` + novel/series 相关测试
- `flutter test`（全量）
- `git diff --check`

## 回滚点

每勾独立可 revert。追更与本地库无共享文件；导出是阅读页一个操作入口，摘掉即回。

## 风险

- GB18030 解码需新依赖或平台 channel → 优先纯 Dart `charset` 包；都没有则 UTF-8 lossy + UI 标注，不静默。
- watchlist 端点响应形状以 PixEz 为准，真机可能不同 → repository 层 lenient 解析 + ApiParseError 可见失败。
- 本地文本走线上排版管线：pixiv markup 在纯 TXT 中不存在，管线自然退化为段落流；大文件分页耗时长 → 排版已有预算上限（novel_layout hard limits）。
