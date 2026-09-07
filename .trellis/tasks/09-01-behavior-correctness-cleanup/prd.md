# 行为正确性清理：错误边界收敛

## Goal

删除阻塞正常行为的错误边界与无人消费的抽象，让代码表达的行为与产品实际需要的行为一致。
**本 child 不是 architecture v2**：不引入新状态机、不重建模块边界，只做「删」与「接线」。

覆盖审计编号：C1–C8, C12, C14, C21, C22。

审计原文已由用户补回并归档到
`../09-01-func-1-0-hardening/research/audit-source.md`。该原文基线是 `9d1cb1b`，
实现前仍需对当前 HEAD 做最小复核；原文缺口不再是本 child 的开工 gate。

下文 R1–R6 是复核后可直接实施的部分，R7 收录此前缺失的六项原始定义及其边界。
`task.py start` 前必须完成当前 HEAD 复核、settings D5 交付和本文件的 design/implement
review；不得把旧基线行号直接当作当前证据。

## Requirements（已核实部分）

### R1. `credentialRevision` 收敛（C1）— 单独阶段

**现状核实（HEAD `409df51`）**：`credentialRevision` 有 **79 处引用，横跨 19 个文件**：

| 文件 | 引用数 |
|---|---|
| `lib/core/paging/feed_request_context.dart` | 12 |
| `lib/core/auth/account_store.dart` | 12 |
| `lib/core/profile/profile_edit_models.dart` | 11 |
| `lib/core/paging/paged_feed_controller.dart` | 11 |
| `lib/core/download/download_recovery.dart` | 7 |
| `lib/core/mutation/mutation_models.dart` | 5 |
| 其余 13 个文件 | 各 1–3 |

问题：一个本应表达「凭据是否换过」的值，被当成了**全局世界版本**，
横跨 feed / mutation / download / ugoira / widget / profile 六个不相干的域。

约束：

- 收敛方向是**减少它的适用面**，不是造 `GlobalRevision2`（parent R6 明确禁止
  `GlobalWorldRevision` / `IdentityEpochManager` 一类）。
- 每一处保留的引用都要能回答：「这里真的关心凭据换没换吗？」答不上来的就删。
- **必须作为本 child 内部的独立阶段**，且排在所有 UI 类 child 之后（复核文档第三节），
  避免与它们的改动在同一批文件上互相覆盖。

### R2. Download recovery 的 owner 定义（C5）— 依赖 settings D5

**现状核实**：`download_providers.dart:39-55` 的 `downloadManagerProvider` 在构造时
才创建 `PreferencesDownloadRecoveryStore` 与 `submissionContext`。recovery 因此
**依赖该 provider 被实际创建**——没人 watch 它时，未完成的下载不会被恢复。

`download_recovery.dart` 现状：`destination` 是一个**普通 String**，默认
`kDownloadDestination = 'Pictures/PixivFunc'`（line 56-59, 86, 195, 252）。

> **跨 child 硬约束（parent R5.1）**：settings 的 D5 会引入 SAF tree URI 作为下载目标，
> 直接改变 `destination` 的语义。本项必须**消费 D5 定稿后的 destination 模型**，
> 不得先行定义。若本项先落地，两条线会在同一批代码上互相覆盖。

### R3. WidgetCoordinator 的启动时机（C6）

**现状核实**：`widget_coordinator.dart:120-124` 的 provider 在被 watch 时立即构造
`WidgetCoordinator(ref)`，App 侧存在 eager start。

要判断的是：小组件协调器是否需要在**没有任何小组件实例**时也启动。若不需要，
启动条件应收敛到「确实存在小组件实例」。

注意 parent 的最终验收矩阵要求「widget（有实例与无实例）全部正常」——
改动后两种情况都要验。

### R4. Home 的 IndexedStack 全量预建（C7）

**现状核实**：`home_page.dart:217` 是 `IndexedStack(index: index, children: pages)`。
`IndexedStack` 会构建**全部** children，因此每个 tab 的 feed controller 在 App 启动时
就同时开始请求，而不是切到该 tab 时才请求。

产品后果：冷启动同时打多路请求，首屏变慢，且用户从未访问的 tab 也消耗了配额与流量。

约束：

- 保留切 tab 不丢滚动位置的现有体验——这正是当初用 `IndexedStack` 的原因。
  改法必须同时满足「不预建」与「不丢状态」，不能用「反正切回来重新加载」搪塞。
- 这是共享导航基建，改动影响全部 tab，需单独验收。

### R5. 用户深链被静默丢弃（C8）

**现状核实**：`intent_router.dart` 完整解析了 `UserRoute`，覆盖四种 URL 形态：

- `pixivfunc://users/<id>`（line 294）
- `/u/<id>` 与 `/users/<id>`（line 308-310）
- `user.php?id=<id>`（line 321）

而消费侧 `home_page.dart:137` 只有：

```dart
if (route is IllustRoute && mounted) { ... }
```

**`UserRoute` 被解析出来后直接丢弃，用户点击 pixiv 用户链接没有任何反应**——
既不跳转也不报错。这是「解析了但没接线」，不是安全边界。

要求：`UserRoute` 走 `showUserPage`（该函数已存在并被详情页使用）。
`UnknownRoute` 保持现有的「留在当前页」行为。

### R6. `fetchPage()` 抛 UnimplementedError（C12）

**现状核实**：`recommended_feed_controller.dart:27-30`

```dart
@override
Future<({List<int> ids, String? nextCursor})> fetchPage(String? cursor) {
  throw UnimplementedError('use fetchPageForContext');
}
```

基类 `PagedFeedController` 声明了一个子类无法实现的抽象方法，子类只能抛异常拒绝。
这是基类契约设计错误——**运行时抛异常是在补偿一个类型系统本可以表达的约束**。

要求：收敛基类契约，让「只支持 context 版本」的子类不必抛异常。
不为此新建继承层次，改基类签名即可。

## R7. 审计原文补全：C2, C3, C4, C14, C21, C22

以下定义来自 `research/audit-source.md`。每项在实现前都要重新定位当前 HEAD；不得仅凭
旧基线行号推断问题仍存在。

### R7.1 C2：认证拒绝后的 mutation 只重放一次

- 明确的 401 / `invalid_grant` 属于认证拒绝：refresh 成功后，原收藏、关注或评论操作
  只允许重发一次。
- timeout、connection reset 或响应未知属于“服务器是否执行”不确定：mutation 不自动重放。
- 重放必须沿用原 operation 的输入与安全边界，不能以通用重试器覆盖所有 POST/PATCH/DELETE。
- 成功、refresh 失败、重放失败和“结果未知”要有可诊断的不同结果。

### R7.2 C3：Entity Store 按 payload 来源合并

- feed/sparse payload 未提供的 rich 字段不覆盖已有值。
- detail/authoritative payload 返回的空 caption、空 tags、`visible=false` 或减少的
  page count 都是真实服务端状态，可以覆盖旧值。
- 只引入一个明确的 source 参数或两个 merge 入口；不新增 AuthorityGraph 或全局版本层。

### R7.3 C4：recovery owner 不依赖进程内 credentialRevision

- recovery 记录不得用重启后会从 0 开始的内存 `credentialRevision` 判断同账号任务是否 orphan。
- owner 消费 settings D5 定稿后的稳定 `accountId + job/output + destination` 模型；不为此
  持久化全局 credential epoch。
- 进程重启、token refresh、profile metadata update 都不得无故使同账号合法任务失主。

### R7.4 C14：迁移完成后删除 legacy network ladder

- 先 repo-wide 确认 `_runLegacyLadder` 的调用方已迁移或仅剩可删除的测试适配。
- 删除第二套不同语义的 production/compat 入口，不改变现有 preflight ladder、route memory
  和 N-FIXED-1~9 已修行为。
- 如果仍有真实调用方不能迁移，保留明确失败/兼容原因并在 research 记录，不静默删除。

### R7.5 C21：删除账号时清理未发送的远端 history outbox

- `removeAccount` 时仅删除该 account 的未发送 remote history outbox。
- 本地 history 数据不随之全部删除；用户主动清历史仍走既有路径。
- 重新加入相同账号后不得补发旧账号生命周期内遗留的 outbox。

### R7.6 C22：orphan recovery 有明确收口边界

- 只清理由 owner/job 明确证明属于 Func 的 pending MediaStore/Ugoira 输出。
- unmatched 或无法证明归属的行不得被盲删，也不新增 Recovery Center 页面。
- 状态必须有用户可理解的结果（已清理、可显式 retry、或明确失败），不能留下“状态存在但
  没有出口”的静默记录。

## Acceptance Criteria（已核实部分）

- [ ] `credentialRevision` 的引用面显著收缩；每一处保留的引用都能说明它为何关心凭据变更。
- [ ] 未新增任何 parent R6 禁止的全局版本 / epoch / 权威图基础设施。
- [ ] download recovery 的 owner 定义消费的是 settings D5 定稿后的 destination 模型。
- [ ] WidgetCoordinator 在有实例与无实例两种情况下行为都正确。
- [ ] 冷启动不再同时构建全部 tab 的 feed；切 tab 不丢滚动位置。
- [ ] 点击 pixiv 用户链接（四种 URL 形态）都能进入该用户页。
- [ ] 未知深链仍保持「留在当前页」，不导航到未校验的目标。
- [ ] `fetchPage()` 不再抛 `UnimplementedError`。
- [ ] C2 的认证拒绝只重放原 mutation 一次，结果未知的 mutation 不自动重放。
- [ ] C3 的 sparse/feed 与 detail/authoritative merge 语义分离，真实空值/false/减少值可覆盖。
- [ ] C4 的 recovery owner 不依赖进程内 `credentialRevision`，并消费 D5 destination 模型。
- [ ] C14 的调用方迁移完成后 legacy ladder 被删除，N-FIXED-1~9 没有回退。
- [ ] C21 删除账号时清理该账号未发送 remote history outbox，但保留本地 history。
- [ ] C22 只清理可证明归属的 pending output，且用户对恢复结果有明确出口。
- [ ] 删除 guard 时，同步删除固化该错误行为的测试
      （`.trellis/spec/frontend/quality-guidelines.md` 的 Forbidden Patterns）。
- [ ] `flutter analyze` 与 `flutter test` 通过。

## Notes

- 上级需求与跨 child 约束见 `../09-01-func-1-0-hardening/prd.md`。
- 复核文档第三节明确要求 C1 作为独立阶段并排在 UI 类 child 之后。
- 本 child 是**删除**为主的任务。每删一个 guard 之前先回答 parent R6 的问题：
  「它对应的真实协议、信任边界、平台要求或已复现 bug 是什么？」
