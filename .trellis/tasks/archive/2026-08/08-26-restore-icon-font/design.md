# 恢复原版 iconFont 字体资产 — Design

## Objective

恢复 beta56 的原始字体图标资产及注册，使当前 Home shell 和后续页面使用与原版一致的 glyph，而不是 Material fallback。

## Architecture and Boundaries

- 字体文件作为不可变上游资产放在 assets/icon.ttf；Dart 代码只保存语义命名与 codepoint。
- pubspec.yaml 是 family 注册的唯一事实来源，不增加运行时 fallback 分支。
- 测试同时检查配置与 Widget 中 IconData family，渲染检查负责发现字体二进制/字形错误。

## Data Flow

pubspec font declaration → Flutter asset bundle → AppIcons IconData(fontFamily: iconFont) → Home Icon render。

## Compatibility, Security, and Migration

- 资产来自 AGPL 参考 commit，任务记录 provenance；许可证整体修正留给最终集成任务。
- 回滚只需移除新增资产、注册和本任务测试，不触碰现有 UI shell。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

