# 设计：屏蔽体系升级

## 范围裁决（父 design §2 委托 child 裁定项）

- **`local_block_filter` 收敛方向**：R18/AI 开关保留为硬过滤谓词（内容分级，不进揭示语义）；`blockedTags` 本地标签表**迁入** MuteStore 的标签屏蔽集合，legacy pref 一次性迁移后清空。`BlockedTagsPage` 被屏蔽管理页取代。
- **呈现默认**：命中屏蔽的作品在 feed 中渲染「模糊+揭示」变体（默认）；浏览设置新增 `muteDisplayMode`（blur/hide），hide 恢复旧的"直接消失"。
- **覆盖面**：本期只覆盖 illust feed 与 `IllustCard`（novel feed 的 `filterPageIds` 走 illustStore，novel 实体天然放行；novel 行级屏蔽记入 watchlist-local-library 的后续范围）。桌面 widget（widget_feed_loader）无论模式一律按 hide 处理——widget 无揭示交互。

## 架构

### core/mute/

```
mute_models.dart     MuteKind(tag|user|work)、MuteKey、MutedUser、MuteState、MuteHit
mute_repository.dart GET /v1/mute/list、POST /v1/mute/edit（pixes 已验证形状）
mute_store.dart      MuteStore extends Notifier<MuteState>
mute_predicate.dart  muteHitFor(IllustEntity, MuteState) -> MuteHit?（纯函数）
```

- `MuteState`：`tags Set<String>`、`users Map<int,MutedUser>`、`workIds Set<int>`、`pending Set<MuteKey>`、`serverSynced bool`、`legacyTagsPending Set<String>`（尚未推送成功的 legacy 迁移标签）。
- **账号边界**：build 里 `ref.watch(accountStoreProvider.select(usableCurrent?.id))`；写操作前 `readMutationBoundary` 校验——与 BookmarkStore 同一边界语义，但不引入完整 MutationLedger：mute edit 是幂等集合操作且不可取消，按 key 单飞 + 失败回滚足够。
- **本地持久化**：`muted_works_<accountId>`（SharedPreferences JSON int 列表，单作品屏蔽官方不支持服务端）。legacy `blocked_tags` pref 为全局键，迁移时并入当前账号的标签集合。
- **hydrate**：build → 本地 works + legacy tags 先进 state（立即生效）→ 异步 `mute/list` 合并服务端 tags/users → legacy 差集通过一次 `mute/edit(add_tags)` 推送成功后清空 pref；失败保留在 `legacyTagsPending`，下次 build 重试。
- **toggle API**：`toggleTag(String)`/`toggleUser(MutedUser)`/`toggleWork(int)`——乐观更新 state → `mute/edit`（works 只写本地）→ 失败回滚该 key 并抛错（adapter/管理页出 snackbar）。pending 中的 key 重入直接返回。

### feed 过滤

- `PagedFeedController.filterPageIds` 增加第二判定：当 `settings.muteDisplayMode == hide` 时，`muteHitFor(entity, ref.read(muteStoreProvider)) != null` 的 id 被过滤；blur 模式不过滤（卡片自己模糊）。controller 在 `localFilterEnabled` 时 watch `muteStoreProvider` + `muteDisplayMode`，mute 变化按既有 C9 路径使 feed 失效重建。
- `isLocallyBlocked` 保持 R18/AI 语义不变（blockedTags 入参改传 `const {}` 由 mute 域接管标签——迁移期 `blockedTagsProvider` 保留读取直到 MuteStore 完成迁移）。

### UI

- `IllustCard`：build 时 `ref.watch(muteStoreProvider.select(muteHitFor(entity)))`；命中且 blur 模式 → 预览图外包 `ImageFiltered(ImageFilter.blur(sigma 8))` + 中央揭示按钮遮罩。tap 命中卡 = 揭示（写入会话级 `revealedMuteIdsProvider` Set，不改屏蔽态）；已揭示卡 tap = 正常进详情。hide 模式 feed 已过滤，卡片无需判 hide 分支。
- 长按菜单追加两项：`_MuteWorkAction`（屏蔽此作品/取消）、`_MuteUserAction`（屏蔽作者/取消）——注册进 `illustCardActionsProvider`，注册位注释已预留。
- 屏蔽管理页 `MutedItemsPage`(`/settings/muted`,common route 不走 settings 子路由以保持 facade 一致——实际按 settings 子路由注册 `path: 'muted'`，与其它设置页一致）：三段（标签/用户/作品）。标签段含输入框添加；每项 trailing 删除=解除屏蔽；用户/作品项 tap 跳转对应页面。
- 设置页：blockTag tile 标题改为屏蔽管理（l10n `mutedItemsSettings`），指向 `/settings/muted`；browse 设置页加「被屏蔽作品显示方式」 blur/hide 切换。

## l10n 新键

`muteWork`/`unmuteWork`/`muteAuthor`/`unmuteAuthor`/`mutedItemsSettings`/`mutedTagsSection`/`mutedUsersSection`/`mutedWorksSection`/`mutedEmpty`/`muteDisplayMode`/`muteDisplayBlur`/`muteDisplayHide`/`muteFailed`/`revealMuted`/`muteTagInputHint`

## 测试计划

- `mute_store_test.dart`：hydrate 合并、legacy 迁移推送与失败重试、toggle 乐观+回滚、账号切换边界、work 仅本地。
- `mute_filter_test.dart`/`card_action_test` 扩展：hide 模式 feed 过滤、blur 模式卡片渲染模糊与揭示、菜单两项 dispatch、管理页增删。
