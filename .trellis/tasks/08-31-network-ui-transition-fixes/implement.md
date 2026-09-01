# 实施计划：无代理网络与作品转场

本仓库当前为 Codex inline 工作流：不分派子 Agent，不生成 `implement.jsonl`/`check.jsonl` 条目；`task.py start` 前只完成规划和审阅，进入执行后由主会话按 `trellis-before-dev → 编辑 → trellis-check` 顺序推进。

## Phase 0：开始前门槛

- [x] 用户已确认 U4 采用“立即转场、冷缓存头像在详情内补齐”的可见性边界；仍需在最终规划摘要后取得启动实现的明确批准。
- [x] 运行 `python3 ./.trellis/scripts/task.py validate network-ui-transition-fixes`，确认 `prd.md`、`design.md`、`implement.md` 结构完整。
- [x] 执行 `python3 ./.trellis/scripts/task.py start network-ui-transition-fixes` 后再修改产品代码；开始前重新检查 `git status --short`，只保留已有用户改动。
- [x] 执行 `trellis-before-dev`，读取 frontend/backend 相关 spec index、质量门槛和图片/网络状态约定。

## Phase 1：先钉住可观察行为

- [x] 在 `test/network_probe_test.dart` 增加“DNS 不一致 + ECH 成功”“ECH HTTP 404/403”“空 SNI 421”和“所有路径失败”断言；更新已有 `dnsPolluted` 期望以反映主结论优先级。
- [x] 为 `DohResolver` 增加 ECH TTL、revision 失效和无取消并发合并测试；验证空 config、过期 config、取消请求不会被当成成功。
- [x] 为首图选择器/`heroImageUrl` 增加有/无 `metaPages`、medium/large 质量一致性测试；扩展 `test/illust_detail_page_test.dart` 证明首帧 Hero 目标和作者信息存在。
- [x] 新增 `test/pull_to_refresh_test.dart`（或合并到现有 widget 测试）覆盖正常下拉触发一次、阈值前反向拉回不触发且 indicator 消失。
- [x] 为 scope 纯函数/卡片构造增加不同 tab、排行模式、搜索词、个人筛选之间不冲突的测试；必要时用现有 detail widget 测试锁定飞行 clip 的子树。

## Phase 2：网络实现

- [x] 修改 `lib/core/network/compat/secure_resolver.dart`：加入带 TTL/revision 的 ECH 结果缓存和无取消并发查询合并；复用现有 DoH 超时、响应上限、endpoint 轮换和取消语义。
- [x] 修改 `lib/core/network/compat/network_policy.dart`：在现有按主机 route memory 之上加入按目的地组的短期首选档位；在 probe/业务成功后记录，在 transport failure、revision、mode/settings 变化或 TTL 到期时清除；保持 direct-only、幂等重选次数和目的地白名单契约。
- [x] 修改 `lib/core/network/compat/network_probe.dart` 与 `network_contracts.dart`：让报告携带 DNS 差异附加信息，调整结论优先级；只把 HTTP 421 作为空 SNI 路由不可用，保留各层原始状态。
- [x] 修改 `lib/features/settings/network_probe_page.dart`：badge 显示可行动路径，补充 DNS 差异说明和复制文本字段；不隐藏直连失败事实。
- [x] 运行网络聚焦测试和静态分析；缓存时序由可控 fake/clock 测试覆盖，未添加生产重试或静默兜底。

## Phase 3：首图、头像和 Hero

- [x] 修改 `lib/core/entity/illust_entity.dart`（或同层 helper）提供首图 URL 选择；修改 `lib/app/pixiv_image.dart` 提供与 `CachedNetworkImage` 相同 headers/cache manager 的 provider/preload 辅助。
- [x] 修改 `lib/features/home/recommended/recommended_illust_page.dart` 的 `IllustCard`：在 `onTapDown`（并以 `onTap` 兜底）计算当前预览 URL，启动首图和头像预热但不等待，立即把 `heroImageUrl`、`initialEntity`、scope 传给详情；固定头像容器尺寸，避免重复点击在同一 context 推入多条路由。
- [x] 修改 `lib/features/illust/detail/illust_detail_page.dart`：index 0 使用传入首图 URL/高质量 fallback，其余页保持 `metaPages` 尺寸和 URL；给详情 Hero 添加匹配的 flight clip；头像使用同一 provider/cache key。
- [x] 修改推荐、新作、排行、搜索的 scope 调用点（`recommended_home_page.dart`、`new_page.dart`、`ranking_page.dart`、`search_result_page.dart`），为存活 tab/模式/查询生成稳定唯一 scope；确认 profile 既有 scope 不被削弱。
- [x] 运行 detail/entity/scope widget tests，并检查多图、单图、ugoira、下载模式和无来源深链入口；新增部分可视卡片与嵌套 pinned-header 的 Hero clip 回归测试。

## Phase 4：共享刷新组件

- [x] 新增 frontend 小组件（`lib/app/pull_to_refresh.dart`），以 scroll notifications 记录真实拖动增量，让 indicator 在 armed 后仍可线性反向回到不可见区域；不复制 Flutter framework 实现，不改变真正 refresh 的 Future；反向等量拖动回归测试锁定线性位移。
- [x] 将 8 个文件中的 14 个 `RefreshIndicator` 使用点迁移，保留每个页面原有的 `NotificationListener` load-more 逻辑、controller、`AlwaysScrollableScrollPhysics` 和错误尾部。
- [x] 运行刷新 widget 测试，并用 `rg -n "RefreshIndicator\\(" lib` 确认只剩共享组件内部（除非有明确例外）。

## Phase 5：质量检查与设备回归

- [x] 执行 `flutter analyze --no-pub`（本机 PATH 不含 Flutter，使用 `/opt/flutter-3.47.0/bin/flutter analyze --no-pub`，通过）。
- [x] 执行聚焦测试：`/opt/flutter-3.47.0/bin/flutter test --no-pub test/network_probe_test.dart test/illust_detail_page_test.dart test/illust_entity_pages_test.dart test/pull_to_refresh_test.dart test/hero_transition_test.dart`，41 项通过。
- [x] 执行 `/opt/flutter-3.47.0/bin/flutter test --no-pub --concurrency=1`，522 项通过。
- [x] 执行 `git diff --check`，复查 `git diff --stat` 和所有任务范围文件。
- [x] 执行 `/opt/flutter-3.47.0/bin/flutter build apk --debug --flavor github`，成功生成 `build/app/outputs/flutter-apk/app-github-debug.apk`。
- [ ] 连接真实手机，在无代理环境复测：设置探测四主机、首次未缓存多图、已有/无头像缓存、返回时列表上下边界、反向下拉取消、各存活 tab 的 Hero；分别记录 Implemented / Compiled / Unit-tested / Device-tested，不用模拟器结果冒充真机。

## Phase 6：收尾门槛

- [x] 如发现同类 bug 的新可执行约定，运行 `trellis-update-spec` 写入对应 frontend/network spec；已将稳定 Hero/首图预热/反向刷新契约写入 frontend component spec，将 ECH 缓存/组路由/探测结论写入 frontend state-management spec。
- [ ] 仅暂存本任务产品代码、测试和 `.trellis/tasks/08-31-network-ui-transition-fixes/` 文件；保留无关工作树改动。
- [ ] 检查通过后按用户授权创建普通提交；未经再次明确要求不强推、不重写历史。最后运行 `trellis-finish-work`，再报告验证边界。

## 主要风险与回滚点

| 阶段 | 风险 | 回滚点 |
|---|---|---|
| 网络缓存 | stale ECH config 或组偏好在网络切换后继续使用 | 只回退 resolver cache / group preference，保留现有 route ladder |
| 探测结论 | badge 过度乐观，隐藏 DNS 污染证据 | 恢复枚举优先级，保留新增 `dnsDisagrees` 原始字段 |
| 预热 | provider/cache key 不一致或头像晚到 | 保留 `heroImageUrl`/scope 修复，改为路由内加载；不回退首图一致性 |
| Hero clip | 详情最终圆角或飞行尺寸改变 | 去掉 shuttle clip，仅保留稳定 scope 和首图 URL，回到普通 route clip |
| 刷新包装 | 某些 sliver 的通知方向判定差异 | 逐页暂时退回标准 indicator，不触碰 feed controller 和 refresh Future |
