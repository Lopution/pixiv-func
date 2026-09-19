# implement — snackbar-unify

一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] 1. fix(snackbar): 分支 messenger 挪进底栏 Scaffold body——floating snackbar 锚定 body 底缘，不再盖底部导航（BranchRootScaffold；宽屏无底栏时 messenger 仍在 body 内兜底）
- [ ] 2. feat(snackbar): showAppSnackBar v2 统一形态——floating+统一 margin+共享 AnimationStyle(medium/fast)+action 参数；app.dart 更新提示、home_page 退出提示收编同一入口
- [ ] 3. test(snackbar): 分支内 snackbar 不盖底栏/统一形态/action/宽屏兜底回归用例
- [ ] 4. chore(task): 收尾簿记（勾选、journal、归档）
