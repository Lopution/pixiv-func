# 执行计划：卡片长按快捷操作与稍后再看

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] `feat(watchlater): core 域落库与 store`——watchlater.db 表 + IllustEntity.toJson/fromJson 往返 + WatchLaterStore（账号隔离、幂等 add、时间序）+ 单测（d26a3a8）
- [x] `feat(card): 动作注册表与长按菜单`——CardAction 类型 + illustCardActionsProvider + card_action_sheet（showAppBottomSheet）+ IllustCard onLongPress/Semantics onLongPress 接线（407dc46，与下一条合并提交）
- [x] `feat(card): 收藏/下载/稍后/分享动作接入`——四个 adapter 注册进 registry；屏蔽注册位注释说明由 mute-system 追加（407dc46）
- [x] `feat(watchlater): 列表页与入口`——watchlater_page（IllustFeedGrid 复用）、common branch 路由 + openWatchLater、设置页 tile、四语言 l10n（3c4e2cb）
- [x] `test(card): 手势/分发/暂存覆盖`——长按弹层、动作分发、watchlater 页渲染与移除
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test`（全绿，含 layering_test）
- `git diff --check`

## 回滚点

每勾独立可 revert；watchlater.db 是全新独立文件，回滚即删，无迁移负担；菜单/长按是纯增量接线，回滚只删 onLongPress 参数与 card_actions 目录。
