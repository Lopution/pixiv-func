# acceptance-fixes-5: 阅读器chrome/TagChip描边/评论pill/feed入场id化/逐页尺寸/Montserrat

## Goal

验收反馈第五轮 + 自查项 + 两个追加发现（小说字数不一致、多页作品比例跳变）一次清账。

## Requirements

- Q1 阅读器底栏被系统手势条顶起、页脚 tip 被裁：chrome `Material` 铺满屏边，inset 只垫控件；tip 垫 `viewPadding.bottom`
- Q2 novel 详情弹层 TagChip 融入底色（深浅双主题同病）：全局 `divider` hairline 描边
- Q3 资料编辑永久 loading：`listenManual` 保活后 `load()`（widget 侧 autoDispose 竞态）
- Q4 冷启动/刷新入场动画重播：played 按实体 id 记、网格 `findChildIndexCallback` 按 `ValueKey(id)` 反查
- Q5 详情页字体混杂：P1 裸样式收敛语义源 + AppBar 标题尺寸修复；P2 Montserrat 400-700 拉丁字族、CJK 系统回退
- Q6 评论回复/翻译/删除不对齐：统一 `_ActionPill`，图标色换 `contentSecondary`
- 追加 1：novel_page 两处 sheet 接 `showAppBottomSheet`（补 `showDragHandle` 透传）
- 追加 2：spotlight 链接 `TextSpan+TapGestureRecognizer`；login info 图标 ≥40px
- 追加 3：`withWebContent` 保留 API `text_length`，仅 0 时解析兜底——点开前后字数一致
- 追加 4：多页作品 `/ajax/illust/{id}/pages` 异步种子每页真实宽高，不阻塞 Ready、失败降级、单页不发

## Acceptance Criteria

- [x] flutter analyze 0 issues
- [x] flutter test 全量 1151 通过（含新增回归用例）
- [x] tag chip golden 重录（hairline 预期变化）
- [x] git diff --check 干净
- [ ] 真机验收：阅读器底栏/详情 tag/资料编辑/刷新动画/评论行/字数/多页比例

## Notes

- 多页尺寸方案对照 Pixiv-Shaft `seedPageDimensions`（PixEz/Pixeval/pansy/skana 均用首页比例估算、承受同样跳变）
- 字体方案对照：Shaft 内置 Montserrat 静态字重；Pixeval 内置 NotoSansSC+用户可选；Flutter 系均未内置
