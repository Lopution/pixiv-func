# 当前 HEAD 复核证据（2026-09-03 实施时逐项定位）

基线：`9d1cb1b`（用户审计原文）→ 当前实现 HEAD。旧行号只作索引，所有结论以实施时
代码为准。

## C2 认证拒绝后的 mutation 只重放一次
- `lib/core/network/pixiv_http_client.dart`：`post()` 默认 `allowAuthReplay: false`；
  `_send` 对 GET/HEAD 保留 refresh+重试。仅 bookmark/follow/comment 三个 repository
  显式 `allowAuthReplay: true`（单次重放入口，`maxRetries = 1`）。
- transport 层 `_safeReplayFactory`（network_policy.dart）只对空 GET/HEAD 生成
  可重放工厂 → POST/PATCH/DELETE 因网络失败（timeout/reset）永不重试。
- 测试：`pixiv_http_client_test.dart`（401/400 invalid_grant/plain400/429/多并发单次
  refresh）、`mutation_ownership_test.dart`、bookmark/follow/comment 相关单测。

## C3 Entity Store 按 payload 来源合并
- `illust_store.dart` 新增 `EntityMergeSource.feed|detail`；detail 允许空 caption/tags、
  `visible=false`、减少的 pageCount 覆盖；feed 保持 no-regress。bookmark 字段仍由
  BookmarkStore authority 决定（R2）。
- 唯一 authoritative 调用点：`illust_detail_controller.dart`（source: detail）；
  feed/搜索/新作/排行/关注列表保持 feed。
- `user_store.dart`/`novel_store.dart`：已有 hasDetail/contentAvailable 防御，未新增
  全局版本层。
- 测试：`illust_detail_page_test.dart`（detail fixture 携 caption 后通过）。

## C4 recovery owner 不依赖 credentialRevision
- `DownloadSubmissionContext`/`DownloadSubmissionSnapshot` 删除 `credentialRevision`；
  owner = accountId + `DownloadDestination.identity`（D5 模型的稳定标识）。
- `download_manager._sameContext/_identityKey` 与 `ugoira_export._checkOwner` 改为
  稳定边界；recovery JSON 持久化 identity（含旧 'Pictures/PixivFunc' → builtin 迁移）。
- `_nextCredentialRevision` 仍被 login/refresh/updateAccount/switch/remove 递增，但
  消费方只剩 widget 域（见 C1）。

## C5 App 启动 recovery bootstrap
- `app.dart` initState 读取 `downloadManagerProvider`（fireImmediately 侦听触发
  `recoverMedia()`）；只读 recovery store/MediaStore pending 列表，不自动重发。

## C6 WidgetCoordinator 启动时机
- Kotlin `WidgetForegroundChannel.hasAnyWidget`（AppWidgetManager 已安装 providers 交集）。
- Dart `WidgetInstanceGate` + `MethodChannelWidgetInstanceGate`；`start()` async 检查；
  app lifecycle resume 时 `ensureStarted()` 重查（widget 添加后免重启生效）。

## C7 Home IndexedStack 懒建
- `_visitedTabs` + `_tabChildren`：只构建访问过的 tab，已访问 tab 保活（滚动位置/
  controller 状态保留）；未访问 tab 冷启动不发请求。

## C8 用户深链
- `home_page.dart`：`UserRoute` → `showUserPage`（四种 URL 形态），`UnknownRoute` 保留
  留在当前页。测试见 `home_page_test.dart`（已通过）。

## C12 fetchPage UnimplementedError
- `PagedFeedController.fetchPageForContext` 成为唯一抽象；删除 `fetchPage`/
  `fetchPageCancellable` 双契约；7 个子类与 `feed_generation_commit_test` 的 fake 全部
  内联迁移；`recommended_feed_controller` 不再 throw。

## C14 legacy network ladder
- `runLadder` 的 `probe` 参数必填；删除 `_runLegacyLadder`（约 70 行）与 no-probe 分支；
  唯一生产调用方（PixivPolicyHttpClient、PolicyDownloadTransport）均传 probe。
  `restricted_compat_network_test.dart` / `network_probe_test.dart` 46 项通过。

## C21 账号删除清理 remote outbox
- `HistoryRepository.clearOutbox(accountId)`（只删 outbox 表）；`AccountStore.removeAccount`
  调用；本地 history 保留；`historyRepositoryProvider` 注入见 `history_repository.dart`。

## C22 orphan recovery 收口
- `download_manager._recover`：只按 `record.owner.ownerId` 精确匹配 pending 行；
  unmatched 行不删；成功+未清 pending 的矛盾行只上报；清理失败与 orphaned 结果均进入
  报告与任务状态（用户可见出口）。

## C1 credentialRevision 收敛（最后阶段）
- 引用面：lib 内 65 处 → account_store 定义/版本管理 + widget 域（4 处）≈ 16 处。
  - feed 域：FeedRequestContext/FeedCommitGate 删除 credentialRevision 与
    credentialChanged 原因；域内 generation + accountId 已覆盖 A→B→A 场景。
  - mutation 域：MutationBoundary/Envelope 只保留 accountId（同账号 token refresh/
    profile 更新不使 in-flight 写失效）；`MutationDiscardReason.credentialChanged` 删除。
  - profile 域：ProfileDraft/Patch/Owner 删除 credentialRevision；staleOwner 只由
    accountId 触发。
  - store 域：bookmark/follow/comment 的 `watch(accountStoreProvider.select)` 只选
    account id，refresh/updateAccount 不重建。
  - widget 域保留（注释说明真实关切：widget 展示账号显示状态，revision 是重键信号）。
