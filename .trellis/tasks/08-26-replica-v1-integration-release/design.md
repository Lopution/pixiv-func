# 完成 Replica v1 集成与发布验收 — Design

## Objective

在所有功能叶子任务完成后，对 Replica v1 做跨功能、真机、安全、性能、许可和可发布构建验收，形成诚实的完成结论。

## Architecture and Boundaries

- EvidenceMatrix 以父 AC 为键，记录自动化/模拟器/真机/真实 API 四层证据和日期/版本。
- Integration tests 只使用可控测试 fixture；真实账号验证脚本/步骤不记录凭据或私有响应。
- Build/flavor 配置隔离 GitHub updater 与 F-Droid 权限；release signing 通过环境/本地忽略配置注入。
- 许可和归属文件与资产 provenance 一并审查，任何不确定依赖在发布前解决。

## Data Flow

all archived child evidence → integrated build/test matrix → device/API/security/performance/license audits → resolve failures → parent AC review → final report；remote publication remains separately authorized。

## Compatibility, Security, and Migration

- 集成任务不重新设计功能；发现 feature defect 时回到 owning child 或创建明确修复任务。
- 版本/Flavor migration 提供升级和清理说明，不修改用户数据 schema 而无 migration 测试。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

