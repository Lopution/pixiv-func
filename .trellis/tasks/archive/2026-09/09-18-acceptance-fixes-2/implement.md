# 执行计划：验收反馈修复桶（第二轮）

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。
> 用户在真机验收父任务时第二轮发现的问题 + 自检新发现的同类缺陷。

- [x] `fix(shell): 底栏分支切换整页闪白`——`FuncBranchFade` 包住含 chrome 的整页从 opacity 0 淡入，旧分支在 IndexedStack 里原子撤下 → 前几帧只剩 Scaffold 底色。Material/PixEz/Shaft 底栏切换均瞬时；移除分支级淡入（底栏指示器动画已提供反馈）
- [x] `fix(search): 键盘呼出不再挤压搜索页`——Scaffold 默认 `resizeToAvoidBottomInset` 把 body 压进键盘上方残余空间。改 `false` + 联想 ListView 底部 padding=viewInsets，页面几何不动、键盘覆盖、列表可滚出键盘上方
- [x] `fix(profile): 折叠工具栏名字与动作不再冲突`——`_CollapsedProfile` 全屏居中 title 仅留 64px 边距，actions 最多 5 icon≈240px 直接叠字。改三段式 Row：leading / Expanded(居中+省略 title) / actions
- [x] `fix(reversesearch): 预览按解码比例自适应并支持重选/取消`——`AspectRatio(input.width/input.height)` 用文件头原始宽高（JPEG SOF 不读 EXIF），而 Image.file 解码应用 EXIF → 竖拍图装进横盒子出黑边；ready 态无任何换图路径。解码后取真实宽高进 AspectRatio + ConstrainedBox 限高，加重选/清除入口
- [x] `fix(profile): 好P友/小说空态居中并补传译`——两处把 FeedEmpty 当 ListView 普通 item（顶部小块）而 illust tab 用 SliverFillRemaining 居中；且漏 retryLabel 掉英文默认。统一改 SliverFillRemaining + 补 `profileRetry`
- [x] `fix(i18n): feed_states 补全漏传译并移除英文默认`——recommended_home:191、bookmark_tags:92/59、watchlater:29、recommended_illust:115 五处补 retryLabel/errorTitle；FeedEmpty/FeedError/FeedTail 删英文默认（required/assert）防再犯
- [x] `fix(watchlater): 错误页标题不再泄露原始错误串`——`title: error.toString()` 把含混淆类名的错误串当标题展示，改用 i18n 文案、error 走 error 参数
- [x] `fix(apierror): ApiParseError 透出解析失败原因`——`toString` 丢弃 cause，webview 提取器精心写的诊断串（缺标记/缺value/截断）全部不可见，用户截图只剩 `ccb(response parse error)` 无法远程定位；toString 携带 cause（无敏感信息，类注释已声明安全）
- [x] `refactor(home): 新作页功能入口归入设置内容组`——追更/本地小说库 icon 与本页内容无关且三页 chrome 不统一（0/1/2 个 action 图标）；从 new_page appbar 移除，补进 settings 内容组（与历史/稍后再看/屏蔽同组）；排行的小说排行 icon 保留（是排行的另一半，语义合理）
- [x] `chore(09-18): acceptance-fixes-2 journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- 相关测试：profile/settings/search/reverse-image 相关 widget 测试 + `flutter test`（全量）
- `git diff --check`

## 回滚点

每勾独立可 revert。feed_states 默认值移除是纯签名收窄；入口归位只搬 UI 不动路由。

## 风险

- `resizeToAvoidBottomInset: false`：联想列表最后一项可能被键盘遮住——用 viewInsets bottom padding 抵消，需 widget 测试覆盖。
- 删 `FuncBranchFade`：wide 布局（NavigationRail）共用同一组件，同步移除并确认 rail 切换无回退。
- `ApiParseError.toString` 带 cause：FormatException 消息不含凭证（类注释已声明诊断安全），仍检查 cause 拼接不超长。
- profile 空态改 SliverFillRemaining：外层是 PullToRefresh(isNested) + NestedScrollView，sliver 方案与 illust tab 一致，无新问题。
- 入口归位后追更/本地库入口藏深一层——与「稍后再看」同组先例一致，用户若嫌深再议「我的」页功能区方案。
