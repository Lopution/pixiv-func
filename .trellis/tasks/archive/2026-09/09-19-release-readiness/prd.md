# release-readiness: 版本源统一/启动自动检查/崩溃日志/changelog

## Goal

对标参考项目（pansy/Pixeval/pixes/skana/Shaft）补齐距首次正式发布的短板。更新基建（签名 manifest + 应用内安装）已超参考对象，不动。

## Requirements

- R1 版本源统一（pansy 惯例）：pubspec `version: X.Y.Z+N` 为唯一真源——release.yml 从 pubspec 读 versionName/versionCode，不再手填 inputs
- R2 启动自动检查（skana/pixes 惯例）：启动延迟一次性 check + 节流（间隔内跳过）；有更新→提示入口；失败/无更新静默不打扰
- R3 崩溃日志（pixes/Shaft 惯例）：`runZonedGuarded`+`FlutterError.onError` 落本地文件（限容滚动），About 页经 share_plus 导出；不引远程遥测 SDK
- R4 changelog（Pixeval 惯例）：release.yml 补 notes 步骤——draft 已有非空 notes 则用，否则 `generate-notes` API 按 PR 自动生成
- R5 首发冒烟清单写入任务文档（人工项）

## Acceptance Criteria

- [ ] flutter analyze 0 issues；flutter test 全量过；git diff --check 干净
- [ ] pubspec 带 +N 后 release.yml 无需 version/versionCode inputs 即可构建
- [ ] 自动检查在无 release 时静默、有更新时给出可见提示、节流生效（测试覆盖）
- [ ] 崩溃写入本地文件、About 可导出（测试覆盖捕获/截断逻辑）
- [ ] release.yml notes 步骤优先用既有 draft body

## Notes

- pansy `UPDATE_OWNER/REPO` dart-define 与"忽略此版本"属锦上添花，本轮不做
- 不引入 Sentry/Crashlytics/Firebase——第三方客户端品类惯例+隐私立场
