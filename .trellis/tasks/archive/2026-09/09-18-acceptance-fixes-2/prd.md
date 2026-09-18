# 验收反馈修复第二轮：底栏闪白/键盘挤压/工具栏冲突/搜图预览/空态i18n/入口归位

## Goal

用户真机验收父任务第二轮报出的 7 个问题 + 全量自检新发现的同类缺陷，按「验收修复桶」规则一 issue 一勾一 commit。

## Requirements

- 底栏五 tab 切换不得有整页淡入/闪白（Material/PixEz/Shaft 均为瞬时切换，底栏指示器动画已提供反馈）
- 搜索页呼出键盘时页面几何不被压缩；联想列表仍可滚到键盘上方
- 个人页折叠工具栏在任何 action 数量/用户名长度下不发生文字与按钮交叠（含非本人页）
- 反向搜图选中图片后：预览按解码后真实比例自适应（EXIF 方向已校正），支持重选与取消
- 个人页各 tab 空态位置统一（与作品 tab 的 SliverFillRemaining 居中一致），且全仓 FeedEmpty/FeedError/FeedTail 不再漏英文默认 label——默认值改为编译期/断言期强制
- `ApiParseError` 的 toString 透出 cause（webview 提取诊断不再被吞，为小说 parse error 定位铺路）
- watchlater 错误页标题使用 i18n 文案，原始错误走 error 参数
- 新作页 appbar 的追更/本地小说入口移入设置「内容」组；排行的小说排行入口保留
- 不回归：reduce-motion 降级、NestedScrollView 嵌套滚动、宽屏 NavigationRail 切换

## Acceptance Criteria

- [ ] 每勾：`flutter analyze` 0 issue + 相关测试通过 + `dart format` 干净
- [ ] 收尾：全量 `flutter test` 0 失败 + `git diff --check` 干净 + journal
- [ ] 代码层面不再存在 `FuncBranchFade`/`branchFade` token
- [ ] feed_states 三控件不存在硬编码英文 label 默认

## Notes

- 自检报告原始记录在会话中；漏标 7 处清单见 implement.md 各勾说明。
