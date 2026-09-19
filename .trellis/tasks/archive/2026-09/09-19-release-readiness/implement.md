# implement — release-readiness

一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] 1. chore(release): pubspec `version: X.Y.Z+N` 为唯一版本源——release.yml 读 pubspec 得 versionName/versionCode，去掉两个 inputs；versionCode 不一致即失败（R1）
- [x] 2. feat(updater): 启动延迟一次性自动检查更新——节流间隔内跳过；有更新→Snackbar/入口提示；无更新/失败静默（R2）
- [x] 3. feat(log): runZonedGuarded+FlutterError.onError 崩溃落盘（限容滚动）+About 页 share_plus 导出（R3）
- [x] 4. chore(release): release.yml 补 generate-notes changelog——draft 有非空 notes 则用，否则按 PR 自动生成（R4）
- [x] 5. chore(task): 收尾簿记（勾选、journal、归档；首发冒烟清单留档）
