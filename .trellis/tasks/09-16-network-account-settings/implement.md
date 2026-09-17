# 执行计划：网络与账号设置增强

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] `feat(mirror): 镜像源模型与 URL 重写`——`ImageSourceMode` 扩展 {normal,pixivCat,pixivRe,pixivNl,custom} + `customImageSource` 字段 + `ImageMirrorResolver`（i./s.pximg 双映射、custom 全前缀、非法输入原样返回）+ AppSettings 序列化 + selectImageSource 校验放开 + 单测
- [x] `feat(mirror): 图片通道接入与探针`——`PixivImageCache`/image client 前接 resolver 重写、`PixivDestinationRegistry`/`NetworkAccessPolicy` 接受 `extraImageHosts`、`networkAccessPolicyProvider` watch imageSource 即时重建、`network_probe` 增镜像步骤 + 测试
- [x] `feat(mirror): 镜像选择 UI`——browse 页解除单源隐藏门控 + preset 单选 + 自定义输入（校验 + 「测试」连通性按钮）+ 四语言 l10n + widget 测试
- [x] `feat(account): 服务端显示设置读写`——`ServerDisplaySettingsRepository`（ai-show GET/edit + restricted-mode GET/POST，form 字段 `show_ai`/`is_restricted_mode_enabled`）+ account 页 section（拉取 + 乐观 toggle + 失败回滚可见）+ l10n + 测试
- [ ] `feat(backup): 导出导入域与 JSON schema`——`core/backup/`：`pixivfunc.backup.v1` envelope（settings/mutes/history）+ 导出服务（prefs→SAF 文件）+ 导入服务（openFile→校验→merge/overwrite 策略）+ `BackupImportException` + 单测
- [ ] `feat(backup): 导出导入 UI`——设置入口 + 合并/覆盖对话框 + 结果反馈 + l10n + widget 测试
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test`（全绿，含 layering_test）
- `git diff --check`

## 回滚点

每勾独立可 revert。镜像默认 normal 即还原现状；服务端设置 section 删除即还原；backup 域为纯新增。
