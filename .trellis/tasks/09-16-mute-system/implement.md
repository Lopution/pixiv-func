# 执行计划：屏蔽体系升级

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(mute): core 域与服务端同步`——mute_models/mute_repository/mute_predicate/MuteStore（边界、hydrate、legacy blockedTags 迁移推送、pending 回滚）+ 单测
- [ ] `feat(mute): 卡片模糊变体与 feed 过滤`——AppSettings.muteDisplayMode + IllustCard 模糊揭示（revealedMuteIds 会话集）+ filterPageIds hide 模式接入 + widget loader 恒 hide + 浏览设置切换项
- [ ] `feat(mute): 长按动作与管理页`——_MuteWorkAction/_MuteUserAction 注册 + MutedItemsPage(/settings/muted 三段) + 设置 tile 换指向 + 四语言 l10n
- [ ] `test(mute): 端到端覆盖`——sheet dispatch、blur 渲染/揭示、hide 过滤、管理页
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test`（全绿，含 layering_test）
- `git diff --check`

## 回滚点

每勾独立可 revert；muted_works pref 键与 legacy blocked_tags 互不破坏（迁移失败原 pref 仍在）；菜单项为注册表追加，回滚即删两行。
