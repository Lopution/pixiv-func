# 建立安全多账号与启动状态 — Design

## Objective

建立可恢复、可测试且不会泄露秘密的账号域，使冷启动和后续认证请求由真实账号状态驱动。

## Architecture and Boundaries

- core/auth/account.dart 仅保存非秘密 metadata 与认证状态；credential.dart 作为短生命周期内存值，不提供可打印的默认 toString。
- CredentialStore 隔离平台安全存储，AccountMetadataRepository 保存普通列表/current ID，AccountStore 协调两者并由 Riverpod 暴露 hydration 状态。
- 使用版本化 key（按 account ID）与可恢复提交顺序：先安全写 credential，再提交 metadata；删除时先取消当前引用，再清理秘密。
- StartupGate 组合 Settings/AccountStore 的 loading、error、data，不在错误时退化成 hasAccount=false。

## Data Flow

secure credentials + metadata hydration → AccountStore state → StartupGate → Login/Home；account mutation → credential commit → metadata commit → provider notification。

## Compatibility, Security, and Migration

- 首次启用时没有旧账号数据需要自动迁移；beta56 剪贴板迁移在独立任务实现。
- 若 schema 不兼容，保留可诊断版本并要求 re-auth，不解析未知结构。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

