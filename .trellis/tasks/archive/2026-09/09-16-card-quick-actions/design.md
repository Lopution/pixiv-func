# 设计：卡片长按快捷操作与稍后再看

## 边界

- 只做 `IllustCard` 长按；NovelRow 等其它卡片不在本期（PRD 验收只列 IllustCard 场景）。
- 菜单项 v1：收藏/取消收藏、下载、稍后再看、分享。屏蔽两项留注册位，由 `mute-system` child 注册（不在本 child import mute）。
- 不打断 `onTapDown` 预加载与 tap→Hero 链路；`PressScale`/`Semantics` 包装层保持。

## 动作注册表（父 design §2 落地）

`lib/app/widgets/card_actions/`：

- `card_action.dart`：`abstract class CardAction { String id; IconData icon; String label(AppLocalizations); bool visibleFor(IllustEntity); Future<void> run(BuildContext, WidgetRef, IllustEntity) }`。
- `illust_card_actions.dart`：`illustCardActionsProvider = Provider<List<CardAction>>` 返回有序实例；每个动作是 app 层的薄 adapter（调 `bookmarkActionsProvider`/`illustDownloadCoordinatorProvider`/`watchLaterStoreProvider`/Clipboard），core 域不感知菜单存在。
- `card_action_sheet.dart`：`showCardActionSheet(context, ref, entity)` → `showAppBottomSheet` + `ListTile(icon,label)` 列表；每项 `Semantics(button: true)`；点选 pop 后执行 `run`。注册位即 provider 列表——mute-system 只需向列表追加实例，不需要改 card/sheet 代码。

## 动作语义

- **收藏/取消收藏**:`bookmarkActionsProvider.toggle(BookmarkKey(BookmarkEntityType.illust, id))`；pending 抑制由 store 内部处理；label 依 store 当前态切换（add/delete）。
- **下载**:`illustDownloadCoordinatorProvider` 全页下载（与详情页 downloadAll 同语义）；任务提交反馈走既有 DownloadEvent/snackbar 路径，菜单只提交不新增反馈面。
- **稍后再看**:`watchLaterStoreProvider.add(entity)`；已在列时 label 显示"从稍后再看移除"并执行 remove。
- **分享**：剪贴板复制 `https://www.pixiv.net/artworks/{id}` + snackbar 确认。v1 不引入 share_plus——跨三平台一致、零新依赖；原生分享面板作为可选升级在 design 留记。

## watch-later 域

- `core/watchlater/watch_later_database.dart`：独立 `watchlater.db`（不塞 history.db——避免跨域耦合，沿用 history_database 的 factory/lazy/close 模式）。表 `watch_later_entries(accountId TEXT, illustId INTEGER, addedAt INTEGER, payload TEXT)`，主键 `(accountId, illustId)`；`payload` 为 `IllustEntity.toJson()` 序列化结果（需给实体补 `toJson`，按 API 字段名输出，`fromJson` 可往返）。
- `core/watchlater/watch_later_store.dart`：Riverpod notifier，账号隔离（accountId 维度过滤），时间倒序；add（幂等，重复覆盖更新时间戳）/remove/clear/list。启动懒加载，纯本地零网络。
- `features/watchlater/watchlater_page.dart`：`IllustFeedGrid`(itemCount+itemBuilder)+ `IllustCard` 复用；行级"移除"操作走菜单或滑动——v1 用卡片长按菜单（同一注册表，watchlater 项在页内变成 remove），空态用 `FeedEmpty`。
- 入口：设置页 tile（与 history 并列）+ `_commonBranchRoutes` 注册 `watchlater` 路由 + `openWatchLater` 推 `${_currentStackRoot}/watchlater`。

## l10n

新增键（zh/en/ja/ru 四语言）:`cardAction*`（各动作 label)、`watchLater*`（页标题/空态/移除）、`linkCopied` 确认文案、`watchLaterSettings` tile。

## 测试

- `test/card_action_sheet_test.dart`：长按弹菜单、项可见性、点选分发到对应 store/provider、屏蔽注册位存在。
- `test/watch_later_store_test.dart`:add/remove/幂等/账号隔离/序列化往返（含 `IllustEntity.toJson`↔`fromJson`)。
- watchlater 页 widget 测试：暂存项渲染、长按移除。
- 不逐帧断言动效；菜单弹出仅断言走 `showAppBottomSheet`（duration 来自 token）。
EOF
