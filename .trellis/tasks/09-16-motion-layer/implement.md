# 执行计划：动效层补齐

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(motion): MotionTokens 扩展 + 减少动态效果设置`——新 token 表、MotionScope、resolve/enabled 双源合并、AppSettings.reduceMotion + 浏览设置开关 + 四语言
- [ ] `feat(motion): 路由转场语义化`——page_transitions.dart（推栈滑入/模态上浮/分支淡入），routes.dart 引用，搜索输入走模态转场
- [ ] `feat(motion): 列表进场 stagger + 卡片按压反馈`——FeedEntranceScope/StaggeredEntrance 挂 feed grid 与小说列表；PressScale 挂 IllustCard/NovelRow
- [ ] `feat(motion): 弹层时长统一 + 动效字面量收口`——showAppBottomSheet + showDialog duration；底栏/滚轮字面量改 token
- [ ] `test(motion): 降级矩阵与进场/按压断言`——token resolve 矩阵、stagger 额度/降级、press scale、settings 序列化
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`（不跑 `.`——vendored cargokit 会被无关重排）
- `flutter test`（全绿，含 layering_test 与新增 motion 测试）
- `git diff --check`

## 回滚点

每勾独立可 revert；StaggeredEntrance/PressScale 是纯包装层，回滚只删包装不改语义。转场变更集中在 routes.dart 的 pageBuilder 引用处。
