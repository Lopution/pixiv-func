# 复刻 New 内容流 — Design

## Objective

复刻 beta56 New 页的 Following、Everyone、My Pixiv 三类入口及 Illust/Novel 内容选择和状态保持。

## Architecture and Boundaries

- NewFeedKey=(scope,type,accountId) 显式建模；controller map 按 key 懒创建 PagedFeedController。
- TypeSelector 是纯 UI state，切换只选择现有 feed controller，不销毁其他组合。
- Illust/Novel repository adapters 共享网络与分页安全，但输出不同 entity IDs。

## Data Flow

tab tap/re-tap → scope/type selection → keyed feed controller → repository → entity store + ID list → preview card → detail/reader。

## Compatibility, Security, and Migration

- 保留 beta56 re-tap 展开行为与 AutoKeep 效果，内部改用 Riverpod/IndexedStack 生命周期。
- API capability 缺失不偷偷合并 scope。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚已独立提交的叶子任务。

