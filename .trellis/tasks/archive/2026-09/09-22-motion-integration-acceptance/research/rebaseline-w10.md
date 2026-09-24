# W10 Rebaseline：W1–W9 合入核对与行号重锚

执行基线：`main@5dbef81`（2026-09-24 复核；分支 `task/09-22-motion-integration-acceptance`
merge-base = 该 SHA，无需 rebase）。规划基线 `main@8067b2d` 的全部 file:line 引用
已逐项对当前 HEAD 重新核实，结论如下。

## 1. W1–W9 合入与归档状态

| Leaf | 状态 | 合入 PR | 归档 |
|---|---|---|---|
| W1 interaction-outcome-correctness | completed | #56（归档 #57） | archive/2026-09 |
| W2 discovery-query-context | completed | #59（check 修复 #78 已合入） | archive/2026-09 |
| W3 profile-bookmark-continuity | completed | #62 | archive/2026-09 |
| W4 artwork-viewer-series-flow | completed | #60（check 修复 #81 已合入） | archive/2026-09 |
| W5 novel-reader-parity | completed | #66（check 修复 #74 已合入） | archive/2026-09 |
| W6 entity-management-consistency | completed | #70 objects / #83 management / #84 downloads（check 修复 #73 已合入） | archive/2026-09 |
| W7 comment-input-state | completed | #65（check 修复 #75 已合入） | archive/2026-09 |
| W8 settings-form-semantics | completed | #63/#69/#71/#72/#80 s1–s5 | archive/2026-09 |
| W9 onboarding-auth-content-layout | completed | #61/#64/#67/#68（check 修复 #77 已合入） | archive/2026-09 |

9/9 leaf `status=completed` 且目录已归档。无 blocked 验收行来自「leaf 未合入」。

## 2. 在途 PR（in-flight，不阻塞 W10）

| PR | 内容 | 与 W10 的交互 |
|---|---|---|
| #79 OPEN | W3 复核：系列统计口径 + 门禁回归（`user_page.dart`、`test/user_profile_test.dart`） | 触碰 user_page.dart——W10 不在该文件做窄修复，只在台账记录其 re-tap 项；无冲突 |
| #82 OPEN | W8 复核：翻译摘要重探/相册名草稿守卫（settings 页 + frame_probe） | 与 W10 窄修复文件零重叠 |
| #85 OPEN | chore：arm64-v8a APK 尺寸上限重定基线 | CI 基线，与代码无关 |

对应验收行标 **in-flight**：W3 系列统计口径、W8 翻译摘要/相册名守卫两条在台账
记「in-flight（PR #79/#82）」，不记绿也不 blocked 整个 W10。

## 3. research file:line 引用复核（motion-inventory §5/§6/§4）

### §5 偏差清单逐项核销

| 原引用（main@8067b2d） | 当前状态 | 处置 |
|---|---|---|
| `detail_image_pager.dart:65-69` 硬编码 180ms+easeOut | **仍在**（:72-76，`_onKeyEvent` 内 `animateToPage`） | N3 有效 |
| `novel_page.dart:670-728` `_ChromeBar` 未过闸 | **W5 已重构**：`_ChromeBar` 移入 `novel_reader_stage.dart:665-739`，`didUpdateWidget` L689-698 `forward()/reverse()` 仍未过闸 | N4 有效（新位置） |
| `novel_reader.dart:279/363` `animateToPage(MotionTokens.fast)` | **仍在**：L338-342（`handle.goToPage`）、L446-450（tap-zone 翻页）未过闸 | N4 有效（行号刷新） |
| `new_page.dart:134` `AnimatedSize`/`_selectorExpanded` | **已吸收**：W2 删除 `_selectorExpanded`/`_onTabTap` 展开路径，`_NewTypeSelector` 常驻 ChoiceChip；全文无 `AnimatedSize`/`_selectorExpanded` 匹配 | N5 核销 |
| `local_novels_page.dart:112` 裸 `showDialog<bool>` | **已吸收**：W6 迁移为 `showAppBottomSheet`（:148）+ `showAppDialog`（:173）；lib/ 内 `showDialog|showModalBottomSheet` 出 `app_overlays.dart` 为 **零命中** | N6 核销 |
| `drag_to_dismiss.dart:43` `_returnAnimation` 未过闸 | **仍在**（L42-49 controller 创建处不变）；调用点增至两处：`_onVerticalDragEnd` L82、`_onVerticalDragCancel` L87 | N2 有效（两处调用点同法过闸） |
| `app_snack_bar.dart:90` `appSnackBarAnimationStyle` 无条件 | **仍在** L90 | design.md 二审已撤销豁免（见 §6 决策冲突裁决） |
| `motion_tokens.dart:82-86` `enabled` 不读 iOS `reduceMotion` | **仍在**：L82-86 仅 `MediaQuery.disableAnimations` OR `MotionScope.reduce` | N1 有效 |
| `frame_probe_page.dart:44` 500ms 轮询 | 仍在 :45 | 数据轮询，非动效，维持排除 |
| `ugoira_viewer.dart:399` `frame.delayMs` | 仍在 :390 | 内容行为，有意不闸 |
| `home_page.dart:164-180` NavigationRail 展开过渡 | 仍在 :164-180 区域 | 框架自有，D3 已知限制 |

### §6 re-tap 基线复核

- 旧基线「无实现」已被 W2/W3 落地：`ReTapChannel`（branch_slide_stack.dart:254）
  + `reTapScrollToTop`（:274）+ `selectIndex` 同槽分支（:142-163，goBranch→popUntil→emit）。
- 消费者（6 文件）：`recommended_home_page`、`new_page`、`search_page`、
  `ranking_page`（含 PR #78 补的 branch 级订阅）、`novel_ranking_page`、
  `branch_slide_stack` owner；页内同索引 tab/chip tap 亦走 `reTapScrollToTop`。
- **缺口（台账待决策）**：宽屏 `NavigationRail.onDestinationSelected` 直连
  `navigationShell.goBranch`（home_page.dart:166）——pager 是后代非祖先，
  绕过 re-tap 通道，宽屏同槽点按不弹栈不回顶（W2 check 报告级项，修需重构
  通道暴露）。`SettingsPage` 未订阅 reTapEvents（观察项）。
- **缺口（台账待决策）**：`user_page.dart:549` `scrollToTop` 经共享
  `PrimaryScrollController.animateTo(0)`——keep-alive 全部 tab 的内层滚动
  归零（W3 check NOTED，D2 机制固有副作用；PR #79 正触碰该文件）。

### §4 反馈通道复核

- SnackBar：`SnackBar(`/`showSnackBar(` 仅 `app_snack_bar.dart` 内部；
  `showAppSnackBarOn` 两个 messenger 直连点 `app.dart:88`（更新提示）、
  `home_page.dart:111`（退出提示 `..hideCurrentSnackBar()` 链式）合法。
  `ScaffoldMessenger` 其余出现均为注释/messenger 装配（func_bottom_nav:907
  BranchRootScaffold、app.dart:42 `_messengerKey`）。
- 触觉：owner `lib/app/haptics/app_haptics.dart` 存在（W4 建立）；`HapticFeedback.`
  直接调用在 owner 外 **零命中**。消费点：comments_page、download_tasks_page、
  info_block、ugoira_viewer、illust_detail_page、image_viewer_page、
  local_novels_page、history_page、illust_card_actions、app.dart（configure）。
- overlays：`showDialog|showModalBottomSheet` 出 `app_overlays.dart` 零命中。
- Hero：4 个 `Hero(` 调用点（illust_card:257、page_image:162、
  ugoira_viewer:302、image_viewer_page:696）全部使用 `illustHeroTag` 单一
  tag 族 + `illustHeroFlightShuttleBuilder`，无第二 tag 族。
- `Duration(milliseconds` 出 `motion_tokens.dart` census：frame_probe 500ms
  轮询、detail_image_pager 180ms（N3 修复对象）、ugoira frame.delayMs、
  smooth_wheel_scroll 10ms 下限、app_haptics 节流间隔 50/80/120ms（W4 新增，
  非动效时长）、download_manager 200ms 节流、download_recovery retryAfterMs、
  search_autocomplete 260ms 防抖、ugoira_scheduler 16ms tick、
  ugoira_limits 60000ms 上限、history_models unsubmittedDuration——
  除 N3 外均非动效时长。
- `FuncSemanticTokens.motion{Short,Standard,Emphasized}`
  （func_semantic_tokens.dart:55-57,103-105,138-140,171-173,195-197,232-234）：
  声明文件外 **零消费者** → N7 有效（按 D5 删除三字段）。
- `illust_card.dart:196-204` `onTapDown` 预载 fire-and-forget，`onTap` 的
  `openIllust` push 未 await 预载——preload 契约成立。

## 4. 基线聚焦测试（真实结果）

`flutter test --no-pub` 八文件全绿，共 67 例（本机一次性通过，无 loopback
噪声出现）：

- motion_test / func_bottom_nav_test / root_swipe_switcher_test /
  hero_transition_test / app_snack_bar_test / navigation_restoration_test /
  novel_reader_chrome_test / shared_component_semantics_test

环境补齐：worktree 新建后需先 `flutter pub get`（已执行，依赖按 lockfile）。

## 5. 条件项现状结论

- `_selectorExpanded`/`AnimatedSize`：**W2 已移除** → N5 核销「已吸收」。
- `local_novels_page` 裸 `showDialog`：**W6 已迁移** → N6 核销「已吸收」。
- `novel_page`/`novel_reader` 闸口点：**W5 已重构**——chrome 迁入
  `novel_reader_stage.dart`，翻页在 `novel_reader.dart` 两处；未过闸事实不变 →
  N4 对新位置执行。
- `FuncSemanticTokens.motion*`：**仍零消费者** → N7 执行删除。
- re-tap→top：**已落地**（branch 级 + 页内同索引），两条已知缺口记台账待决策。

## 6. 决策冲突裁决：appSnackBarAnimationStyle

- prd.md D2（44a067c 初始规划）：保持不闸。
- design.md 决策区（4486ea4 二审修订，**后于** prd）：「豁免撤销（评审修正）…
  in-app reduced-motion 闸须接入」。
- 裁决：以**后修订的 design.md** 为准——豁免已撤销，按闸口级窄修复并入阶段 2
  （`AnimationStyle.noAnimation`，与 app_overlays 同法；消息/按钮/时长不动）。

## 7. Leaf check 遗留项处置映射

| 项 | 现状复核 | 处置 |
|---|---|---|
| W3 re-tap 共享 PrimaryScrollController 全 tab 归零 | user_page.dart:549 仍在 | 台账待决策（D2 机制固有；精确定位需结构改动；PR #79 在途触碰同文件） |
| W3 `_isDirty` 不含 tagInput 残文 | bookmark_switch_button.dart:202 仍在 | **阶段 2 小修**：`_isDirty` 并入 `_tagInput.text.trim().isNotEmpty`（一行，与 `_confirm` L236-237 提交时并入残文的语义对齐） |
| W3 sheet submitting 期间可关闭 | `_closableFreely` L208 注释声明有意 | 台账记「有意取舍已确认」 |
| W2 宽屏 NavigationRail 绕过 re-tap | home_page.dart:166 仍在 | 台账待决策（通道暴露需结构调整） |
| W2 SettingsPage 未订阅 re-tap | settings 无 ReTapChannel | 台账待决策 |
| W5 阅读设置写失败无内存回滚 | novel_reader_stage.dart:163-171 | 台账记「有意取舍已确认」（leaf implement.md 有改判记录） |
| W5 进度 sheet % 口径不一致 | sheet `preview/(count-1)` L440 vs 底栏 `(page+1)/count` L183,342 | 台账待决策（建议对齐 `(page+1)/count`，口径选择属产品决策） |
| W5 `_loadPrefs` 失败永久 loading | L138-147 无 catch | **阶段 2 小修**：catch → `FeedError`+retry（复用本文件既有错误契约，不改变成功路径） |
| W5 `_persistAnchor` 写失败静默 | L173-175 | 台账待决策（进度写入为 best-effort，反馈策略待定） |
| W4 check 四项（保存订阅/双击阈值/chrome 泄漏/语义标签） | PR #81 已合入 | 已吸收 |
| CI `git diff --check` 不检冲突标记 | W5 index.md 曾漏网 | **阶段 3 C1 纳入**：静态闸口增加 `<<<<<<<`/`>>>>>>>` 冲突标记扫描 |

## 8. 范围变化判断

窄修复面：N1/N2/N3/N4/N7 有效（N4 落点换至 novel_reader_stage/novel_reader 新位置），
N5/N6 已吸收；另加 design.md 二审裁决的 snackbar 动画闸口与两项 check 遗留小修
（bookmark `_isDirty`、`_loadPrefs` 错误态）。全部为最小 diff 闸口级/错误处理级
改动，无结构变更——**范围无实质变化，不需回到规划审阅**。
