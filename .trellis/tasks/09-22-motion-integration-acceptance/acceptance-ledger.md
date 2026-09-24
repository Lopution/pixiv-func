# W10 动效集成与最终验收 —— 验收台账

## 0. 方法、环境与分级约定

- 验收基线：`origin/main@5dbef81`（分支 merge-base，W1–W9 全部合入归档，9/9 completed；在途 PR #79/#82/#85 按 in-flight 处理，不阻塞本任务）。
- 执行环境：WSL2 Ubuntu 24.04、Flutter 3.47.2。可执行层 = `flutter test --no-pub`（widget/unit/golden）、`flutter analyze --no-pub`、`dart format`、静态 grep/layering 闸口。
- 分级取值：`implemented`（静态/结构确认）/ `unit-widget-tested` / `desktop-tested` / `device-tested` / `未验证` / `blocked`。
- 本环境不可用的执行面一律标 **未验证**，不由 widget 测试推断通过：Android/iOS 设备与模拟器、Windows 宿主、桌面 runtime（`flutter run -d linux` 未执行）、TalkBack/Narrator、真实性能采集、触觉体感、手势体感、进程死亡恢复。
- 在途 PR 相关行标 **in-flight（PR #n）**，不记绿也不阻塞。

## 1. 动效契约（design §4.10 / 动效闸口）

| 验收项 | 契约来源 | 方法 | 证据 | 自动化 | 分级 | 缺失面 |
|---|---|---|---|---|---|---|
| MotionTokens 单一时长来源 | design §5.6/§4.10 | 静态闸口 + 单元测试 | `test/architecture/feedback_channels_test.dart`；`test/motion_test.dart` | 全 | implemented | — |
| 平台 reduceMotion 接入 | design §4.10（iOS 洞修复） | 单元测试：accessibilityFeatures.reduceMotion 注入 | `test/motion_test.dart`；`lib/app/motion/motion_tokens.dart` `MotionTokens.enabled` | 全 | unit-widget-tested | 物理开关体感未验证 |
| in-app MotionScope 闸 | design §4.10 | 双注入矩阵（default/reduce） | `motion_test.dart`、`ranking_feed_test.dart`、`illust_detail_page_test.dart`、`novel_reader_chrome_test.dart`、`app_snack_bar_test.dart`、`shared_widgets_test.dart` | 全 | unit-widget-tested | 体感未验证 |
| reduced motion 只去装饰 | design §4.10 | 负断言：ugoira 播放/滚动物理/SnackBar 可见性在 reduce 下仍工作 | `ugoira_viewer_test.dart`、`motion_test.dart`、`app_snack_bar_test.dart` | 全 | unit-widget-tested | 体感未验证 |
| DragToDismiss 返回闸 | §4.10 窄修复 N2 | 单元测试：cancel/dragEnd 两调用点 reduce 下直接落位 | `motion_test.dart`；`lib/app/motion/drag_to_dismiss.dart` | 全 | unit-widget-tested | 手势体感未验证 |
| 详情分页器时长过闸 | §4.10 窄修复 N3 | 单元测试 | `illust_detail_pager_test.dart`；`detail_image_pager.dart` | 全 | unit-widget-tested | — |
| 小说阅读器 chrome/翻页闸 | §4.10 窄修复 N4 | widget 测试：reduce 下单帧落位 | `novel_reader_chrome_test.dart`（'reduced motion: chrome and page turns land in one frame'）；`novel_reader.dart`、`novel_reader_stage.dart` | 全 | unit-widget-tested | 体感未验证 |
| SnackBar 动画样式过闸 | design 二审撤销豁免 | widget 测试：normal=`appSnackBarAnimationStyle`，reduce=`AnimationStyle.noAnimation` | `app_snack_bar_test.dart`；`app_snack_bar.dart` | 全 | unit-widget-tested | — |
| 同级分类点击/拖动一致 | §4.10 | 双注入 widget 测试：tap 与 drag 落同一路由/状态；枚举路由值与 apiValue 分离断言 | `ranking_feed_test.dart`（'category tap and drag land identically' reduce:false/true）；`novel_ranking_page` 已收敛无 index-swap | 全 | unit-widget-tested | 拖拽体感未验证 |
| feed→detail→viewer 单一空间链 | §4.10 | widget 测试：共享 `illustHeroTag` 族、preload 先于 push、无第二 tag 族 | `hero_transition_test.dart`；C1 静态 census | 全 | unit-widget-tested | — |
| loading 动效不延迟内容 | §4.10 | 断言 entrance 为曝光触发；`onTapDown` preload 不被 `onTap` push await | `hero_transition_test.dart`、feed entrance 既有测试 | 全 | unit-widget-tested | — |
| TabBar 点击动画未过 in-app 闸 | §4.10 | 结构说明：framework `kTabScrollDuration`，in-app 设置无法在不换 tap 语义下折叠；平台 flag 覆盖 | research/implementation-draft.md §1 | 全 | implemented（已知限制已记录） | in-app 闸不覆盖该项（结构性） |

## 2. 导航 / QueryContext / re-tap（§5.1）

| 验收项 | 契约来源 | 方法 | 证据 | 自动化 | 分级 | 缺失面 |
|---|---|---|---|---|---|---|
| re-tap 当前 tab 回顶不刷新 | §5.1 + 非 Astra 契约 | widget 测试：ScrollController→0、无刷新调用、无隐藏展开 | `root_swipe_switcher_test.dart`、`new_content_feed_test.dart`、`ranking_feed_test.dart`、`recommended_home_test.dart` | 全 | unit-widget-tested | — |
| branch-root re-tap `goBranch` 同 index | §5.1 | widget 测试：同 index emit + 分支栈回根；异 index/程序化同步不 emit | `root_swipe_switcher_test.dart`；`branch_slide_stack.dart` `ReTapChannel` | 全 | unit-widget-tested | — |
| re-tap 消费者覆盖 | §5.1 | census | new/search/recommended/ranking 四页订阅 `reTapEvents` | 全 | implemented | **缺口**：宽布局 `NavigationRail`（`home_page.dart:169` 直调 `goBranch`）不经过 re-tap 通道；settings 页无订阅 |
| durable 路由值恢复等级 | §5.1 + Route Restoration Contract | widget 测试：recommended type、new scope+type、bookmark tag feed、feed offset 的 replace/restore | `navigation_restoration_test.dart`（4 条） | 全 | unit-widget-tested | **进程死亡恢复未验证**（无设备/模拟器） |
| 共享 PrimaryScrollController 副作用 | §5.1 已知项 | 结构核对 | `user_page.dart:549` `scrollToTop` 走 `PrimaryScrollController.maybeOf`——keep-alive 全 tab 内层滚动归零（D2 机制固有） | 全 | **未解决 · in-flight**（PR #79 触碰同文件） | 精确到单 tab 需结构改动，未在 W10 范围 |

## 3. 覆盖层与反馈通道（§5.5/§5.6 + PR #48）

| 验收项 | 契约来源 | 方法 | 证据 | 自动化 | 分级 | 缺失面 |
|---|---|---|---|---|---|---|
| SnackBar 单通道 | §5.6 | 静态闸口：owner 恰一 + 白名单调用点 | `feedback_channels_test.dart`；`app_snack_bar.dart` | 全 | implemented | — |
| SnackBar 分支根 margin/样式 | §5.6 | widget 测试：分支根抬升、共享 shape、action、reduce 样式、宽布局 | `app_snack_bar_test.dart` | 全 | unit-widget-tested | — |
| HapticFeedback 单来源 | §5.6 + 非 Astra 契约 | 静态闸口：恰一 owner + W4/W6/W7 白名单 | `feedback_channels_test.dart`；`app_haptics.dart` | 全 | implemented | **触觉体感未验证**（无设备） |
| 触觉分级调用点 | §5.6 | 单元测试：下载/保存、管理模式、危险确认、发送终态级别 | `comments_replies_test.dart`（send/stamp/reply success mediumImpact、failure 无触觉）等 | 全 | unit-widget-tested | 体感未验证 |
| 模态覆盖层单一入口 | §5.5 | 静态闸口：`showDialog|showModalBottomSheet` 出 `app_overlays.dart` 为零 | `feedback_channels_test.dart`；`local_novels_page.dart:148/173` 已迁移（N6 核销） | 全 | implemented | — |
| Hero tag 单族 | §5.6/§4.10 | 静态 census | `feedback_channels_test.dart` | 全 | implemented | — |
| 合并冲突标记 | 流程闸口 | 静态扫描 `^(<<<<<<<|=======|>>>>>>> )` | `feedback_channels_test.dart` 内嵌扫描 lib/test/.trellis | 全 | implemented | — |

## 4. 表单 / 返回 / 术语（§5.3 / §5.4 / §5.7）

| 验收项 | 契约来源 | 方法 | 证据 | 自动化 | 分级 | 缺失面 |
|---|---|---|---|---|---|---|
| 键盘展开不位移主控件 | §5.3/§4.10 | widget 测试：focus+viewInsets 下 AppBar/feed 不平移、composer 内部抬升 300、send 落键盘顶之上 | `comments_replies_test.dart`（'keyboard open pads only the composer'）；shell `resizeToAvoidBottomInset:false`（home_page/comments_page/search_page） | 全 | unit-widget-tested | 物理键盘/IME 未验证 |
| 键盘/面板互斥与几何 | §5.3 | 既有 widget 测试：互斥、system back、键盘高度采样、面板 fallback、连续切换 | `comments_replies_test.dart` | 全 | unit-widget-tested | — |
| 反向搜图 cancel-vs-leave | §5.4 抽查（W1） | 既有 widget 测试：进度取消停留页面；app-bar back 取消并 pop | `reverse_image_search_page_test.dart`（L410/L437） | 全 | unit-widget-tested | — |
| 资料编辑 dirty 确认 | §5.4 抽查（W3） | 既有 widget 测试：dirty 确认、clean 直接离开、discard 后 pop | `profile_edit_test.dart`（L418/L504/L632） | 全 | unit-widget-tested | — |
| 小说阅读器 PopScope 顺序 | §5.4 抽查（W5） | widget 测试三腿：chrome 可见 back 先关 chrome；显式 back 离页；chrome 隐藏 system back 离页（本分支补第三腿） | `novel_reader_chrome_test.dart`（L128/L176/新增）；`novel_reader_stage.dart:193` `canPop:!_chromeVisible` | 全 | unit-widget-tested | — |
| §5.7 动作术语表 | §5.7 | arb 逐词核对 + 冻结文案 focused test + 改动页文案人工过 | `action_terminology_test.dart`（13 断言）；人工核对清单见 §8 | 全 | implemented | 见 §8 偏离记录 |
| 收藏 `_isDirty` 含 tagInput 残文 | W3 已知项 → 阶段 2 吸收 | 单元测试：残文即脏 | `bookmark_switch_button_test.dart`；`bookmark_switch_button.dart` `_isDirty` | 全 | unit-widget-tested | — |
| 收藏 sheet submitting 可关 | W3 已知项 | 结构核对 | `_closableFreely` 注释声明为有意取舍 | 全 | implemented（有意取舍已确认） | — |
| 小说偏好读取失败错误态 | W9 已知项 → 阶段 2 吸收 | widget 测试：load 失败 → FeedError + retry | `novel_reader_chrome_test.dart`/`novel_reader_settings_test.dart`；`novel_reader.dart` `_loadPrefs` catch | 全 | unit-widget-tested | — |

## 5. 无障碍（§7 屏幕阅读器代表路径）

| 验收项 | 契约来源 | 方法 | 证据 | 自动化 | 分级 | 缺失面 |
|---|---|---|---|---|---|---|
| 作者头部语义 | §7 优先页 | 语义树断言：stat chip 单节点 'label, value'、button+tap action、ExcludeSemantics 防双播 | `priority_surface_semantics_test.dart`；`profile_header_delegate.dart:51-60` | 全 | unit-widget-tested | **TalkBack/Narrator 未验证** |
| 查看器语义 | §7 优先页 | 语义树断言：每页 image 节点 '第 n 页，共 N 页'、chrome 控件命名按钮 | `priority_surface_semantics_test.dart`；`image_viewer_page.dart:663-665` | 全 | unit-widget-tested | **TalkBack/Narrator 未验证**；真实焦点顺序未验证 |
| 评论输入语义 | §7 优先页 | 语义树断言：回复上下文单播报单元、EditableText isTextField、send/emoji/stamp 命名按钮 | `priority_surface_semantics_test.dart`；`comment_input.dart:132/267-307` | 全 | unit-widget-tested | **TalkBack/Narrator 未验证** |
| 共享组件语义 | §7 | feed 状态件 label+button+loadingSpinner role | `shared_component_semantics_test.dart` | 全 | unit-widget-tested | — |

## 6. 运行时矩阵（§7 可执行层）

| 验收项 | 契约来源 | 方法 | 证据 | 自动化 | 分级 | 缺失面 |
|---|---|---|---|---|---|---|
| 宽度 320/390/600/840/1200+横屏 | §7 | 断点单测 + 各页 physicalSize widget 测试 | `responsive_layout_test.dart`（illustColumnsFor/useNavigationRail 矩阵）；`login_page_test`/`onboarding_pages_test`/`backup_test`/`profile_edit_test`/`download_tasks_page_test`/`home_page_test`/`app_snack_bar_test`/`comments_replies_test`（grid 320/390/840） | 全 | unit-widget-tested | 真实窗口 resize 未验证；**新发现缺口**：`comment_item.dart` actions `Row`（L260）在 390dp 溢出 49px，无既有窄宽覆盖 |
| textScaler 1.0/1.3x | §7 | textScaler 注入 widget 测试 | `scrollable_form_shell_test.dart`、`login_page_test`、`onboarding_pages_test`、`backup_test`、`shared_widgets_test`、`updater_about_test` | 全 | unit-widget-tested | 系统大字号真机未验证；长翻译人工 pass 未验证（ru 长文案行未逐页过） |
| 键盘模拟 | §7 | viewInsets 注入 + KeyEvent 测试 | `comments_replies_test.dart`（C7）、`profile_edit_test.dart:626`、`detail_image_pager_test`（方向键） | 全 | unit-widget-tested | 物理键盘 Tab/Enter/Escape 未验证 |
| 鼠标滚轮 | §7 | PointerScrollEvent 模拟 | `feed_prefetch_cursor_test`、`illust_detail_page_test`、`motion_test`、`navigation_restoration_test` 等 | 全 | unit-widget-tested | 真实滚轮手感未验证 |
| 七态（loading/content/refresh-error/load-more-error/empty/busy/failure-retry） | §7 | feed 状态测试 + golden | `shared_component_semantics_test`、各 feed 测试、`golden_matrix_test`（light/dark empty/error/loading） | 全 | unit-widget-tested | — |
| reduced-motion 三注入 | §7 | default / MotionScope / accessibilityFeatures 三注入 | `motion_test.dart` + 各 gated consumer 测试 | 全 | unit-widget-tested | 体感未验证 |
| Android/iOS 设备 | §7 | 本环境无设备/模拟器 | — | 无 | **未验证** | 全部设备面 |
| Windows 宿主 / 桌面 runtime | §7 | 本环境无 Windows；`flutter run -d linux` 未执行 | — | 无 | **未验证** | 桌面运行时、窗口 resize、系统集成 |
| 真实性能 | §7 | 无 profile 设备 | `frame_probe` 设施存在（W8） | 无 | **未验证** | 帧耗时、滚动 jank |
| 触觉体感 | §7 | 无设备 | 通道/分级已测 | 无 | **未验证** | 振动强度/时机体感 |
| 手势体感 | §7 | 无设备 | 行为已测 | 无 | **未验证** | 拖拽/回弹手感 |
| 进程死亡恢复 | §5.1 声明 | 无设备/模拟器可杀进程 | 声明逐页核对（C4） | 无 | **未验证** | keyword 草稿经 `q` 参数存活已声明；cursor/selection 不存活为已知限制 |

## 7. W10 阶段 2 已吸收修复（implemented）

| 修复 | 来源 | 证据 |
|---|---|---|
| `MotionTokens.enabled` 并入平台 `accessibilityFeatures.reduceMotion` | iOS 洞 | `be432a7` |
| DragToDismiss 取消/结束返回动画过闸 | N2 | `8777e5a` |
| 详情分页器时长/曲线走 MotionTokens | N3 | `bbaf5ac` |
| 小说阅读器 chrome+翻页过闸 | N4 | `3c651ad` |
| `FuncSemanticTokens.motion*` 未用别名删除 | N7 | `adc518d` |
| SnackBar 动画样式过闸（豁免撤销） | design 二审 | `88ab77a` |
| 收藏 `_isDirty` 并入 tagInput 残文 | W3 已知项 | `4fde9b8` |
| 小说偏好读取失败落错误态+重试 | W9 已知项 | `bf1c439` |

## 8. 有意取舍与已知偏离（保留可见，不记绿）

| 项 | 位置 | 性质 |
|---|---|---|
| `searchCancel`（'取消'）标注纯导航 back | `search_page.dart:538` → `Navigator.pop` | §5.7 偏离；W2 codebase-search.md:22 已记录、leaf 接受未改；`action_terminology_test.dart` 固定现文案防悄悄改义 |
| `watchlistRemove` key 名 Remove vs 文案「取消追更」 | arb | 命名不一致但语义为条目级 undo，不违后果分级 |
| 宽布局 NavigationRail 绕过 re-tap | `home_page.dart:169` 直调 `goBranch`，不经 `BranchSlideStack` re-tap 通道 | 结构缺口，未在 W10 窄修范围 |
| settings 页无 re-tap 订阅 | `settings_page.dart` | re-tap 契约仅四 feed 页消费；settings 无滚动回顶入口 |
| 小说设置保存失败保留内存新值 | `novel_reader_stage.dart:164-179` `_applySettings` 先 setState 后 save | 有意语义：失败可见（snackbar）不回滚；记录为已确认取舍 |
| 进度 sheet 百分比公式与 footer 不同 | sheet `preview/(pageCount-1)*100`（L461-463）vs footer `(page+1)/pageCount*100`（L190/364/404） | 公式差异：sheet 为拖块位置比例、footer 为已读比例；记录为已知差异 |
| `_persistAnchor` 失败静默 | `novel_reader_stage.dart:182-184` `unawaited(progress.save)` 无 catch | 锚点持久化失败无任何信号；记录为已知缺口 |
| 收藏 sheet submitting 期间可关闭 | `_closableFreely` | 有意取舍（W3 注释声明） |
| 共享 PrimaryScrollController 全 tab 归零 | `user_page.dart:549` | W3 NOTED，D2 机制固有；PR #79 在途 |
| comment_item actions 390dp 溢出 49px | `comment_item.dart:260` `Row` | 本任务窄宽核对新发现；未修（记录待后续 leaf） |
| W8 翻译摘要重探/相册名草稿守卫 | settings 页 | **in-flight（PR #82）** |
| W3 系列统计口径/门禁 | `user_page.dart` | **in-flight（PR #79）** |

## 9. 未验证面总清单（同步 PR body）

1. Android/iOS 设备与模拟器：全部设备面未验证。
2. Windows 宿主与桌面 runtime（`flutter run -d linux` 未执行）：窗口 resize、系统集成未验证。
3. TalkBack/Narrator 代表路径：语义树断言已过，真实 AT 走查未验证。
4. 真实性能（帧耗时/滚动 jank）：未验证。
5. 触觉体感（强度/时机）：未验证。
6. 手势体感（拖拽/回弹手感）：未验证。
7. 进程死亡恢复：声明已核对，实际杀进程恢复未验证。
8. 物理键盘 Tab/Enter/Escape、物理滚轮、IME 实际行为：未验证。
9. 长翻译人工 pass（ru 等长文案逐页过）：未验证。
10. 视觉等价人工复核（卡片/弹层视觉角色收敛）：未验证。
