# 执行计划：动效集成与最终验收（W10）

需求见 `prd.md`，技术设计见 `design.md`，逐项改动与回归方法草案见
`research/implementation-draft.md`（行号钉 `main@8067b2d`，阶段 1 重锚）。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。验证命令（每阶段收尾都要跑）：

```bash
dart format --output=none --set-exit-if-changed lib test
flutter analyze --no-pub
flutter test --no-pub
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-motion-integration-acceptance
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：看似不相关的
> 测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec 判定噪声再排查。

本机不可验项（TalkBack/Narrator、触觉手感、手势跟手、进程死亡恢复、真机性能、
`flutter run -d linux` 桌面运行）在台账与 PR 标「未验证」，不得由 widget test
推断通过。

## 阶段 1：Rebaseline（W1–W9 合入核对）

- [ ] 记录当时 `main` SHA；逐 leaf 确认 W1–W9 已合入并归档；未合入 leaf
      对应的验收行标 blocked/未验证，不记绿。
- [ ] research 全部 file:line 引用对 HEAD 复核刷新（motion-inventory §5
      偏差清单、§6 re-tap 基线、§4 反馈通道）；已被责任包修掉的窄修复项
      核销记「已吸收」。结果写入 `research/rebaseline-w10.md`（新文件）。
- [ ] 跑基线聚焦测试并记录真实基线：`motion_test`、`func_bottom_nav_test`、
      `root_swipe_switcher_test`、`hero_transition_test`、
      `app_snack_bar_test`、`navigation_restoration_test`、
      `novel_reader_chrome_test`、`shared_component_semantics_test`。
- [ ] 条件项现状确认：`new_page.dart` `_selectorExpanded`/`AnimatedSize`
      是否被 W2 移除；`local_novels_page.dart` 裸 `showDialog` 是否被 W6
      迁移；`novel_reader.dart`/`novel_page.dart` 闸口点是否被 W5 重构；
      `FuncSemanticTokens.motion*` 是否仍零消费者；re-tap→top 是否已落地
      （未落地则对应行 blocked）。
- [ ] 若事实使范围实质变化，先回到规划审阅，不带着过期假设进窄修复。
      提交：`docs(trellis): W10 验收基线与 W1–W9 合入核对`

## 阶段 2：窄修复（各项以阶段 1 核销后仍存在为前提）

- [ ] **N1**：`lib/app/motion/motion_tokens.dart` `enabled()`（L82-86）OR 入
      `View.of(context).platformDispatcher.accessibilityFeatures.reduceMotion`。
      测试：`test/motion_test.dart` 新增 `accessibilityFeaturesTestValue =
      FakeAccessibilityFeatures(reduceMotion:true)` 用例，断言 `enabled`
      返回 false、`resolve` 落零。
      提交：`fix(motion): MotionTokens.enabled 覆盖平台 reduceMotion`
- [ ] **N2**：`lib/app/motion/drag_to_dismiss.dart:43` 取消返回动画过闸——
      手势结束回调处按 `MotionTokens.enabled(context)` 选 forward 或置终值
      （controller 在 initState 创建时无 context）。
      测试：reduce:true 下取消拖拽 → 直接落终态无飞行动画。
      提交：`fix(motion): DragToDismiss 返回动画接入 reduced-motion 闸`
- [ ] **N3**：`lib/features/illust/detail/widgets/detail_image_pager.dart:67`
      硬编码 180ms+easeOut → `MotionTokens.resolve(context, MotionTokens.fast)`
      + `MotionTokens.fastCurve`。
      测试：键盘翻页在 reduce:true 下瞬时到页。
      提交：`fix(illust): 详情分页器时长走 MotionTokens 闸口`
- [ ] **N4**：`lib/features/novel/novel_page.dart` `_ChromeBar`（L670-728）
      `didUpdateWidget` 的 `forward()/reverse()` 过闸——`!enabled` 直接置
      目标值；`lib/features/novel/novel_reader.dart` L279/L363
      `animateToPage` 时长改 `resolve`。
      测试：`novel_reader_chrome_test.dart` 补 reduce:true 下 chrome 显隐与
      翻页瞬时断言（W5 若已重构则对新代码同法验证）。
      提交：`fix(novel): 阅读器 chrome 与翻页动画接入 reduced-motion 闸`
- [ ] **N5**：`lib/features/new/new_page.dart:134` `AnimatedSize` 时长改
      `MotionTokens.resolve`（若 W2 已移除 `_selectorExpanded` 路径，核销
      记「已吸收」）。
      提交：`fix(new): 选择器展开动画接入 reduced-motion 闸`
- [ ] **N6**：`lib/features/localnovel/local_novels_page.dart:112` 裸
      `showDialog<bool>` → `showAppDialog`（若 W6 已迁移则核销）。
      提交：`fix(localnovel): 删除确认走统一 showAppDialog`
- [ ] **N7**：`FuncSemanticTokens.motion{Short,Standard,Emphasized}`
      （func_semantic_tokens.dart:136-140）仍零消费者则删除三字段；
      已有消费者则保留并在台账注明「引用别名、非第二常量集」。
      提交：`chore(theme): 删除未使用的 motion 别名字段`

## 阶段 3：逐契约回归（父 §5 + §4.10）

- [ ] **C1 静态闸口**：新增 layering 风格守卫测试（`test/architecture/` 扩展
      或新文件）：`SnackBar(`/`showSnackBar(` 出 `app_snack_bar.dart` 为零
      （`showAppSnackBarOn` 调用点白名单）；`HapticFeedback` 恰一 owner +
      W4/W6/W7 白名单调用点；`showDialog|showModalBottomSheet` 出
      `app_overlays.dart` 为零；`Duration\(milliseconds` census 无新增动效
      时长；`Hero(` census 无第二 tag 族。owner 缺席的项标 blocked。
      提交：`test(architecture): 反馈通道与动效常量单一来源静态闸口`
- [ ] **C2 SnackBar 通道**：`app_snack_bar_test.dart` 扩展——分支根 margin
      抬升断言 + `appSnackBarAnimationStyle` 挂载断言。
      提交：`test(app): SnackBar 分支根 margin 与动画样式回归`
- [ ] **C3 re-tap→top**：widget test——tap 当前 tab → `ScrollController` 回 0、
      无刷新调用、无隐藏展开入口；branch-root re-tap `goBranch` 同 index
      语义不变（W2/W3 未落地则 blocked）。
      提交：`test(nav): re-tap 当前标签回顶且不触发刷新`
- [ ] **C4 QueryContext 声明核对**：逐页核对 PageStorageKey/restorationId/
      durable 路由值恢复等级声明（搜索/新作/排行/作者/收藏），与 leaf PRD
      声明对齐；进程死亡恢复标未验证。结果记入台账。
      提交：`test(nav): 查询上下文恢复等级声明逐页核对`
- [ ] **C5 空间连续性 + loading**：grep 各消费点命中
      `illustHeroTag`/`FuncRouteTransition`/`DragToDismiss`；grep `onTapDown`
      preload 未被 `onTap` push await；跑 `hero_transition_test`。
      提交：`test(motion): feed→detail→viewer 单一空间链回归`
- [ ] **C6 同级分类一致**：默认+reduced 双注入下跑 ranking/new/recommended
      tap 与 drag 路径；grep `novel_ranking_page` 已收敛、无 index-swap 残留。
      提交：`test(nav): 同级分类点击/拖动双注入一致性回归`
- [ ] **C7 表单/键盘**：widget test——sheet 内聚焦字段 pump 后主动作不平移
      （shell `resizeToAvoidBottomInset:false` + leaf viewInsets 契约）。
      提交：`test(form): 键盘展开不位移主控件`
- [ ] **C8 术语表核对**：arb key 对 §5.7 逐词核对；冻结文案 focused test；
      改动页文案人工过一遍并记录文件清单入台账。
      提交：`test(l10n): §5.7 动作术语表对照回归`
- [ ] **C9 抽查**：反向搜图 cancel-vs-leave（W1）、profile edit dirty 确认、
      novel reader PopScope chrome-vs-页面顺序——跑既有测试 + 缺口记录。
      提交：`test(regression): FormState/BackAndCancel 三处抽查`
- [ ] **C10 语义树**：`ensureSemantics`/`matchesSemantics` 断言作者头部、
      查看器、评论输入三优先页；TalkBack/Narrator 标未验证。
      提交：`test(a11y): 三优先页语义树代表路径断言`

## 阶段 4：最终验收矩阵 + 父任务归档交接

- [ ] §7 可执行层全跑：宽度 320/390/600/840/1200+横屏（`tester.view
      .physicalSize`）、textScaler 1.0/1.3x、键盘/滚轮模拟、reduced-motion
      三注入矩阵、七态聚焦测试重跑。
- [ ] 产出 `acceptance-ledger.md`：逐行验收项/契约来源/方法/证据/自动化/
      分级（implemented / unit-widget-tested / desktop-tested /
      device-tested / 未验证 / blocked）；未验证行写明缺失面。
- [ ] 更新父任务 `research/astra-review-traceability.md` 状态列 + 父
      implement.md §3「W10 汇总所有条目的最终状态与跨工作包回归证据」勾选。
      提交：`docs(trellis): W10 最终验收矩阵与追踪矩阵关闭`

## 收尾

- [ ] `dart format` 检查、`flutter analyze --no-pub`、全量
      `flutter test --no-pub`（噪声按 spec 判定）、`git diff --check`、
      `task.py validate` 全绿。
- [ ] PR：`gh pr create --fill`（body 列出未验证项）；CI 绿后
      `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后提交）。

## 边界（不做）

- 不重建 MotionTokens/MotionScope、不新造动效组件/常量/反馈通道；
- 不替 W1–W9 完成功能范围；结构问题退回责任包/另建任务（prd D6）；
- 不闸有意不闸项（prd R1 表）；不 shim 框架自有动画（prd D3）；
- 不改网络/数据/持久化语义，不扩 §5.7 术语表（只核对）；
- 不以 widget test 推断设备项通过；未验证项保持显式可见。
