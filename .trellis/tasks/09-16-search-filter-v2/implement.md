# 执行计划:搜索过滤补全

> 一个勾一个 commit,提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] `feat(search): 过滤模型与 wire 契约`——SearchFilters 新字段 + SearchSort 男/女向 + SearchAiFilter/Ratio/ContentType 枚举 + toQuery(illust-only 范围与 novel sort 归一) + cacheKey + NextPageParser 白名单 + 单测
- [x] `feat(search): 客户端兜底谓词与会员路由`——_SearchFeedController.filterPageIds 覆写(bookmark 区间 + aiOnly) + _spec popular 三档非会员 preview 路由 + 测试
- [ ] `feat(search): 过滤 sheet 分组`——type 参数分组渲染(排序扩展/AI 三态/收藏数/纵横比/作品类别/分辨率) + 四语言 l10n + sheet 测试
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`(0 issue)
- `dart format --set-exit-if-changed lib test`
- `flutter test`(全绿,含 layering_test)
- `git diff --check`

## 回滚点

每勾独立可 revert;新过滤器全是可选字段,回滚即删字段与枚举;sheet 分组删除即还原。
