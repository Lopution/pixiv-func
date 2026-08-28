# 实现 GitHub 与 F-Droid Updater flavors — Design

## Objective

复刻About中的检查更新体验，同时通过flavor彻底隔离GitHub自更新与F-Droid无安装权限构建。

## Architecture and Boundaries

- UpdateCapability由flavor编译注入；F-Droid绑定NoSelfUpdateService且UI显示商店说明。
- SignedManifestService先以编译期公钥验签、限长和schema校验，再返回typed ReleaseInfo；GitHubReleaseService不能绕过该信任根。
- UpdateDownloadCoordinator复用DownloadManager但使用受控APK sink和验证器。
- InstallerAdapter封装FileProvider/permission/Intent并验证package与APK signing certificate等于当前安装 signer。

## Data Flow

About check → fetch bounded manifest+signature → verify pinned public key → typed release/channel compare → user confirm → single-flight streamed APK + exact size/hash/package/signer verify → installer intent → result/cleanup；F-Droid→store-managed message且无updater graph。

## Compatibility, Security, and Migration

- 保留beta56检查/进度/确认体验，权限按现代flavor隔离。
- manifest私钥与release签名材料只通过外部CI安全输入；App/仓库只保存可公开的验证公钥和schema，缺失时fail closed。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。
