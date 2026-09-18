# 执行计划：开源竞品审计落地

> 父任务不产出代码。本文件定义子任务顺序、各 child 的进入条件、验证门与回滚点。
> 每个 child 激活前仍需补齐自己的 `design.md`/`implement.md`（复杂任务）或仅 `prd.md`（轻量），
> 并 `task.py add-context` 填充 implement.jsonl/check.jsonl 后才能 `task.py start`。

## 波次与依赖

```
Wave 1: novel-domain-repair ─┐
        motion-layer ────────┤  (并行可做；motion 先立语法)
Wave 2: card-quick-actions ─→ mute-system ─→ search-filter-v2
Wave 3: bookmark-tags, content-expansion, network-account-settings, reverse-search-engines
Wave 4: download-v2, feed-resilience, watchlist-local-library(依赖 wave1 小说)
Wave 5: tablet-desktop-layout
```

硬依赖：
- `mute-system` 依赖 `card-quick-actions`（长按菜单是屏蔽入口；mute 的注册项走 §2 动作注册表）
- `watchlist-local-library` 依赖 `novel-domain-repair`（小说域先可用）
- `tablet-desktop-layout` 建议最后（功能面稳定）

软依赖（建议但不强制）：
- `motion-layer` 尽量在 `card-quick-actions`/`mute-system` 之前，避免后补动效返工
- `feed-resilience` 在各 feed 消费者稳定后做，减少快照 schema 变更次数

## 每个 child 的进入/退出门

进入：`git switch -c task/09-16-<slug>`（从 main)→ `task.py start <dir>` → 补齐 child 的 design/implement + jsonl 上下文。

退出（PR 前置）:
- `flutter analyze` 0 issue、`dart format --set-exit-if-changed .` 干净、`flutter test` 全绿（含 layering_test)
- `git diff --check` 干净；分支 rebase 到当时 main
- 若动 `.trellis/workspace/`：按 AGENTS.md 重建 journal(main 版本 + 本 session 块，Session 标题严格递增）

## 逐 child 检查表（提交粒度 = implement.md 勾选框）

- [ ] 1. `novel-domain-repair`：根因证据 → 修复 + 回归测试 → bookmark/comments/ranking/trending-novel 各一步
- [ ] 2. `motion-layer`:token 扩展 → 页面转场 → 列表/反馈动效 → 降级开关 → 审计存量动效改 token 引用
- [ ] 3. `card-quick-actions`：动作注册表 + 长按菜单 → watch-later 域 → 页面 + 卡片集成
- [ ] 4. `mute-system`:MuteStore + 服务端同步 → 模糊揭示卡片变体 → 管理页 → 长按菜单注册项
- [ ] 5. `search-filter-v2`：新过滤字段进 query/wire → filter sheet 分组 → cursor 校验与测试
- [ ] 6. `bookmark-tags`:detail/tags 端点 → 收藏 UI 标签选择 → 管理页
- [ ] 7. `content-expansion`:illust series → pixivision 列表/详情
- [ ] 8. `network-account-settings`：镜像预设 → ai-show/restricted-mode 同步 → 导出导入
- [ ] 9. `reverse-search-engines`:provider 抽象 → 三引擎实现 → 引擎选择 UI
- [ ] 10. `download-v2`:Range 续传 → 批量任务 → caption 导出 → 命名变量
- [ ] 11. `feed-resilience`：快照 schema/恢复 → 离线队列表/重放 → 遥测可观测
- [ ] 12. `watchlist-local-library`:watchlist 域 → 本地 txt 导入/库页 → TXT 导出
- [ ] 13. `tablet-desktop-layout`：断点体系 → 详情 two-pane → 桌面导航/窗口化

## 集成评审（parent 收尾时做）

- 13 个 child 全部 archive 后，对照父 PRD「需求集」逐项回归：每个需求在 main 上可演示
- 交叉验收：长按菜单含 mute 项；mute 卡片在搜索/推荐/排行一致模糊；快照覆盖新 feed；离线队列覆盖 mute 操作
- 更新 spec：新增的 owner（mute store、动作注册表、快照、队列、motion token 治理）写回 `.trellis/spec/`

## 回滚点

任一 child 合入后出问题：`git revert -m 1 <merge-sha>` 单 child 回退，不动兄弟 child。
数据表新增全部走 additive migration；回滚代码后旧版本读到新表不崩（设计 §7)。
