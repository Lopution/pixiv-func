# 实现 GitHub 与 F-Droid Updater flavors — Design

## Objective

复刻About中的检查更新体验，同时通过flavor彻底隔离GitHub自更新与F-Droid无安装权限构建。

## Architecture and Boundaries

- UpdateCapability由flavor编译注入；F-Droid绑定NoSelfUpdateService且UI显示商店说明。
- GitHubReleaseService返回typedReleaseInfo并执行严格repo/host/version policy。
- UpdateDownloadCoordinator复用DownloadManager但使用受控APK sink和验证器。
- InstallerAdapter封装FileProvider/permission/Intent并验证package/signature。

## Data Flow

About check → flavor service → typed release compare → user confirm → streamed APK + verify → installer intent → result/cleanup；F-Droid→store-managed message。

## Compatibility, Security, and Migration

- 保留beta56检查/进度/确认体验，权限按现代flavor隔离。
- 签名材料通过外部配置，仓库只保存验证/配置接口。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

