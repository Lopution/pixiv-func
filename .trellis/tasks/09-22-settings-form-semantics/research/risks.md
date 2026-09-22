# W8 风险与 stage 预拆建议

> implement.md §2 已标记 W8 为最大工作包之一，要求叶子规划时按 stage 预拆 `task/09-22-settings-form-semantics-<stage>` 分支。

## Stage 预拆（建议 5 个串行 stage）

| Stage | 内容 | owning files | 规模 |
|---|---|---|---|
| `s1-foundation-root` | `SettingsTile.subtitle`、限宽包装、`persistSettings` 收敛（删 `_persistNetwork`/`_write` 副本）、PopScope dirty-guard 模式、根页全量摘要行 | `app/widgets/settings/*`、`settings_helpers.dart`、`settings_page.dart`、（可能）`network_settings_page.dart` 仅收敛部分 | 中 |
| `s2-browse-simple` | 主题/语言选中态、浏览页归组重排+自定义图源 draft 守卫、历史页 pixivHistory 去重、屏蔽页解除动词+失败输入恢复+pending 可视 | `theme/language/browse/history/muted` 五页 | 中 |
| `s3-account-translate-download` | 账号切换 busy、凭据页确认+双色 status+清输入框、下载设置 slider/命名 draft 语义、保存位置人类可读名 | `account/translate/translation_credentials/download_settings/download_destination` | 中 |
| `s4-network-diagnostics` | 网络主页摘要优先级、高级页统一 draft/重置确认、probe 先摘要后细节、帧探针录制/控制分离 | `network_settings_page.dart`、`network_probe_page.dart`、`frame_probe_page.dart` | 中大 |
| `s5-about-backup-terms` | 仓库链接打开/复制、更新错误按原因拆分、备份两步策略流、术语全量核对+矩阵验收 | `about_settings_page.dart`、`backup_settings_page.dart`、4 份 arb | 中 |

拆分支线：s1 先行（其余 stage 都消费 subtitle/限宽/guard）；s2-s5 文件不相交，可串行合入（同一 worktree 规矩）；l10n key 按 stage 批量加入，避免 4×arb 反复冲突。

## 风险登记

- **R1 W1 前置未落地**：W1 仍 `planning`。browse `_testMirror` 隐式持久化（browse:86→:68）、凭据清除不同步输入框（credentials:131-156）等"W1 修…"项在当前 HEAD 仍在。W8 rebaseline 必须先确认 W1 合入内容；若 W1 延期，s2/s3 的相关控件按"现状+W8 目标态"处理并在 PRD 写明接法分支。**决策点：W1 未合并时 W8 是否吸收其设置域条目。**
- **R2 限宽值无现存 owner**：`AppBreakpoints` 只有断点（600/1200），无内容栏宽常量；`welcome_page` 用 520 属 W9 文件。建议设置取 600（对齐 medium 断点语义）并写入本包 design；若 W9/W3 后续取不同值会出现"各包栏宽不一"——**建议在本包 design 中登记候选值与理由，W10 验收时核对一致性**。决策点 D1。
- **R3 共享件扩展规则**：`SettingsTile.subtitle` 属 `app/widgets/` 改动，必须与首个消费者（根页）同 commit——满足"扩展+接入同 stage"。勿提前提交孤立扩展。
- **R4 历史入口归属**：根页"历史"tile 指向 `/settings/history`（开关页），`openHistory` 直达 `/settings/history/view`。"查看历史绕经配置页"修法两选：根页 tile 直达 view（配置摘要仍显示开关态）或保留两级。**决策点 D5——涉及导航习惯变化，须 PRD 写明。**
- **R5 凭据"已配置"状态需 core 新读法**：`TranslationCredentialsStore` 无存在性查询；加 `hasBaidu()/hasLlm()`（只判 key 存在，不读 secret）是 `core/comments/` 文件修改——属本包边界内最小 core 触碰，须在 PRD/implement 声明。若拒绝改 core，可用 `readBaidu()!=null` 一次性读（成本：读 secret 字段但仅判空）。
- **R6 SAF 人类可读名是启发式**：tree URI 末段解码覆盖 `primary:`/SD 卡卷标；解码失败必须回退原串，不能吞错。纯 UI 层函数+单测可覆盖常见形态。
- **R7 帧探针"离开继续录"**：`_frames` 无上限（frame_probe.dart:22）——长录制内存增长；建议 PRD 声明 cap（如 10k 帧滚动丢弃）或文档化限制。另一风险：用户离开后忘记录制中——需要"录制中"的持续可见指示（页内状态条 + 可选根页摘要/返回时提醒）。**决策点 D4。**
- **R8 第三方可达性进页面即发包**（network_settings:481 initState→_check）：保留自动 vs 改手动是行为变化；保守做法是保留自动但归入 action 语义明示。**决策点。**
- **R9 备份两步流程**：radio+确认的两段式 vs 单 dialog 内选择+同权确认——M3"dialog 最多两动作"倾向后者内的 radio 方案；`showAppDialog`/`showAppBottomSheet` 均可用。**决策点 D3。**
- **R10 W6 边界**：`download_tasks_page.dart`、`HistoryPage` 归 W6；本包只可做根页入口摘要（读 `downloadManagerProvider` 快照），不得改任务页 UI/动作映射（暂停→retry 图标项归 W6）。
- **R11 测试陷阱**：`material_ui` 遮蔽（SwitchListTile/SnackBar 断言要用其类型）；`settingsProvider` AsyncNotifier `.future` 错误态不 settle——断言 error state 用 listen（quality-guidelines 已载）；四语齐全测试 `settings_test.dart:522` 会因缺 key 失败。
- **R12 动词本地化风险**："取消屏蔽"→"解除屏蔽"改动影响 ja/ru/en 对应文案，需统一审校；属于种子语义执行而非新造词。
- **R13 `_busy`/`_saving` bool 散落**：六处页面各自 bool——不强制抽 FormState 封装（避免过度设计），但每页须在 PRD/design 声明控件分类；若实现中发现 ≥3 页同形 busy 按钮可提取私有 helper 于 settings_helpers（feature 内，不违反组件层规则）。

## 决策点汇总（供 design/implement 定稿）

- D1 设置栏宽值：600（对齐 medium）/640/720；是否全宽度生效（建议）或仅 ≥expanded 门控。
- D2 Slider（最大并行数）归类：immediate-with-preview（保现状语义写明）vs draft。
- D3 备份策略选择呈现：单 dialog radio+同权确认 vs 两段 dialog vs bottom sheet。
- D4 帧探针离开后的可发现性方案与 `_frames` 上限策略。
- D5 根页历史入口直达 `/settings/history/view` 与否。
- D6 屏蔽项 trailing 图标是否随"解除屏蔽"动词一并更换。
- D7 第三方可达性自动探测是否保留进页即发。
- D8 凭据 configured 态读法：core 加 hasX() vs UI 层 read 判空。
