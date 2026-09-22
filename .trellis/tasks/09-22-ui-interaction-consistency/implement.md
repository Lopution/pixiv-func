# 全局交互一致性与页面体验收敛：执行计划

## 1. 父任务规则

本目录只负责审查追踪、任务地图、依赖和最终集成，不运行 `task.py start`，也不直接承载产品代码。当前规划确认前不创建叶子任务；确认后创建的叶子也先保持 `planning`。

## 2. 阶段与依赖

| 阶段 | 工作包 | 前置 | 阶段出口 |
|---|---|---|---|
| 0 | 当前父任务规划 | 无 | PRD、复核矩阵、设计和执行计划通过审阅 |
| 1 | W1 interaction-outcome-correctness | 阶段 0 | 确定性动作/结果错误关闭并合入 `main` |
| 2A | W2 discovery-query-context | W1 | 分类和查询上下文连续 |
| 2B | W3 profile-bookmark-continuity | W1 | 作者/收藏范围和关键动作连续 |
| 3A | W4 artwork-viewer-series-flow | W1，必要的 W2/W3 共享契约 | 作品浏览/查看/系列路径完成 |
| 3B | W5 novel-reader-parity | W1 | 在线/本地小说完整舞台一致 |
| 4 | W6 entity-management-consistency | W2 落地的发现页结构；W4/W5 的进度语义 | 对象组件、排名变体接入与管理页迁移完成 |
| 5A | W7 comment-input-state | W1 | 评论/回复输入状态机完成 |
| 5B | W8 settings-form-semantics | W1 | 设置和诊断表单语义完成 |
| 5C | W9 onboarding-auth-content-layout | W1；W8 的表单/弹层限宽契约 | 引导/登录/文章/宽屏布局完成 |
| 6 | W10 motion-integration-acceptance | W2–W9 | 跨页面一致性、reduced motion 与最终矩阵完成 |

2A/2B、3A/3B、5A/5B/5C 只在文件所有权不重叠且使用独立 worktree 时才可并行。若共享本地化、主题或公共组件，按依赖串行。

规模预期：W6、W8 为最大工作包，创建叶子时应先按 `implement.md` stage 预拆分支（`task/<slug>-<stage>`）；W1 是多点确定性修复合集，规模最小；其余为中等包。规模只影响拆分，不改变依赖顺序。

## 3. 父任务检查清单

- [x] 读取 `/root/Astra.txt` 全文并记录其基线与静态审查限制。
- [x] 对照当前 `main@8215fa8` 复核关键 Widget、事件处理与 Git 历史。
- [x] 记录 PR #48、#50 已吸收范围，避免重复整改。
- [x] 将 25 个编号项、管理页、设置子页、弹层、宽度、动效与视觉角色映射到工作包。
- [x] 定义跨工作包 QueryContext、ObjectPresentation、FormState、BackAndCancel 和 ResponsiveContent 契约。
- [x] 定义阶段、依赖、自动化与运行时验收矩阵。
- [ ] 用户审阅并确认完整范围与工作包边界。
- [ ] 确认后创建 W1–W10 的 leaf task，补齐各自 PRD/design/implement 和 owning files。
- [ ] 每个 leaf 在独立批准后按顺序 start、实现、检查、PR、合并与归档。
- [ ] W10 汇总所有条目的最终状态与跨工作包回归证据。
- [ ] 所有叶子完成后归档本父任务。

## 4. 叶子创建模板

每个工作包创建时必须包含：

1. 从 `research/astra-review-traceability.md` 复制自己拥有的条目和当前状态，不复制其他范围。
2. 记录创建时 `main` SHA、前置 PR/任务和已吸收差异。
3. 列出 owning files、可能消费的 shared components，以及明确不由本叶子修改的相邻页面。
4. 将行为写成 Given/When/Then 或同等可测试断言。
5. 设计 loading/content/empty/error/busy/cancel/back/reduced-motion 状态。
6. 列出聚焦测试、分析、全量测试和设备/桌面人工矩阵。
7. 若条目改变原版或既有肌肉记忆行为（如长按进入隐藏模式、重复点击展开），PRD 写明旧行为 → 新行为的差异与迁移说明。
8. 提交最终 planning summary，收到明确实施批准后才创建分支并 `task.py start`。

建议任务名采用创建日而非本父任务日期，例如 `<MM-DD>-interaction-outcome-correctness`。创建后通过 `task.py add-subtask <parent> <child>` 关联本父任务（`create --parent` 亦可直接建立关联）。

## 5. 每个叶子的实现节奏

### Step A — Rebaseline

- 更新 Astra 条目状态和代码定位；
- 检查前置任务是否已经改变 owning files；
- 运行现有聚焦测试，记录真实基线；
- 若事实使范围实质变化，先回到规划审阅。

### Step B — Contract first

- 先固定状态模型、动作后果和可访问语义；
- 优先扩展现有共享组件，避免复制；
- 为确定性错误先写失败测试或可复现断言；
- 不把视觉调整与数据层重构混在同一 checkbox。

### Step C — Incremental migration

- 一次迁移一个页面族或明确行为；
- 每个 `implement.md` checkbox 对应一个可审查提交；
- 保留兼容行为，直到所有调用者迁移并有测试覆盖；
- 共享组件删除/重命名由最后一个明确 owning stage 执行。

### Step D — Verify and integrate

- 运行聚焦测试和受影响的 golden/widget tests；
- 运行 `/opt/flutter-3.47.2/bin/flutter analyze --no-pub`（项目 SDK；`/opt/flutter-3.47.0` 仅为回滚残留）；
- 按风险运行 `/opt/flutter-3.47.2/bin/flutter test --no-pub`；
- 涉及 Android/WebView/back/plugin 时增加 debug APK 与适用真机检查；
- 运行 `python3 ./.trellis/scripts/task.py validate <leaf>` 与 `git diff --check`；
- 记录未执行的设备/桌面场景，不推断通过。

## 6. 工作包专项门禁

### W1

- 显式 back 与系统 back 分别测试；
- local novel 的空记录、有效记录、越界/陈旧记录分别测试；
- test/clear/cancel/reload 的持久化副作用断言为零或与文案一致；
- mobile/desktop WebView 的每个按钮都断言真实后果；
- §5.6 共享触觉入口在本包落地为单一实现并有基本测试。

### W2/W3

- 跳转前后范围、类型、筛选和私密状态一致；
- 当前控件在 loading/error/empty 中仍可见；
- 大字号/长翻译不通过无限缩字维持单行；
- 排行/作者 header 的点击和滑动规则有回归测试；
- 重复点击当前分类/标签固定为回到顶部，不再承载展开入口。

### W4/W5

- 1、2、长多页、空媒体、Ugoira、系列分页分别验证；
- zoom、toolbar、页码、键鼠输入和 back 优先级分别验证；
- 更新游标与真实阅读进度使用不同状态/断言；
- 在线/本地 reader 对字号、主题、恢复和进度跳转跑同一契约测试；
- 查看器/阅读器主操作有屏幕阅读器可达路径；下载/保存动作按 §5.6 分级触觉。

### W6/W7

- shared object variant 在所有调用点不丢主动作；
- 删除/移除/取消支持确认或撤销并使用正确文案；
- 下载状态动作映射覆盖 queued/running/paused/retryable/completed/canceled；
- 评论输入四态互斥，失败保留草稿，发送成功后的 reply target 行为固定；
- 管理模式进入、危险确认与发送终态使用 §5.6 触觉分级；动作文案按 §5.7 术语表核对。

### W8/W9

- immediate/draft/action/destructive 表单分别有状态测试；
- W8 落地 §5.5 表单/弹层限宽契约并补全 §5.7 术语表（种子已随父规划冻结）；W9 消费而非重定义；
- 320/390/600/840/1200dp、横屏与大字号无不可达主操作；
- Spotlight 正文可选择，登录主动作不因帮助消失；
- mobile sheet/desktop dialog 的语义顺序一致。

### W10

- 逐项关闭追踪矩阵，不以“整体观感较好”代替证据；
- 对默认/reduced motion、触摸/键鼠、中文/长翻译运行代表路径；
- 至少一条 TalkBack/Windows Narrator 代表路径已验证或明确标注未验证；
- 验证没有新平行组件、冲突手势、过时文案（按 §5.7 术语表核对）或绕过共享 SnackBar；触觉通道无第二来源；
- 只修集成暴露的窄问题；新需求另建任务。

## 7. 完成定义

父任务只有在以下条件同时满足时才完成：

- Astra 全部条目都有已合入叶子、证据化不采纳、已吸收或明确延期归属；
- W1–W10 对应叶子已完成并归档，或用户明确缩减范围并更新父 PRD；
- 自动化与运行时矩阵分别记录真实结果；
- 跨页面没有重新出现同动作不同后果、同对象平行实现或隐藏关键入口；
- 父任务最终集成记录区分 implemented、analyzed、unit/widget-tested、desktop/device-tested；
- 未执行或失败项保持可见，不以空操作、弱化断言或视觉占位标记完成。

## 8. 当前停止点

本轮停在阶段 0：父任务已创建，产品代码未修改，未运行 `task.py start`，未创建实现分支。下一步只在用户确认本规划后创建 planning 状态的叶子任务；该确认不自动批准任何叶子开始实现。
