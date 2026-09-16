# 追更与本地小说库

## Goal

watchlist 追更 + 本地 txt 小说导入/阅读 + 线上小说 TXT 导出。

## Requirements

- `/v1/watchlist/manga|novel` 列表 + add/delete；追更入口（详情/系列页）与更新角标
- 本地 txt 导入（file_selector）、本地库列表页、与线上阅读器共用排版
- 线上小说导出 TXT（含元信息头）

## Acceptance Criteria

- [ ] watchlist 增删查、更新可见；离线变更走离线队列（若已落地）
- [ ] 本地 txt 可读（排版复用 novel reader 管线）；导出 TXT 格式正确

## References

- PixEz：`page/novel/new/novel_watch_list*`、`models/novel_watch_list_model.dart`
- Shaft：`ui/novel/local/`、`ui/novel/reader/export/`
- skana_pix：`utils/text_composition/`（本地排版引擎思路）

## Dependencies

依赖 `novel-domain-repair`。
