# 动效集成与最终验收（Roadmap W10）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图）。
规划基线：`main@8067b2d`（research 全部行号对该基线核实）；执行基线是 W1–W9
全部合入后的 `main`，由 implement.md 阶段 1 rebaseline 重锚。

## Goal

本包是路线图收口包，性质为 **gap-closing + 集成验收，不是重建**：
`MotionTokens`/`MotionScope` 单一闸口、`AppSettings.reduceMotion` 设置开关
（app_settings.dart:255 → browse_settings_page.dart:349-356 → app.dart:200-210）、
`app_overlays` 统一弹层入口、#48 共享 SnackBar 通道、17 项 `motion_test.dart`
门禁均已存在且测试全绿。W10 双轨推进：

1. **窄修复**：只修集成验收暴露的、最小 diff 可关闭的确定性缺口——闸口漏接、
   硬编码时长、裸 overlay 入口、iOS reduce-motion 覆盖洞（见 R2）；
2. **验收矩阵**：对父 design.md §4.10 动效八项、§5 跨包契约、§7 运行时矩阵
   逐条取证关闭，产出分级证据台账并向父任务归档交接（见 R3–R6）。

不借验收新建设计系统、平行入口或公共动效组件（父 §3、§4.11）。

## Requirements

### R1. Reduced-motion 接入收敛到单一闸口

- `MotionTokens.enabled`（`lib/app/motion/motion_tokens.dart:82-86`）是唯一
  应用闸口：`MediaQuery.disableAnimations` OR `MotionScope.reduce`；
  `resolve()`（L91-93）把时长压为零。消费模式只允许四种：时长参数走
  `resolve`（先例 routes.dart:166,198、press_scale.dart:106）；分支语义走
  `enabled ? animate : snap`（func_bottom_nav.dart:114,139,636,661、
  branch_slide_stack.dart:253,336、root_swipe_switcher.dart:258,266、
  feed_entrance.dart:128）；overlay 呈现走 `AnimationStyle.noAnimation`
  （app_overlays.dart:23,49）；无 BuildContext 的 controller 注入
  `bool Function()`（branch_slide_stack.dart:26,43 先例）。
- 关闭 iOS 覆盖洞：平台 `AccessibilityFeatures.reduceMotion` 不置位
  `disableAnimations`（Flutter 官方行为，external-reduced-motion.md §1），
  `enabled` 需 OR 入
  `View.of(context).platformDispatcher.accessibilityFeatures.reduceMotion`——
  一行改动 + 一个 `accessibilityFeaturesTestValue` 用例。
- 有意不闸项固定为下表并写入 design.md 决策区，验收时作负向断言
  （它们在 `reduce:true` 下必须仍工作）：

  | 项 | 位置 | 不闸理由 |
  |---|---|---|
  | PixivImage 淡入/淡出 | `pixiv_image.dart:46,89,647-648` | 加载反馈；fadeOut>fadeIn 不对称是白闪回归的承重契约 |
  | SmoothWheelScroll 滚轮时长 | `smooth_wheel_scroll.dart:79,122-132` | 功能性滚动，非装饰；`AnimationBehavior.preserve` 平台闸本就不塌陷它 |
  | PullToRefresh 指示器 | `pull_to_refresh.dart` | 刷新反馈通道 |
  | Ugoira `frame.delayMs` | `ugoira_viewer.dart:399` | 内容行为，非动画装饰 |
  | `appSnackBarAnimationStyle` | `app_snack_bar.dart:90` | 反馈通道；平台 `disableAnimations` 已自动塌陷（决策 D2） |
  | 框架自有动画 | TabBar `kTabScrollDuration`、NavigationRail 展开过渡（home_page.dart:164-180） | 应用闸够不到；平台闸覆盖；记已知限制（决策 D3） |

### R2. 窄修复清单（以 rebaseline 时仍存在为前提）

全部 file:line 已对 `main@8067b2d` 核实（codebase-motion-inventory.md §5）；
责任包可能已先行修掉，rebaseline 时逐项核销：

| 位置 | 缺口 | 修法 | 文件责任包 |
|---|---|---|---|
| `motion_tokens.dart:82` | `enabled` 不读 iOS `reduceMotion` | OR `accessibilityFeatures.reduceMotion` | 本包（motion 层） |
| `detail_image_pager.dart:67` | 硬编码 `Duration(milliseconds:180)`+`easeOut`，双闸都绕过 | `MotionTokens.resolve(context, MotionTokens.fast)` + `fastCurve` | W4 |
| `drag_to_dismiss.dart:43` | `_returnAnimation` 用 token 未过闸 | 手势结束回调处按 `enabled` 选 forward/置终值（initState 无 context，闸放调用点） | 本包（motion 层） |
| `novel_page.dart:670-728` `_ChromeBar` | token 已用，`didUpdateWidget` 的 `forward()/reverse()` 未过闸 | `!enabled` → 直接置目标值 | W5 |
| `novel_reader.dart:279/363` | `animateToPage(duration: MotionTokens.fast)` 未过闸 | `resolve(context, MotionTokens.fast)` | W5 |
| `new_page.dart:134` `AnimatedSize` | 未过闸隐式动画 | `resolve`；若 W2 已移除 `_selectorExpanded` 路径则核销 | W2 |
| `local_novels_page.dart:112` | 裸 `showDialog<bool>` 绕过 `showAppDialog`（无 `MotionTokens.dialog`、无闸） | 一行换成 `showAppDialog`；若 W6 已迁移则核销 | W6 |

**越界处理规则**：验收发现的越界问题优先按窄修复并入本包（最小 diff 原则，
上表即全部获批修复面）；凡涉及结构变更（状态机重写、新组件、新路由、
schema、布局重构）一律退回责任包或另建任务，本包只在台账登记缺口，不顺手做。

### R3. 逐契约回归（父 §5 + #48）

逐条按 design.md §四 ledger 方法取证，每条标自动化/半自动/人工：

- §5.6 SnackBar 单通道：`rg "SnackBar\(|showSnackBar\(|ScaffoldMessenger\." lib/`
  白名单闸口——只允许 `app_snack_bar.dart` owner 文件与 `home_page.dart:111`、
  `app.dart:87` 两个经 `showAppSnackBarOn` 的合法调用点；可固化为 layering 风格
  静态测试。`app_snack_bar_test.dart` 扩展断言分支根 margin 抬升与
  `appSnackBarAnimationStyle` 挂载。
- §5.6 触觉单来源：`rg "HapticFeedback" lib/` 恰为一个 owner 文件 +
  白名单调用点（W4 下载/保存、W6 管理模式/危险、W7 发送终态）；
  owner 不存在则该行标 blocked，不记绿。
- §5.7 ActionTerminology：arb key 对照 §5.7 表逐词核对（应用/保存/删除/
  移除/解除屏蔽/取消/重试/重新登录/重新打开/返回/查看）；叶子冻结文案处
  跑 focused test；改动页面文案人工过一遍并记录检查过的文件。
- §5.2 ObjectPresentation：`NovelCard|NovelRow|IllustCard` 调用点 census，
  断言无 feature 本地平行卡片类（layering 测试模式可扩展）；视觉等价人工。
- §5.1 QueryContext + re-tap→top：逐页核对 PageStorageKey/restorationId/
  durable 路由值的恢复等级声明（component-guidelines Route Restoration
  Contract）；re-tap widget test——tap 当前 tab → `ScrollController` 回 0、
  无刷新调用、无 `_selectorExpanded` 式隐藏展开入口；branch-root re-tap 的
  `goBranch` 同 index 语义不变。
- §5.3 FormState / §5.4 BackAndCancel：不批发重验（叶子各负其责），
  W10 抽查三处——反向搜图 cancel-vs-leave（W1）、资料编辑 dirty 确认、
  novel reader `PopScope` chrome-vs-页面顺序。
- §5.5 overlays/breakpoints：`rg "showDialog|showModalBottomSheet" lib/`
  出 `app_overlays.dart` 应为零；`AppBreakpoints`/`two_pane` 消费编译锁定。
- 无第二常量来源：`rg "Duration\(milliseconds" lib/` census 停留在既有
  非动效集合（frame_probe 轮询、ugoira 帧延时、wheel 下限、core 常量）+
  `motion_tokens.dart`；`FuncSemanticTokens.motion{Short,Standard,Emphasized}`
  未使用别名按决策 D5 标记处理。

### R4. §4.10 动效八项逐条验收

同级分类点击/拖动一致；作者头部仅改布局不改可达动作；查询信息在输入/结果间
连续；feed→detail→viewer 单一空间连续性链（`illustHeroTag` scope 配对 +
`FuncRouteTransition` + `DragToDismiss`，无第二 Hero tag 族）；表单/键盘动画
不移动正在操作的主控件；loading 动效不延迟真实内容；reduced motion 只去
装饰性位移/缩放；#48 SnackBar 与 MotionTokens 继续作为唯一共享来源。
逐项给测试 id / grep 证据 / 人工标记，不以"整体观感较好"关闭（方法详见
design.md §五）。

### R5. §7 运行时矩阵与证据分级

- 本环境可执行层：focused + 全量 `flutter test --no-pub`、
  `analyze --no-pub`、`dart format` 检查、`git diff --check`、
  `task.py validate`；宽度 320/390/600/840/1200+横屏用
  `tester.view.physicalSize` widget 测试；文本 1.0/1.3x 用 `textScaler` 注入；
  键盘 Tab/Enter/Escape 与滚轮用 `Focus`/`KeyEvent`/`PointerScrollEvent`
  模拟；屏幕阅读器用 `ensureSemantics`/`matchesSemantics` 语义树断言
  （作者头部、查看器、评论输入三个优先页）；动效默认/reduced 用
  `MediaQuery`/`MotionScope`/`accessibilityFeaturesTestValue` 三层注入。
- 台账每行分级：implemented / unit-widget-tested / desktop-tested /
  device-tested / **未验证** / blocked。本环境无 Android 模拟器/设备、
  无 Windows host、无 iOS 工具链，`flutter run -d linux` 未证实——
  TalkBack/Narrator 代表路径、触觉手感、手势跟手、进程死亡恢复、真机性能
  一律标「未验证」并写明缺失面，不得由 widget test 推断通过。

### R6. 追踪矩阵逐项关闭与父任务交接

- Astra 全条目 + 增补契约项（触觉分级、屏幕阅读器、re-tap 回顶、术语表、
  恢复等级）逐条拿到证据化终态：已合入叶子 / 已吸收 / 有证据的不采纳 /
  明确延期归属；更新 `research/astra-review-traceability.md` 状态列。
- 产出本 leaf `acceptance-ledger.md`（逐行：验收项/方法/证据/分级），
  勾选父 implement.md §3「W10 汇总所有条目的最终状态与跨工作包回归证据」；
  父任务归档由父任务自身节奏完成，本包交付其输入。

## Acceptance Criteria

- [ ] rebaseline 记录：当时 `main` SHA、W1–W9 各 leaf 合入/归档状态、
      research 行号复核结果；未合入 leaf 的对应验收行标 blocked/未验证。
- [ ] R2 窄修复每项完成或经 rebaseline 证明已不存在（台账记「已吸收」）；
      完成的项各有 ≥1 个 focused test，含 iOS `reduceMotion` 的
      `accessibilityFeaturesTestValue` 用例。
- [ ] 静态闸口全绿：`HapticFeedback` 恰一 owner；`SnackBar(`/`showSnackBar(`
      白名单制；`showDialog|showModalBottomSheet` 出 `app_overlays.dart` 为零；
      `Duration\(milliseconds` census 无新增动效时长；`Hero(` census 无第二
      tag 族。
- [ ] `FuncSemanticTokens.motion*` 零消费者别名已按 D5 处理并在台账写明结论。
- [ ] §4.10 八项、§5 各契约、§7 矩阵每行有测试 id / grep 证据 / 人工标记 /
      「未验证」标签，无证据行不得关闭。
- [ ] reduced-motion 矩阵：全部过闸 widget 在 `MotionScope(reduce:true)`、
      `disableAnimations:true`、`reduceMotion:true` 三注入下落终态；
      不闸项负向断言仍工作。
- [ ] 每 commit 对应 implement.md 一个勾选框；分支
      `task/09-22-motion-integration-acceptance`。
- [ ] `flutter analyze --no-pub`、`dart format` 检查、focused + 全量
      `flutter test --no-pub`、`git diff --check`、`task.py validate` 全绿。
- [ ] `acceptance-ledger.md` 与 traceability 状态列更新随本包 PR 合入；
      未验证项显式可见。

## Decisions（规划定案）

- D1 `MotionTokens.enabled` 拓宽纳入平台 `accessibilityFeatures.reduceMotion`
  （iOS 覆盖洞）——采纳；一行 + 一个测试，不新增第二闸口。
- D2 `appSnackBarAnimationStyle` 保持不闸：SnackBar 是反馈通道非装饰，
  平台 `disableAnimations` 已把其 controller 压至 5%；台账记为有意决策，
  评审异议时三行可闸（`AnimationStyle.noAnimation`），本包不预改。
- D3 框架自有动画不接应用内闸：`TabBar` 点按 `kTabScrollDuration` 置零需
  替换点按语义（`controller.index = i`），NavigationRail 展开过渡同理；
  平台闸已覆盖，记为已知限制，不逐页 shim。
- D4 不闸项清单按 R1 表固定；禁止把 PixivImage 淡入、wheel 平滑、刷新指示、
  Ugoira 帧时序"顺手"过闸——它们是反馈/功能/内容行为。
- D5 `FuncSemanticTokens.motion{Short,Standard,Emphasized}`
  （func_semantic_tokens.dart:136-140）：rebaseline 若仍零消费者则随窄修复
  删除三字段（消除"第二来源"歧义）；若已有消费者则保留并在台账注明其为
  引用别名、非第二常量集。
- D6 越界文件（W2/W4/W5/W6 所有）的闸口级缺口按窄修复并入本包；
  结构问题退回责任包/另建任务并在台账记缺口，不扩大本包 diff。
- D7 不以 golden 截图做动效证据（golden 动效盲）；golden 仅用于稳定
  宽度/字体场景，动效断言走 controller/时长/终态。
- D8 不可验面不做替代性通过：设备缺失项只做声明核对 + 「未验证」标记。

## Out of scope

- 重建或扩展 MotionTokens/MotionScope、新公共动效组件、新反馈通道、新路由。
- W1–W9 职责范围内的功能改动（页面层级、状态机、业务语义、视觉收敛）。
- 接管 `AnimationBehavior.preserve` 物理/无界动画；逐页 shim 框架自有动画。
- 触觉手感、TalkBack 朗读、真机 60fps 等需设备能力的「替代性通过」；
  新需求与评审发现的非窄问题另建任务。
