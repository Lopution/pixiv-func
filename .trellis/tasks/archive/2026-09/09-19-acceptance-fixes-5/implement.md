# implement — acceptance-fixes-5

一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] 1. fix(profile): 资料编辑 listenManual 保活防 autoDispose 竞态——initialize 先订阅后 load；补延迟 session 回归测试（Q3）
- [x] 2. fix(novel): 阅读器 chrome SafeArea 反包（Material 铺满屏边/inset 垫控件）+页脚 tip 垫 viewPadding；两处 sheet 接 showAppBottomSheet（Q1/S1）
- [x] 3. fix(novel): withWebContent 保留 API text_length，仅 0 时用解析长度兜底——点开前后字数一致
- [x] 4. fix(ui): TagChip 全局 divider hairline 描边，sheet 场景不再融入；golden 重录（Q2/S3）
- [x] 5. fix(comments): 操作行收敛统一 _ActionPill（surface 底+hairline+icon+label），图标色 dividerColor→contentSecondary，delete 用 danger（Q6/S2）
- [x] 6. fix(feed): 入场动画 played 改实体 id 语义；网格 itemIds+ValueKey+findChildIndexCallback，刷新头部插入旧卡不重播（Q4/S5）
- [x] 7. fix(illust): 详情 AppBar 标题样式修正（裸 TextStyle→titleLarge 链路）+相关区裸样式收敛语义源（Q5-P1）
- [x] 8. fix(ui): spotlight 链接改 TextSpan+TapGestureRecognizer（Stateful 管理生命周期）；login info 图标 IconButton 40px（S4/S7）
- [x] 9. feat(illust): 多页作品逐页尺寸——/ajax/illust/{id}/pages 异步种子合并 metaPages，不阻塞 Ready、失败降级作品级比例、单页不发
- [x] 10. feat(theme): Montserrat 400-700 拉丁/数字字族（pyftsubset 子集化+OFL 许可），CJK 引擎级系统回退（Q5-P2）
- [x] 11. style(test): dart format 回流（app_api_profile_edit/user_profile）
- [x] 12. chore(spec): spec 补 SafeArea/Material 层级、hairline 分层、widget 侧 autoDispose 竞态、id 入场、web 端点异步种子约定
- [x] 13. chore(task): 收尾簿记（勾选、journal、归档）
