# 执行计划：行为正确性清理与错误边界收敛

## 开工条件

- [ ] 已阅读 `../09-01-func-1-0-hardening/research/audit-source.md`，并对当前 HEAD 的
      C2/C3/C4/C14/C21/C22 完成最小复核。
- [ ] `settings-productization` 已交付 D5 destination value object 与迁移测试。
- [ ] settings、reverse-image、comment-translation 等 UI child 的改动已完成或明确不再
      触及本 child 将修改的文件；C1 不得提前开始。
- [ ] `design.md` 中的 owner、replay、merge 和 lazy-state 合同获得 review；未确认的产品
      行为不以实现方推测替代。

## 阶段 0：当前 HEAD 证据与测试基线

- [x] 逐项复核完成，证据见 `research/current-head-recheck.md`（含 C1 引用面统计）。
- [x] D5 destination 类型边界（DownloadDestination.identity）、widget bridge
      （hasAnyWidget）、history outbox（clearOutbox）接口变更点均已落实。

## 阶段 1：低耦合行为修复

- [x] C2：post 默认不放行；bookmark/follow/comment 显式 opt-in 单次 replay（maxRetries=1）；
      timeout/reset 不重放（transport canReplay 仅空 GET/HEAD）。测试见
      pixiv_http_client_test / mutation_ownership_test / bookmark / comment。
- [x] C3：EntityMergeSource.feed|detail；detail 允许空值/false/减少 pageCount 覆盖；
      bookmark 仍走 BookmarkStore authority。
- [x] C8：`UserRoute -> showUserPage`；四种形态与 UnknownRoute 回归通过。
- [x] C12：fetchPageForContext 成为唯一抽象；Recommended production throw 删除；未新增
      继承层次；feed_generation_commit_test fake 同步迁移。
- [x] C14：runLadder probe 必填，`_runLegacyLadder` 删除；N-FIXED-1~9 行为保留
      （restricted_compat_network / network_probe 46 项通过）。
- [x] C21：removeAccount 时 clearOutbox(accountId)，本地 history 保留。
- [x] 阶段门：相关单测与 `flutter analyze` 通过（全量 570+）。

## 阶段 2：启动与导航生命周期

- [x] C6：Kotlin hasAnyWidget + Dart WidgetInstanceGate；无实例不订阅/不发请求；
      lifecycle resume 重查；有实例保持现有更新/清理契约。
- [x] C7：_visitedTabs/_tabChildren 懒建+保活；冷启动只建当前 tab；切 tab 不丢滚动与
      controller 状态（home_page_test 29 项通过）。
- [x] 阶段门：Home/widget widget test 通过。

## 阶段 3：D5 依赖的 recovery 收口

- [x] C4：owner = accountId + DownloadDestination.identity（D5 模型）；credentialRevision
      从 context/snapshot/manager/ugoira 删除；旧 'Pictures/PixivFunc' 记录迁移到 builtin。
- [x] C5：app.dart initState 读取 downloadManagerProvider（fireImmediately 触发一次
      轻量扫描）；只恢复/清理明确归属输出，不自动重发。
- [x] C22：pending 行按 owner 精确匹配；unmatched 不盲删；清理失败/orphaned 可诊断可见。
- [x] 阶段门：download_recovery/ugoira_recovery/ugoira/download_manager 测试通过
      （token refresh 语义变更由 C1 域化覆盖）。

## 阶段 4：C1 credentialRevision 独立收敛（最后）

- [x] lib 引用面 65 → 约 16 处（account_store 定义 + widget 域）；按域分类见 recheck。
- [x] feed 改用域内 generation+cancel；mutation 改用 accountId+operation identity；
      profile 用稳定 accountId；store 用 account id；下载/ugoira 用稳定 owner。
- [x] 保留引用（widget）注释说明真实关切（账号显示状态重键信号）；未创建全局 epoch。
- [x] 阶段门：feed/mutation/profile/widget 相关测试全绿。

## 最终验证

- [ ] `flutter analyze`
- [ ] `flutter test`
- [ ] `git diff --check`
- [ ] 检查 `flutter test` 输出中不存在因删除 guard 而被静默跳过的测试。
- [ ] 用户真机：API 29、高版本、无/有 widget、kill/restart recovery、收藏/关注/评论 token
      refresh、用户深链和多 tab 滚动状态。

## 回滚点

1. C2/C3/C8/C12/C14/C21 低耦合行为；2. C6/C7 生命周期；3. C4/C5/C22 recovery；
4. C1 引用收敛。任何误重放、误删、跨账号污染或状态丢失只回滚当前阶段，不修改 settings 或
其它 child 已交付的契约。
