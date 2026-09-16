# 收藏标签体系

## Goal

补齐收藏的标签维度：详情、标签列表、收藏时选标签、管理页。

## Requirements

- `/v2/illust/bookmark/detail`（收藏详情：已用标签/私密）；`/v1/user/bookmark-tags/illust`（标签列表分页）
- 收藏/编辑收藏时的标签选择 UI（含新标签输入、restrict 切换）
- 收藏标签管理/浏览页（按标签过滤自己的收藏 feed）
- 与 BookmarkStore 确认态/revision 协议兼容

## Acceptance Criteria

- [ ] 收藏动作可带标签并回显；按标签过滤收藏列表可用
- [ ] 端点契约与解析测试；pending/fail 可观测

## References

- PixEz：`page/book/tag/book_tag_page.dart`、`store/book_tag_store.dart`、`models/bookmark_detail.dart`

## Dependencies

无。
