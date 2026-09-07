# 技术设计：行为正确性清理与错误边界收敛

设计基线：当前规划基线为 `409df51`；用户提供的审计原文基线为 `9d1cb1b`，原文来源与
C2/C3/C4/C14/C21/C22 定义见上级 `research/audit-source.md`。本 child 不是 architecture v2，
不新增全局状态机、epoch、journal 或 authority graph，只修复已核实的行为边界并删除无消费者
的兼容入口。

## 1. 开工 gate 与依赖

- 开始实现前，对 C2/C3/C4/C14/C21/C22 在当前 HEAD 重新定位，保存代码与测试证据；旧审计
  行号只作为索引。
- C4/C5/C22 必须消费 `settings-productization` 定稿后的 destination value object，不能
  重新引入 String path 或持久化 `credentialRevision`。
- C1（79 处 `credentialRevision` 引用）是本 child 最后一个独立阶段，排在 settings、
  reverse-image、comment-translation 等 UI 类 child 之后，避免大范围文件冲突。
- 若当前 HEAD 已经修复某项，只补回归验证并从实施列表划掉，不重复重构。

## 2. 逐项行为合同

### 2.1 C2：认证拒绝与 mutation replay

在 HTTP 层区分“明确认证拒绝”（401 或明确 `invalid_grant`）与“结果未知”（timeout、reset、
连接中断）。前者 refresh 成功后只重放原 mutation 一次；后者不得自动重放。Bookmark、follow、
comment 的 repository 只通过显式的单次 replay 入口参与，不把通用 retry 应用于所有写请求。
刷新失败、重放失败和结果未知各自保持可观察错误码，避免用户看到假成功。

### 2.2 C3：按 payload 来源合并 Entity

保留 feed/sparse 的“未提供字段不覆盖”语义，同时为 detail/authoritative payload 提供明确的
覆盖入口。authoritative 返回的空 caption、空 tags、`visible=false`、减少的 page count 都
可以覆盖旧值。优先使用 source 参数或两个 merge 方法；不引入 `EntityAuthorityGraph`、全局
版本或按字段的时间戳系统。

### 2.3 C4/C5/C22：下载与 Ugoira recovery

- owner 由稳定 `accountId + job/output + destination` 组成，destination 直接消费 settings
  D5 的规范化描述；token refresh、profile metadata update、进程重启不得改变同账号任务归属。
- App 启动做一次轻量 recovery bootstrap：恢复记录、清理能明确证明归属的 pending output，
  不自动重发下载；用户通过现有 retry 入口显式重试。
- unmatched 或无法证明属于 Func 的 MediaStore/Ugoira 行保留并记录可见结果，不盲删，不为
  低概率情况新建 Recovery Center。

### 2.4 C6/C7：小组件与 Home lazy 状态

- WidgetCoordinator 先通过 native `hasAnyWidget` gate 确认存在实例；无实例时不创建 feed、
  不发网络请求，有实例时保持现有加载/清理契约。
- Home 首次只创建当前 tab，首次访问其它 tab 时再加入保活的 children 集合；保留已访问 tab
  的滚动位置和 controller 状态，不退回每次切换都重载的假 lazy。

### 2.5 C14：legacy network ladder

先 repo-wide 查找 `_runLegacyLadder` 和所有兼容入口。调用方完成迁移后删除第二套不同语义的
production/compat 路径；保留 preflight ladder、route memory 及 N-FIXED-1~9 的现有行为。若
仍有真实调用方，先写出迁移原因，不直接删除导致静默失败。

### 2.6 C21：history remote outbox

`removeAccount` 只删除该 account 的未发送 remote history outbox；本地 history 保留，用户主动
清理仍走既有路径。重新添加相同账号后不能补发上一个生命周期遗留的 outbox。

### 2.7 C1：credentialRevision 最终收敛

逐处回答“这里是否真的关心凭据是否更换”：feed 改用稳定 account + generation/cancel，mutation
改用 operation identity，download/Ugoira/widget/profile 使用各自稳定 owner。删除不相关引用，
不得以 `GlobalRevision2` 或 `IdentityEpochManager` 替换它。

## 3. 文件责任与测试边界

| 责任 | 主要文件 |
|---|---|
| mutation replay | `lib/core/network/pixiv_http_client.dart`、bookmark/follow/comment repositories |
| Entity merge | `lib/core/illust/illust_store.dart` 及 detail/feed mapper |
| recovery owner/bootstrap | `lib/core/download/download_recovery.dart`、`download_providers.dart`、ugoira recovery |
| widget gate | `lib/core/widget/widget_coordinator.dart`、Android widget bridge |
| Home lazy tabs | `lib/features/home/home_page.dart` |
| deep link/contract | `lib/features/home/home_page.dart`、`paged_feed_controller.dart`、intent consumer |
| legacy ladder/outbox | `lib/core/network/compat/`、history repository/store |
| 回归 | mutation/recovery/entity/home/widget/intent/network 测试；C1 阶段补引用审计 |

测试必须覆盖：401 refresh 后一次成功、refresh/replay 失败、timeout 不重放；sparse 与 authoritative
空值覆盖；重启/refresh 不 orphan；有/无 widget；lazy tab 保持 scroll；四种 UserRoute；outbox
删除边界；legacy ladder 无真实调用方。删除 guard 时同步删除固化错误行为的测试或 forbidden
pattern，不能只删断言。

## 4. 风险与回滚

- 先落低耦合的 C2/C3/C8/C12/C14/C21，再落 D5 依赖的 C4/C5/C22，最后做 C1 大范围引用收敛。
- 每阶段独立提交并运行相关测试；出现 mutation 重复、实体字段倒退、下载误删或 tab 丢状态时
  只回滚当前阶段，不覆盖其它 child。
- 任何无法从当前代码证明的 C2/C3/C14 行为，不以“更安全”假设补 guard；保留明确失败并
  在 research 记录待决策项。
