# 复刻 Search 与结果页 — Design

## Objective

复刻 beta56 Search 首页、输入页和 Illust/Manga、Novel、User 三类结果，并用可取消 debounce 和共享实体状态实现稳定搜索。

## Architecture and Boundaries

- SearchQuery 是 typed union（illust/novel/user + ID/keyword + filters）；SearchRepository 按类型映射 endpoint。
- DebouncedSearchController 持 generation ID、Timer 和 CancelToken，只有当前 generation 可提交结果。
- Trending/Result controllers 复用 PagedFeedState；实体进入 Illust/Novel/User store，页面只持 ID。
- SearchRouter 统一 detail tag、Home fake box 和 numeric ID 路由。

## Data Flow

Search guide → input query/debounce → typed query → repository/page → entity stores + result IDs → detail/user/reader；tag actions进入同一 router。

## Compatibility, Security, and Migration

- beta56 UI/gesture冻结；内部取消旧 Future-delay 模式。
- Reverse image button 使用已注册的独立 feature route；在 `reverse-image-search` 完成前，开发构建必须以明确的 unavailable 状态呈现，Replica v1 最终验收前不得保留空操作或伪成功。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚已独立提交的叶子任务。
