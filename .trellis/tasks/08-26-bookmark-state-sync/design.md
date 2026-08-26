# 同步跨页面收藏状态与交互 — Design

## Objective

精确复刻 beta56 收藏操作，并通过 ID-keyed BookmarkStore 消除推荐卡片与详情页状态不同步。

## Architecture and Boundaries

- BookmarkStore 是账号隔离的 canonical mutation store；BookmarkKey 包含 type+ID，BookmarkState 区分 confirmed/pending/error。
- BookmarkRepository 只封装 add/delete API；controller 用 operation ID/revision 串行化同 key mutation并丢弃旧响应。
- UI 组件是纯订阅者，sheet 只产生 Restrict 选择，不保存 bookmark bool。
- IllustStore mapper 将远端 bookmark snapshot 提交给 BookmarkStore merge API，而非直接覆盖。

## Data Flow

UI action → BookmarkStore begin(op, previous) → repository add/delete → success commit confirmed | failure rollback previous → all key subscribers rebuild。

## Compatibility, Security, and Migration

- 接口预留 Novel 类型但本任务只验收 Illust；不得以未测试的通用化扩大范围。
- beta56 视觉与时序保留，内部不沿用 GetX tag controller。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

