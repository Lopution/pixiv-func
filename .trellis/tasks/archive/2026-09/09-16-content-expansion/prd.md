# 内容扩展：插画系列与 pixivision

## Goal

接入 illust series 与 pixivision spotlight 两个内容域。

## Requirements

- illust series：`/v1/illust/series`（系列详情+作品列表）、`/v1/illust-series/illust`；详情页/用户页入口
- pixivision：`/v1/spotlight/articles` 列表 feed + 文章详情结构化渲染（标题/段落/站内作品卡），避免整页 webview
- 新 feed 复用 PagedFeedController + cursor 校验

## Acceptance Criteria

- [ ] 系列页可浏览并逐作进入详情；详情页可进入所属系列
- [ ] pixivision 文章应用内可读，站内作品链接跳详情
- [ ] 端点/解析/游标测试

## References

- PixEz：`page/series/illust_series_page.dart`、`page/spotlight/`、`page/soup/`（文章渲染思路）
- Shaft：`ui/pivision/`、`ui/series/`

## Dependencies

无。
