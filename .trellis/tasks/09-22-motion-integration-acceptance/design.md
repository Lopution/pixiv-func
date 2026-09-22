# 技术设计：动效集成与最终验收（W10）

设计基线：`main@8067b2d`；research 行号均对该基线核实，执行时由
implement.md 阶段 1 rebaseline 重锚。窄修复与回归方法草案见
`research/implementation-draft.md`；环境限制与不可验面见 `research/risks.md`；
动效原语与偏差清单见 `research/codebase-motion-inventory.md`；测试基建见
`research/codebase-test-infra.md`；Flutter reduced-motion 契约见
`research/external-reduced-motion.md`。本文件只固定闸口契约、窄修复边界、
逐契约回归方法与台账格式。

## 一、阶段划分与依赖

前置：W1–W9 全部合入 `main`（父 implement.md §2 阶段 6）。单分支
`task/09-22-motion-integration-acceptance`；若超出单个可审查 PR，按
implement.md 阶段拆 `-s1`（rebaseline+窄修复）/`-s2`（回归+矩阵）。

| 阶段 | 内容 | 出口 |
|---|---|---|
| 1 Rebaseline | W1–W9 合入核对、行号重锚、基线聚焦测试、窄修复清单核销 | 基线记录 + 有效窄修复清单 |
| 2 窄修复 | prd.md R2 表七项（已核销项跳过） | 各项带 focused test |
| 3 逐契约回归 | §5.1–§5.7 + §4.10 八项，grep 闸口 + widget/语义测试 | ledger 逐行有证据 |
| 4 最终矩阵 + 交接 | §7 可执行层、分级台账、traceability 状态列与父 §3 勾选 | `acceptance-ledger.md` + 父任务输入 |

## 二、Reduced-motion 闸口契约

### 单一闸口（已存在，验收而非重建）

- `MotionTokens.enabled`（motion_tokens.dart:82-86）=
  `MediaQuery.disableAnimations` OR `MotionScope.reduce`；W10 窄修复 OR 入
  `View.of(context).platformDispatcher.accessibilityFeatures.reduceMotion`，
  关闭 iOS「减弱动态效果」覆盖洞（该 flag 不置位 `disableAnimations`）。
- `MotionTokens.resolve`（L91-93）塌陷时长为零——语义是「移除飞行，不移除
  它表达的状态」（token 文件原注释）。
- `MotionScope` 挂 `MaterialApp.builder`（app.dart:200-210）读
  `settings.reduceMotion`，覆盖全部路由/sheet/dialog context
  （含 `useRootNavigator` 弹窗）。
- 四种合法消费模式（enforce，不新造）：时长 `resolve`、分支 `enabled`、
  overlay `AnimationStyle.noAnimation`、非 context 场景注入
  `bool Function()`。

### 有意不闸项（决策区，验收作负向断言）

| 项 | 角色 | 理由 |
|---|---|---|
| PixivImage `imageFade`/`imageFadeOut` | 反馈 | 冷载淡入是加载反馈；fadeOut>fadeIn 不对称与 `useOldImageOnUrlChange` 共同承重白闪回归契约，塌陷可能复现白闪 |
| SmoothWheelScroll `wheelScroll` | 功能 | 滚轮步进平滑是滚动功能本体；`AnimationBehavior.preserve` 使平台闸本就不作用 |
| PullToRefresh 指示器 | 反馈 | 刷新进行中的可见反馈，非装饰 |
| Ugoira `frame.delayMs` | 内容 | 帧时序是内容行为；spec：reduced motion 不得移除 |
| `appSnackBarAnimationStyle` | 反馈 | SnackBar 是结果反馈通道；平台闸已自动压 5%，in-app 闸不接（D2） |
| TabBar `kTabScrollDuration`、NavigationRail 展开 | 框架 | 应用闸不可达；置零需替换点按语义；记已知限制（D3） |

判断规则：**装饰可去，状态必达**。reduced motion 只移除位移/缩放类装饰；
反馈可保留淡化但不得消失；内容与物理不动。

### 窄修复 vs 退回责任包

- 闸口级缺口（加 `resolve`/`enabled`/`AnimationStyle`、换 `showAppDialog`、
  删零消费者别名）→ 本包窄修复，跨责任包文件亦然（D6；最小 diff）。
- 结构变更（状态机、组件层级、路由、schema、布局重构）→ 退回责任包或
  另建任务；本包台账登记缺口并标 blocked，不扩大 diff。
- 顺序：窄修复排在对应 leaf 合入之后；rebaseline 时已被责任包修掉的项
  核销记「已吸收」。

## 三、测试基建分层（本包最容易踩错的坑）

两套 flag 两层注入，混用会产生假绿：

| 层 | 生产读取 | 测试注入 | 覆盖范围 |
|---|---|---|---|
| 平台 `disableAnimations` | `MediaQuery.disableAnimations` | `tester.platformDispatcher.accessibilityFeaturesTestValue = FakeAccessibilityFeatures(disableAnimations:true)` | MediaQuery + SemanticsBinding + 所有 `AnimationBehavior.normal` controller（×0.05、fling ×200） |
| iOS `reduceMotion` | `accessibilityFeatures.reduceMotion`（窄修复后并入 `enabled`） | `FakeAccessibilityFeatures(reduceMotion:true)` | 只经 `MotionTokens.enabled` 生效 |
| 应用内 `MotionScope.reduce` | `MotionScope.maybeOf(context)` | `MotionScope(reduce:true)` 包裹 | 仅 `MotionTokens` 消费点 |

- `MediaQuery(data: ...disableAnimations:true)` 覆盖**只喂应用闸**，不塌陷
  framework controller——断言「framework 路径也塌陷」必须用
  `accessibilityFeaturesTestValue`；断言应用闸用 `MediaQuery`/`MotionScope`
  注入（motion_test.dart `_wrap` 先例）。
- 应用内 `reduceMotion` 不影响 framework：用 token 未过闸的 widget 在
  `reduce:true` 下照动——这正是窄修复清单的缺口形态，逐消费点验证，
  不逐 token 验证。
- golden 只做静态视觉回归（现有 golden_matrix 范围）；动效断言走
  controller/时长/终态，不用 `matchesGoldenFile` 做动效证据（D7）。
- 回归测试须对未修代码证明会失败；断言终态不断言轨迹
  （quality-guidelines 既有要求）。

## 四、逐契约回归 ledger（父 §5 + #48）

| 契约 | 回归方法 | 自动化 |
|---|---|---|
| §5.6 SnackBar 单通道 | grep 白名单闸口（owner + `showAppSnackBarOn` 两点）；扩展 `app_snack_bar_test` 断言分支根 margin + animationStyle；可固化 layering 风格静态测试 | 自动 |
| §5.6 触觉单来源 | `rg "HapticFeedback" lib/` 恰一 owner + 白名单调用点（W4/W6/W7）；owner 缺席 → blocked | 自动；手感人工/未验证 |
| §5.7 ActionTerminology | arb key 逐词对表 + 冻结文案 focused test + 改动页人工过一遍（记录文件清单） | 半自动 |
| §5.2 ObjectPresentation | `NovelCard|NovelRow|IllustCard` census，无 feature 本地卡片类 | 半自动；视觉等价人工 |
| §5.1 QueryContext + re-tap | 恢复等级声明逐页核对（PageStorageKey/restorationId/durable 路由值）；re-tap widget test：回顶、无刷新、无隐藏展开入口 | 大部自动；进程死亡恢复未验证 |
| §5.3/§5.4 | 抽查三处：反向搜图 cancel-vs-leave、profile edit dirty 确认、novel reader PopScope 顺序 | 半自动 |
| §5.5 overlays/breakpoints | `showDialog|showModalBottomSheet` 出 `app_overlays.dart` 为零；`AppBreakpoints`/`two_pane` 编译锁定 | 自动 |
| 无第二常量来源 | `Duration\(milliseconds` census 白名单；`Hero(` census 单一 tag 族；`FuncSemanticTokens.motion*` 按 D5 | 自动 |

## 五、§4.10 八项回归设计

| 条目 | 方法 | 自动化 |
|---|---|---|
| 同级分类点击/拖动一致 | `root_swipe_switcher_test`/`func_bottom_nav_test` 已有交接覆盖；W10 在默认+reduced 双注入下跑 ranking/new/recommended 的 tap 与 drag；grep 确认 `novel_ranking_page` 已收敛到 `TabSlideStack`/`BranchSlidePager`、无 index-swap 正文残留 | 大部自动；拖动跟手未验证 |
| 作者头部仅改布局 | W3 所有；W10 跑 `user_profile` 测试 + 展开/收起动作可达性人工扫（follow/share/more/stats） | 半自动 |
| 查询输入/结果连续 | `navigation_restoration_test` 式：search 输入→结果→back 恢复 keyword/filters/type；AppBar 可编辑上下文（post-W2） | 自动 |
| feed→detail→viewer 单一空间链 | `hero_transition_test` 覆盖 clip/flight；grep 各消费点命中 `illustHeroTag`/`FuncRouteTransition`/`DragToDismiss`；`Hero(` census 无第二 tag 族 | 自动 |
| 表单/键盘不动主控件 | shell `resizeToAvoidBottomInset:false`（home_page.dart:160）+ leaf `viewInsets` padding；widget test：sheet 内聚焦字段 → 主动作不平移 | 半自动；桌面键盘人工 |
| loading 不延迟内容 | `StaggeredEntrance` 曝光触发非挂载门控（既有测试）；grep `onTapDown` preload 未被 `onTap` push await；冷路由首帧经 `initialEntity` | 自动 |
| reduced motion 只去装饰 | 过闸 widget 三注入落终态（motion_test 模式扩展）；负向断言 Ugoira 播放、滚动物理、SnackBar 可见性仍工作；R2 窄修复落地 | 大部自动 |
| #48 + MotionTokens 单一来源 | §四 grep 闸口行 + `FuncSemanticTokens.motion*` D5 处理 | 自动 |

## 六、最终验收矩阵（父 §7 映射）

- **自动化层**：focused + 全量 `flutter test --no-pub`；`analyze --no-pub`；
  `dart format --output=none --set-exit-if-changed lib test`；
  `git diff --check`；`task.py validate`。
- **宽度**：320/390/600/840/1200 + 横屏，`tester.view.physicalSize`
  （golden_matrix 先例）断点行为；真实窗口 resize = 桌面人工（未证实）。
- **文本**：`textScaler` 1.0/1.3x 注入关键页；长翻译人工（可 ru locale pump
  抽查）。
- **输入**：触摸=widget test；系统返回=既有 `PopScope` 测试；
  Tab/Enter/Escape=`Focus`/`KeyEvent`（detail_image_pager 先例）；
  滚轮/拖拽=`PointerScrollEvent` + 桌面人工。
- **辅助技术**：`ensureSemantics`/`matchesSemantics` 语义树断言三优先页
  （作者头部、查看器、评论输入）；TalkBack/Narrator = **未验证**。
- **反馈**：触觉 owner 存在性 + 白名单 grep；on/off 可辨性若 W4 owner 暴露
  闸口则单测；手感未验证。
- **动效**：默认/reduced 三注入（§三表）全覆盖过闸 widget。
- **状态**：loading/content/refresh-error/load-more-error/empty/busy/
  failure-retry 复用各 leaf 与 feed-state 既有测试，W10 重跑聚焦文件，
  不重复造。
- **台账格式**（`acceptance-ledger.md`）：每行 = 验收项 | 契约来源 | 方法 |
  证据（测试 id/grep/人工记录）| 自动化 | 分级（implemented /
  unit-widget-tested / desktop-tested / device-tested / 未验证 / blocked）。
  未验证行写明缺失面；无 vibe-check 关闭。

## 七、复用与禁止

- 复用：`MotionTokens`/`MotionScope`/`resolve`/`enabled`、`AnimationStyle`、
  `showAppDialog`/`showAppBottomSheet`、`motion_test.dart` `_wrap` 注入模式、
  `func_bottom_nav_test` 真壳 pump 模式（createPixivRouter + ProviderScope
  overrides + memoryPreferences）、`layering_test` 静态扫描模式、
  `accessibilityFeaturesTestValue`、`ensureSemantics`/`matchesSemantics`。
- 禁止：新动效常量/曲线/组件、第二反馈通道、第二 `HapticFeedback` 调用族、
  新路由/provider、schema 变更、顺手重构高冲突区（路由/导航/主题/l10n）。

## 八、运行时证据缺口（一律标「未验证」）

TalkBack/Narrator 代表路径（无 Android 设备/Windows host，替代证据 =
语义树断言）、触觉手感与 on/off 体感（只验单来源）、手势跟手与拖动阈值
手感（只验终态正确性）、进程死亡恢复（需真杀进程，只核对声明）、真机
60fps/掉帧（FrameProbe 是人工工具，无帧时序断言）、`flutter run -d linux`
桌面运行（WSL 无显示服务，未证实）。
