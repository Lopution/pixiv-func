# 执行计划:收藏标签体系

> 一个勾一个 commit,提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] `feat(bookmark): 标签模型与 wire 契约`——BookmarkTagFacet/UserBookmarkTag/BookmarkDetail/UserBookmarkTagPage 模型 + add 携带 `tags[]`(空格拼接) + fetchDetail/fetchUserTags 端点 + NextPageParser 白名单 + 单测
- [x] `feat(bookmark): op/entry 携带标签`——BookmarkOp.tags + beginAdd/addWithRestrict 透传 + BookmarkEntry.tags 确认态(add 写入/delete 清空) + 测试
- [x] `feat(bookmark): 收藏 sheet 标签编辑`——_showBookmarkSheet 扩展标签区(已选 InputChip + 建议 FilterChip + 新标签输入) + 已收藏长按改编辑(prefill detail) + userBookmarkTagSuggestionsProvider/bookmarkDetailProvider + 四语言 l10n + widget 测试
- [x] `feat(bookmark): 标签页与过滤收藏`——UserBookmarkTagsController + BookmarkTagsPage(public/private) + ProfileFeedKey.bookmarkTag + fetchBookmarks/cursor 校验带 tag + BookmarkTagFeedPage + 路由与 profile 入口 + 测试
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`(0 issue)
- `dart format --set-exit-if-changed lib test`
- `flutter test`(全绿,含 layering_test)
- `git diff --check`

## 回滚点

每勾独立可 revert;tags 全为可选参数,回滚即删字段;sheet/页面新增独立文件删除即还原。
