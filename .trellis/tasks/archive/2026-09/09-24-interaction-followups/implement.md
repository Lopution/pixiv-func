# 执行计划：交互一致性后续修复

需求见 `prd.md`（含全部 file:line 定位，基于 `main@463ff33`，动手前对 HEAD 复核）。

## 环境

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

每阶段收尾：`dart format --output=none --set-exit-if-changed lib test`、
`flutter analyze --no-pub`、聚焦测试、`git diff --check`、
`python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-24-interaction-followups`。
全量测试的本机 loopback 噪声按 spec 判定；提交前必跑 `dart format lib test`（CI 门禁）。

## 阶段 1：收藏弹层（A 组）

- [ ] **A1** `lib/app/widgets/bookmark_switch_button.dart`：`awaitingPrefill &&
      !prefillFailed` 时禁整组编辑区——`SegmentedButton.onSelectionChanged` 置 null、
      标签输入与建议 chips 禁用（沿用 sheet 内已有禁用样式）。测试：延迟返回带
      标签详情，加载期尝试切 restrict，详情到达后标签完整填入并可提交。
      提交：`fix(bookmark): 预填充到达前禁整组编辑，阻断旧标签被空值提交`
- [ ] **A2** 核 `showAppSheet`/`showModalBottomSheet` 调用链是否已按 viewInsets
      收缩可用区；无则让 sheet 约束随 `MediaQuery.viewInsetsOf` 调整，保证
      键盘展开时输入框与确认键可达。测试：注入 viewInsets 后 confirm 仍命中。
      提交：`fix(bookmark): 收藏弹层可用高度随键盘收缩`

## 阶段 2：作者页与导航（B 组）

- [ ] **B1** `lib/features/profile/profile_header_delegate.dart`：返回/更多等关键
      操作移入贯穿折叠全程的常驻层（不随 expandedIdentityExitProgress 移除、
      不受 IgnorePointer 误封）；展开/收起只换身份布局。测试：collapse progress
      0.60/0.80/0.95 注入下 back 与更多动作可实际触发。
      提交：`fix(profile): 头部关键操作贯穿折叠全程可达`
- [ ] **B2** `lib/features/profile/user_page.dart` re-tap：改从活跃 tab context
      取 `Scrollable.of(ctx).position.animateTo(0)`；不得包独立
      PrimaryScrollController（会脱离 NestedScrollView innerController）。
      测试：tab A 滚动→切 tab B 滚动→对 B re-tap→断言 B 回顶且 A 位置不变。
      提交：`fix(profile): re-tap 只回顶活跃标签，保留其他页位置`
- [ ] **B3** `lib/features/home/home_page.dart:169` NavigationRail 接底栏同一
      导航动作入口（同 index→re-tap、异 index→goBranch）；若入口在
      BranchSlideStack 内部则提取为两者共用。测试：宽/窄布局三场景 parity。
      提交：`fix(nav): 宽屏侧栏与底栏共用导航动作入口`
- [ ] **B4** `profile_header_delegate.dart` 标签槽：对齐发现页决策序
      （文案→空间→横滚→有限缩放兜底），不再让 FittedBox 缩到不可读。
      测试：窄屏+长翻译+5 标签场景不溢出且字号不低于可读下限。
      提交：`fix(profile): 标签槽决策序对齐发现页，缩放只作兜底`

## 阶段 3：阅读器（C 组）

- [ ] **C1** `novel_reader_stage.dart` sheet % 统一为 `(preview+1)/pageCount*100`
      （对齐 footer L192/364）；补 1/2/N 页边界断言。
      提交：`fix(novel): 进度弹层百分比与底栏统一口径`
- [ ] **C2** `_persistAnchor`（L182-184）：`unawaited` 改带 catch + debugPrint。
      提交：`fix(novel): 锚点持久化失败可观测`
- [ ] **C3** `lib/core/novel/reader_settings.dart` `write()`：赋值前先
      `all.remove(key)` 保证重写到队尾；测试：旧条目重读后不被新增淘汰。
      提交：`fix(novel): 进度存储重读条目真正更新 LRU 序`
- [ ] **C4** `novelSettingsSaveFailed` 四语言 arb 文案改为「仅本次生效，
      未能保存」语义；`flutter gen-l10n` + `python3 tool/gen_l10n_lookup.py` +
      `dart format`（lookup 生成物必须过 format）。测试断言新文案挂载。
      提交：`fix(novel): 设置保存失败文案明示仅本次生效`

## 阶段 4：入场错峰（D 组）

- [ ] **D1** `lib/app/motion/feed_entrance.dart`：错峰延迟仅作用于首屏批
      （首批可见/首帧内卡），持续滚动新曝光卡 `delayUs=0`。测试：index≥9 的
      新曝光卡入场无 240ms 零透明等待；首屏批错峰保留。
      提交：`fix(motion): 入场错峰限定首屏批，持续滚动零等待`

## 阶段 5：记账同步 + 收尾

- [ ] **E1** 父任务文档同步：`09-22-ui-interaction-consistency` 的停止点/
      W6 残留措辞更新为实际（#83/#84 已合入形态）。只改文档。
      提交：`docs(trellis): 父任务停止点与 W6 措辞同步实际实现`
- [ ] rebase `origin/main`（#79/#82 若先合入注意 user_page/settings 重叠区）、
      全量测试、`task.py validate`、`gh pr create --fill`（body 列未验证项）、
      CI 绿后 `gh pr merge --merge`。
- [ ] `add_session.py` + `task.py archive` 为分支最后两提交随 PR 合入。

## 边界

同 prd.md：不新建动效组件/通道/常量集；不做逐字段 draft 合并框架；
不改数据层语义；不为美观追加新动效。
