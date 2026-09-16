# 搜索过滤补全

## Goal

在现有 sort/duration/日期过滤基础上补齐 Shaft v3 级过滤器面。

## Requirements

- 收藏数范围（bookmark_count 区间）；AI 作品三态（全部/仅AI/排除AI）；宽高比与分辨率过滤；性别向热度排序（popular_male_desc/popular_female_desc）
- 过滤器分组 sheet 参考 Shaft `ui/search/v3/`（DateRangePickerSheet/NumberRangeInputSheet/OtherFilterSheet）
- 非会员不可用的参数先验证服务端行为再决定 UI 呈现（父 design §5）
- 全部新参数进入 SearchQuery wire 契约 + cursor 校验；不回归 09-16 刚落地的 duration/日期互斥

## Acceptance Criteria

- [ ] 新过滤器可组合、可清除、状态进入路由/恢复
- [ ] wire 参数与 cursor allowlist 测试齐全
- [ ] i18n 四语言

## References

- Shaft：`ui/search/`（SearchConfig/SortType/v3 sheets）
- 本仓：`lib/core/search/`、`features/search/search_filter_sheet.dart`

## Dependencies

无。
