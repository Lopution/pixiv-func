# 执行计划：下载体系增强

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(download): sink 续传契约与 Range 重试`——`storedBytes`/`ResumableDownloadSink.detach`/`ResumableDownloadSinkFactory.resumeOwned` + `ResumeAnchor` + manager pause/Range/206-200-416 分支 + recovery 保留可续传 pending + 记录 resume 字段 + 单测
- [ ] `feat(download): 平台续传落点`——MediaStore `resumePending` + SAF `rename`/`resume` + `.part` 命名 + 桌面 `.part` append + Kotlin 单测
- [ ] `feat(download): 暂停/恢复与批量组管理`——`pause/pauseGroup/resumeGroup/cancelGroup` + 任务页组卡片聚合进度与操作 + l10n + widget 测试
- [ ] `feat(download): 整作者作品批量入队`——`AuthorWorksEnumerator`（fetchWorks 枚举，ugoira 排除，2000 上限）+ 作者页「下载全部」入口 + 枚举进度与确认对话框 + widget 测试
- [ ] `feat(download): caption 导出与命名变量扩展`——`CaptionExporter` + `beginRaw` + `AppSettings.downloadCaption` + 变量 `{author_id,pages,page1,series,series_order,chapters,w,h,created}` + 单测
- [ ] `chore(09-16): download-v2 journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test test/download_manager_test.dart test/download_sink_test.dart test/download_recovery_test.dart` + 新增测试
- `cd android && ./gradlew :app:testGithubDebugUnitTest`（MediaStore/SAF channel 单测）
- `flutter test`（全量）
- `git diff --check`

## 回滚点

每勾独立可 revert。续传全是能力接口：去掉 resume 分支即回原 abort 语义；SAF `.part` 命名在 begin/finalize 两点，可单独退回。

## 风险

- pending 行尺寸与记录不一致 → 设计内退化为全新下载（非错误路径）。
- SAF rename 在部分 provider 上可能拒绝 → 失败可见，`.part` 名留存。
- 枚举大作者（千级作品）入队耗时长 → 确认对话框 + 2000 上限 + 枚举可取消。
