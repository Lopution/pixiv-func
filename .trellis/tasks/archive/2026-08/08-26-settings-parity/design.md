# 复刻完整 Settings — Design

## Objective

补齐beta56 Settings信息架构和默认值，用版本化、类型安全的现代设置存储驱动主题、浏览、下载、历史、翻译和账号入口。

## Architecture and Boundaries

- AppSettings是immutable typed aggregate并带schemaVersion；SettingsRepository按字段读写和migration。
- SettingsController/细粒度providers暴露各setting，service通过provider注入而非单例静态getter。
- SecretSettingRef只保存CredentialStore key reference，实际凭据不进入AppSettings。
- Settings route registry只有在目标feature真实注册时启用条目；阶段性实现期间明确任务未完成但最终不得残留空入口。

## Data Flow

Settings UI intent → controller validate/persist await → repository result → immutable state/provider notifications → consuming services；startup migrate/hydrate → app theme/routes。

## Compatibility, Security, and Migration

- 从当前replica.* keys迁移到versioned schema且保留guide/language/theme。
- beta56 JSON只作字段/default参考，不导入旧硬编码AES或固定IP默认。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

