# 补齐 Android API 36 平台行为 — Design

## Objective

在 API 36 上建立 Replica 所需的统一 Android 平台契约，使导航、外部 intent、文件与媒体操作安全且可真实验证。

## Architecture and Boundaries

- PlatformIntentRouter 将 Android launch/new-intent 数据转换为纯 Dart typed command；URI/MIME/size 验证在平台边界和 Dart 层各有防线。
- ReplicaBackCoordinator 统一 root double-back 与 nested PopScope，不让各页面维护独立退出 Timer。
- PlatformMediaStore 暴露 begin/write/finalize/abort 流式句柄；FileProvider 只用于受控输出分享。
- AndroidWebKitCapabilities 只报告能力并执行受限代理配置，不由 feature 直接调用 Android API。

## Data Flow

Android intent/newIntent → platform validator → typed Flutter command → router；Flutter back gesture → PopScope/coordinator → pop or root exit；media request → MediaStore pending → stream → finalize/abort。

## Compatibility, Security, and Migration

- Manifest 采用最小权限和 flavor merge；未来 updater/widget 声明不得污染基础 Manifest。
- 旧外部存储路径不迁移；新下载统一进入 MediaStore。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

