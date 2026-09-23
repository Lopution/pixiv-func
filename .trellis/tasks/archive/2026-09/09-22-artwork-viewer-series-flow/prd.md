# 作品浏览、查看器与系列流程（Roadmap W4）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图，
§4.4 工作包范围、§4.11 所有权冻结、§5 跨包契约、§7 验收矩阵）。
规划基线：`main@8067b2d`（PR #54 合入后）。全部代码断言由本 leaf
`research/codebase-*.md` 在当前 HEAD 逐条复核；行号以 research 为准。

## Goal

让作品详情、全屏查看器、动图与系列页达到父契约要求的信息可达、显式模式、
直接菜单与一致动作语义；并以「下载/保存」为首个真实消费者建立 §5.6 唯一
触觉 owner。本 leaf 不改详情页排版基线（PR #50 保留）、不建持久化章节进度
schema、不预建 W6/W7 才会消费的能力扩展。

## 依赖与启动门禁

本 leaf 必须等 **W1 interaction-outcome-correctness** 合入 `main` 后
rebaseline（记录新 SHA、复核 owning files 行号、跑聚焦测试记录真实基线），
再启动第一个 stage。W1 不改本包 owning files，但 PopScope/显式返回契约
（`pop()` 命令式 vs `maybePop` 征询式）是本包查看器返回优先级的既有结论，
直接消费不重新论证。

## 追踪条目归属（自 `research/astra-review-traceability.md` §4/§5 复制）

| Astra | 条目 | 复核状态 | 本 leaf 落点 |
|---|---|---|---|
| 12 | 多页详情信息 | 部分（#50 改排版未改位置） | R1 |
| 13 | 下载/屏蔽隐藏模式 | 确认 | R2 + R3 |
| 14 | 全屏查看器 | 确认 | R4 |
| 15 | 动图操作 | 确认 | R5 |
| 16 | 系列目录 | 确认 | R6 |
| — | 触觉反馈分级（非 Astra 增补） | 确认（lib/ 零命中） | R7 |
| — | 屏幕阅读器代表路径（非 Astra 增补） | 确认 | 贯穿 R1–R5 |

## Requirements

### R1. 窄屏详情信息可达：常驻 compact header + 信息/评论直达锚点

现状：`illust_detail_page.dart` 窄屏分支把全部 imageSlivers 排在
metaSlivers 之前（L422-425），标题/作者/页数/标签/评论入口全部在
InfoBlock（L354-358），长多页作品需滑过全部图片才到；无任何跳转或常驻
信息入口（codebase-detail.md §1 关键缺口 1）。双栏分支（≥1200，
`AppBreakpoints.useTwoPaneDetail` L403）右栏信息面常驻，天然满足等价。

- 窄屏在 AppBar 之下挂**常驻 compact header**（ pinned 一行，不因滚动消失）：
  标题一行 ellipsize + 作者 + `n/共N页` + 「信息」跳转动作。
- 「信息」动作 → `Scrollable.ensureVisible(InfoBlock 锚点 context)` 滚动直达
  （用目标 context，不接管 `SmoothWheelScroll`/`PrimaryScrollController` 的
  controller 所有权——risks.md R9）。
- compact header **不复制** AppBar 动作：分享/收藏/下载全部留在常显 AppBar
  （即「核心动作在首图附近可见」由 AppBar 满足，header 是信息面+跳转面），
  消解 risks.md R5 的双份状态同步面。
- 双栏布局不出现该 header（右栏即等价信息面）；InfoBlock 保持 PR #50
  完整布局不变。
- Given 窄屏打开多页作品，When 滚动任意深度，Then 标题/作者/页数持续可见，
  且一次点击直达 InfoBlock；双栏下 header 不渲染。

### R2. 普通下载可见入口 + 批量页选择显式模式

现状：「下载全部」AppBar 动作仅 `_downloadMode` 为真时出现（L167-185）——
普通下载反而藏在批量模式后面；长按图片/Ugoira/双栏 pager 经
`_toggleDownloadMode`（L90，4 处复用）进入隐式模式：无模式标题、无已选
计数、无完成/取消，退出靠点空白（全屏作品无可见空白，risks.md R4）。
模式内点页角标 `_DownloadBadge` 立即下载该页（page_image.dart L232-248）。

- 「下载全部」改为 AppBar **常显动作**（解除 `_downloadMode` 门控）——
  普通下载的可见入口。
- 长按进入**显式选择模式**：底部 chrome 显示「已选 n / 共 N · 全选 ·
  完成 · 取消」；模式内点图片/角标 = 切换该页选中态（不再立即下载）；
  「完成」= 循环 `download(entity, index)` 提交所选页（协调器层
  dedupe/retry 安全）后退出模式；「取消」/点空白/系统返回 = 退出模式，
  不触碰进行中任务（任务归 DownloadManager，模式只是 UI 选择层）。
- 模式内已选 n=0 时「完成」禁用（语义化 disabled）。
- 进行中页的角标仍显示下载态（`stateFor` 语义保留，优先级：spinner >
  选中 > 未选中）。
- Ugoira 作品无页概念：长按不进入页选择模式；导出 GIF 改为常显 overlay
  动作（见 R5）。
- Given 详情页长按图片，When 进入模式，Then 底栏显示计数与完成/取消；
  选中 2 页点完成后两个任务入队、模式退出、既有任务不受影响。

### R3. tag 长按收敛为直接菜单

现状：`TagChip.onLongPress` = 切换 tag 屏蔽模式（tag_chips.dart L42-44 +
info_block.dart L142），搜索/复制/屏蔽分散在 tap/模式两态，无统一菜单，
无复制入口。

- tag 长按 → `showAppBottomSheet` 动作菜单：「搜索该 tag / 复制 tag 名 /
  屏蔽该 tag（已屏蔽时为解除屏蔽）/ 进入批量屏蔽模式」。
- 「屏蔽该 tag」直接调 `muteStoreProvider.notifier.toggleTag(name)`
  （单条屏蔽不经模式）；「进入批量屏蔽模式」= 现有 `_blockMode` 翻转
  （模式内 tap=切屏蔽态的行为不变——唯一批量屏蔽路径保留）。
- 复制成功 → `showAppSnackBar` + `AppHaptics.selectionClick()`（R7 触点）。
- PixEz 式「收藏 tag」**不做**（无对应 store）。
- 菜单挂在详情侧调用点（InfoBlock 的 onLongPress），不改 `TagChip` 共享
  组件签名——novel 侧 tag 行为不在本包（W5/W1 页面）。
- Given tag 长按，When 菜单弹出，Then 四个动作各自后果可断言；屏蔽/解除
  写 `MuteStore.tags`；复制落剪贴板并有反馈。

### R4. 查看器完整 chrome / 缩放 / 页码 / 动作 / 空态 / 键鼠 / 返回优先级

现状：`image_viewer_page.dart` 常驻 AppBar（title=`n/total`）+ PageView +
`InteractiveViewer`（0.9–6.0）；无 chrome 显隐、无双击缩放、无复位动作、
页码不可点、无保存/分享/信息动作、空列表渲染 `1 / 0`、无键鼠等价、返回
直接退路由（codebase-viewer.md §3 差距表）。已有资产：`DragToDismiss`
（非放大态下拉退路由）、逐页 `TransformationController`、±1 预取、Hero。

- chrome 显隐：默认可见；点击媒体区切换 + 底栏全屏钮 + 键盘 `F` 三通道；
  隐藏 = AppBar + 底部工具栏 + 移动端系统栏（`SystemUiMode.immersiveSticky`）；
  淡入淡出用 `MotionTokens.fast`；隐藏态下缩放/翻页手势仍可用（chrome 只是
  视觉层）。
- 双击缩放循环：fit(1.0) → 2.5 → fit，以双击点为焦点，`MotionTokens.fast`
  动画；单击/双击冲突由**同一 GestureDetector 双注册**让框架手势竞技场消歧
  （tap 延迟 ~kDoubleTapTimeout），不自造计时器（risks.md R1）。
- 「适应屏幕」显式动作：底栏 `Icons.fit_screen` 复位当前页至 identity；
  键盘 `0`。
- 页码 `n / total` 可点击 → `showAppBottomSheet` 页码选择（网格或滑条）→
  `PageController.jumpToPage`；跳页后 `replaceImageViewerPage` 链路照旧。
- 动作（对齐详情页同名同义）：保存当前页（`download(entity, page)` 复用
  `stateFor` 判定与 snackbar 语义）、分享（`ShareService`）、信息（
  `Icons.info_outline` → **bottom sheet** 显示标题/作者/日期/ID + 「保存
  全部」+「打开详情页」入口——查看器不回详情即可读元信息）。
- 实体注入：`ImageViewerPage` 增加 `IllustEntity? entity` 参数，
  `routes.dart` `_ImageViewerRoute` 把已解析实体透传（store 优先、extra
  快照兜底，不建并行数据通道——risks.md R10）；entity 为 null（深链冷启
  store miss）时保存/分享/信息动作不渲染。
- 空态：`_pageCount == 0` 时不渲染 `n / total` 计数位（现有 `1 / 0` 误导
  页码移除），body 保留 `viewerNoImages` 占位文案，保存/分享/跳页动作
  语义化禁用（可见但 disabled，读屏可感知）。
- 键鼠等价：`←`/`→`/`J`/`K` 翻页、`+`/`-`/`0` 缩放/复位、`F` 全屏、
  `Esc`/`Backspace` 返回、`S` 保存当前页、`I` 信息；滚轮 = 以指针位置为
  焦点缩放，Shift+滚轮翻页；载体 `CallbackShortcuts` + `Focus(autofocus)`
  + `Listener(onPointerSignal)`（项目无 shortcut 基建，最小形态）。
- 返回优先级（PopScope，消费 W1 契约）：系统返回 `canPop = !zoomed &&
  chromeVisible`，拦截回调里 zoomed → 复位当前页缩放、chrome 隐藏 →
  恢复 chrome，否则放行；显式 AppBar back 与 `DragToDismiss` 保持命令式
  `pop()` 直接离页（绕行语义与 W1 一致）。`replaceImageViewerPage` 换页
  重建 route/State——新 route 携带同一拦截属自然事实，但须测试钉住
  （risks.md R2）。
- 读屏：全部 icon-only 动作带本地化 tooltip；页内容 Semantics label；
  底栏焦点顺序 = 视觉顺序。
- Given 查看器放大后按系统返回，When 第一次返回，Then 缩放复位且路由
  不弹；chrome 隐藏态返回先恢复 chrome；显式 back 始终离页。

### R5. Ugoira 保存入口状态一致

现状：导出 GIF 的 IconButton 只在 `downloadMode && _asset != null` 时出现
（ugoira_viewer.dart L223-243）——保存能力与隐式模式+资源准备耦合；
running/finalizing 仅 icon↔spinner 切换；终态只有 snackbar；失败无按钮级
重试入口（只能再点同一图标）。

- 导出钮改为**常显 overlay 动作**（ugoira 的「普通下载可见入口」等价物）。
- 同一入口按 `_exportJob.snapshot.status` 呈现三态：准备中（asset 未就绪
  → 禁用/加载态）、导出中（running/finalizing → 进度 spinner，tooltip/
  文案含进度）、失败（error 图标，点击 = 重试 `_export`）。
- 登录态/下载上下文缺失保持显式报错（现有 `ugoiraLoginRequired` 路径），
  不静默（risks.md R7）。
- Ugoira 不进 `ImageViewerPage` 查看器变体（缩放对逐帧动画无意义）——
  显式排除，见 Decisions D6。
- Given ugoira 作品详情，When 资源未加载/导出中/失败，Then 同一位置入口
  分别呈现准备中/进度/可重试态。

### R6. 系列诚实阅读语义

现状：`WatchlistReadCursor`（watchlist_store.dart L249-275）是按
(accountId, seriesType, seriesId) 分键、只前进的「见过最新 ID」游标，
驱动「有新内容」角标，**不是章节级阅读位置**；打开系列页即 `markSeen`
到最新一话（illust_series_page.dart L151-160）。系列 feed 按 `last_order`
游标分页、**newest first**（series_repository.dart L11 注释 + 测试 fixture
`illusts: [912, 911]`）——draft 中「第 1 话 = feed 首个 ID」的写法不成立：
feed[0] 是最新一话。响应 envelope 已带 `illust_series_first_illust`
（test fixture L88），仓库当前丢弃未解析。

- **core 最小增量声明**（父 §6 允许的最小必要边界，类比 W6 D2）：
  `IllustSeriesEntity` 增 `firstContentId`（int?，merge 时保留旧值），
  `_parseSeriesWorksPage` 解析 `illust_series_first_illust.id`——读 wire
  已有字段，不改 API/持久化语义。
- 系列页 `_SeriesHeader` 增动作行：
  - 「开始阅读」→ `openIllust(firstContentId)`（`firstContentId` 为 null
    时不渲染，不做假目标）；
  - 「返回第 n 话」→ 会话内存态 `seriesRecentOpenProvider`（新建
    `lib/core/series/series_recent_open_store.dart`，进程内 Map，键含
    accountId+seriesId，值=最近打开的 illustId+contentOrder），有记录才
    渲染；点击 `openIllust(记忆 illustId)`；**不落盘**——杀进程后消失是
    设计特性，文案明确用「返回第 n 话」而非「继续阅读」。
  - 写入点：详情页 `IllustSeriesSection` 的 `illustSeriesContextProvider`
    解析出 context 时 post-frame 记录（seriesId+contentOrder 来自
    `/v1/illust-series/illust` 真实响应）。
- 「上一话 / 下一话」沿用 `IllustSeriesSection` prev/next 不变；卡片本体
  tap = 查看全部/目录（现状即 `openIllustSeries`）。
- `markSeen` 行为不变，UI 上继续只承担「有新内容」角标语义，任何文案不
  称其为阅读进度。
- **不建持久化章节进度 schema**（需要 account+series→lastReadContentId
  新表+迁移+回滚，超出 W4 最小闭环——决策 D4，risks.md R3）。
- Given 打开过系列第 n 话详情后回到系列页，When 查看 header 动作行，
  Then 「返回第 n 话」指向该作品且断言其不写入任何持久化 store；
  `markSeen` 游标语义断言与阅读进度无关（W4 门禁：两者用不同状态/断言）。

### R7. 唯一触觉 owner + 首个消费者（§5.6）

现状：`grep HapticFeedback lib/` 零命中——无调用、无封装、无设置开关；
从零建 owner 比收敛散点简单，但必须防各 feature 日后直连
（codebase-download-haptics.md §1）。

- 新建 `lib/app/haptics/app_haptics.dart`（与 `app/motion/`、`app/widgets/`
  平级——交互反馈层，不放 core/非领域逻辑、不放 widgets/非 widget）：
  `abstract final class AppHaptics`，公开 `selectionClick()`（轻确认：
  选中/切换/复制成功）与 `heavyImpact()`（明确震动：进入管理模式/危险
  确认/保存/发送成功/失败）两级。
- 内部三件套（对齐 PixEz `HapticUtil`，裁剪版）：enabled 判定经
  `AppHaptics.configure({required bool Function() isEnabled})` 由
  `PixivFuncApp.build` 注入（读 `settingsProvider` 新字段），不 import
  Riverpod 保持静态可调；最小间隔节流 selectionClick 50ms /
  heavyImpact 120ms；`try/catch` 吞 `MissingPluginException`/平台异常。
- `AppSettings.enableHaptics`（默认 true，additive `_bool` 读取，无
  schemaVersion 变更）+ `SettingsController.setHapticsEnabled` + 浏览设置页
  一行 `SettingsControl`——消费者=AppHaptics 同 stage 落地（质量规范：
  无真实消费者的设置项不得存在；§4.11：禁止先落未使用封装）。
- 首个消费者触点（视觉反馈先自足，触觉同点同分支冗余嫁接）：

  | 触点 | 位置 | 级 |
  |---|---|---|
  | 进入下载选择模式（长按） | `_toggleDownloadMode`（detail L90） | heavyImpact（进入管理模式） |
  | 模式内选中/取消页、退出模式 | 选择态切换处 | selectionClick（选中/切换） |
  | 下载提交成功（单页/所选/全部/卡片） | 各处 `downloadQueuedMessage` snackbar 同点 | heavyImpact（保存成功） |
  | 下载提交失败 | 各处 `downloadSubmissionFailed` snackbar 同点 | heavyImpact（失败需感知） |
  | Ugoira 导出完成/失败 | `_export` snackbar 处 | heavyImpact |
  | 查看器保存当前页成功/失败 | 新工具栏保存动作 | heavyImpact |
  | tag 长按菜单弹出、tag 复制成功 | InfoBlock 菜单处 | heavyImpact（长按确认）/ selectionClick（复制轻确认） |

- 公开注释写明「唯一触觉入口，禁止直连 `HapticFeedback`」；在
  `.trellis/spec/frontend/component-guidelines.md` 增补该约束条目（spec
  修改属本任务范围）；W6/W7 消费契约面见 design.md §2.1。
- 不做 success/warning/error/vibrate/自定义时长——YAGNI，W6/W7 需要时再扩。
- Given 任一触点触发，When 开关开/关、平台不支持，Then 分别断言：
  调用发生 / 不调用且视觉反馈不受影响 / 静默不炸。

## 贯穿约束（loading/empty/error/busy/back/reduced-motion）

- 新表面（compact header、下载选择底栏、查看器工具栏、系列动作行、tag
  菜单）在 entity null/降级态（IllustDetailRestricted/Error 快照回落，
  `_entityOf` L221-229）下的渲染规则逐个声明：header 无实体不渲染或
  仅占位；查看器 entity null 时保存/分享/信息不渲染；系列动作行无目标
  不渲染。
- 所有 icon-only 动作带本地化 tooltip；新文案走 `app_{en,ja,ru,zh}.arb`
  → `flutter gen-l10n` → `tool/gen_l10n_lookup.py` 流程。
- 动画一律取 `MotionTokens`（fast/sheet/dialog），不自造时长；reduced
  motion 下去位移不去状态语义（`MotionTokens.enabled` 现有门控复用）。
- 触觉是冗余通道：开关关或设备不支持时全部反馈仍视觉可辨（§5.6）。

## 行为迁移说明（旧 → 新）

| 页面 | 旧行为 | 新行为 | 迁移说明 |
|---|---|---|---|
| 详情图片 | 长按 = 切隐式下载模式，点空白退出 | 长按 = 进显式选择模式（底栏计数/全选/完成/取消） | 模式从隐式变显式：可发现、可确认；退出路径从"找空白"变"取消/返回" |
| 下载模式内页角标 | 点角标 = 立即下载该页 | 点页/角标 = 选中该页，「完成」统一提交 | 单页下载变为"选 1 页+完成"；批量选择是新能力，肌肉记忆变更显式声明 |
| AppBar 下载全部 | 仅模式内出现 | 常显 | 普通下载不再藏在模式后 |
| Ugoira | 长按进模式 → overlay 导出钮 | 导出钮常显；长按不进模式 | 无页可选对象不挂选择模式；保存入口可见 |
| tag | 长按 = 切屏蔽模式 | 长按 = 动作菜单（搜索/复制/屏蔽/批量模式入口） | 单 tag 屏蔽一步直达；批量屏蔽从菜单项进入，能力不回退 |
| 查看器 | 常驻 AppBar，无动作 | chrome 可显隐 + 底栏动作（页码/适应/全屏/保存/分享/信息） | 点击媒体区切 chrome 为 M3 惯例；单击判定延迟 ~300ms 为双击消歧代价 |
| 查看器双击 | 无 | fit→2.5→fit 循环 | 新增手势，无旧路径破坏 |
| 查看器系统返回 | 直接退路由 | 放大→先复位；chrome 隐→先恢复；再退路由 | 显式 back 与下拉 dismiss 仍直接离页 |
| 系列页 | header 仅信息 | +「开始阅读」「返回第 n 话」 | 纯增量；「继续阅读」措辞刻意不用 |

## Acceptance Criteria

- [ ] 详情：1 页、2 页、长多页、Ugoira、空/降级实体分别 widget 测试；
      窄屏 compact header 渲染+跳转断言；双栏断言 header 不渲染（W4 门禁：
      1、2、长多页、空媒体、Ugoira、系列分页分别验证）。
- [ ] 下载选择模式：进入/勾选/全选/完成提交/取消退出全链路测试；
      「完成」按选中集合逐页提交且进行中任务不取消；`downloadAll` 常显
      断言；`illust_detail_page_test.dart` 中固化旧"点角标即下载"行为的
      用例**同 commit 改写**（质量规范：删 guard 同步删固化测试）。
- [ ] tag 菜单：搜索/复制/屏蔽/批量模式四动作 + 已屏蔽态「解除屏蔽」
      分支测试。
- [ ] 查看器：chrome 显隐三通道、双击缩放循环、单击-不触发-双击-再单击
      序列、fit 复位、页码 sheet 跳转、保存/分享/信息动作（含 entity null
      分支）、空态无 `1 / 0` 且动作禁用、键盘/滚轮等价、PopScope 三级
      优先级（zoom→chrome→放行）与 replace 后拦截仍生效——分别测试
      （W4 门禁：zoom、toolbar、页码、键鼠输入和 back 优先级分别验证）。
- [ ] Ugoira：准备中/导出中/失败重试三态同一入口测试；登录缺失显式报错。
- [ ] 系列：`firstContentId` 解析+merge 保留单测；「开始阅读」打开第 1 话；
      「返回第 n 话」内存态记录/读取/不落盘断言；`markSeen` 游标与阅读
      进度用不同状态/断言（W4 门禁）；系列 feed newest-first 分页回归不破。
- [ ] 触觉：`AppHaptics` 单测（开关/节流/异常吞）；`lib/` 无
      `HapticFeedback` 直调（grep 或分层测试断言）；各触点分级符合
      §5.6；设置开关有真实消费者（AppHaptics）。
- [ ] 读屏：查看器主操作（返回/保存/分享/信息/跳页/全屏）tooltip 与
      Semantics 断言；下载模式角标语义 label。
- [ ] l10n：新 key 四语齐备 + `gen_l10n_lookup.py` 重新生成。
- [ ] `flutter analyze --no-pub`、相关 `flutter test`、`git diff --check`、
      `task.py validate` 全绿；运行时矩阵未覆盖项在 PR 显式标"未验证"。
- [ ] 每个 implement.md checkbox 一个提交；分支
      `task/09-22-artwork-viewer-series-flow`（或按 stage 拆分，见
      implement.md）。

## Decisions（规划定案）

- D1 信息可达 = **常驻 compact header 为主 + ensureVisible 锚点为辅**
  （§4.4 原文要求"首图附近可见标题/作者/页数"，纯锚点不满足可发现性；
  header 只承载信息+跳转，AppBar 常显动作不复制——risks.md R5 的折中
  定案；对比 PixEz 纯锚点方案的评估见 research/implementation-draft.md §1）。
- D2 下载模式语义改「选择-确认」：旧"模式内点角标=立即下载"改为"选中+
  完成提交"——父 §4.4"带标题、计数、完成和取消的显式模式"只有选择
  模型成立；旧用例同 commit 改写。
- D3 双击缩放自实现（InteractiveViewer 无双击），单击/双击由同一
  GestureDetector 双注册交框架消歧，不引 PhotoView、不自造计时器。
- D4 系列只做诚实语义：「开始阅读」走 `illust_series_first_illust`（修正
  draft「feed 首个 ID」之误——feed 是 newest-first）；「返回第 n 话」
  走会话内存态；**不做持久章节进度 schema**（新 schema+迁移+回滚属后续
  任务，risks.md R3）。
- D5 查看器「信息」动作 = bottom sheet 元信息（不回详情页跳转）——
  chrome 内可达，比 pop+ensureVisible 链路更短。
- D6 Ugoira 查看器变体显式排除；PixEz 式 pinch-zoom overlay（zoom_page）
  与 tag 收藏不做。
- D7 触觉两级起步（selectionClick/heavyImpact）；触点级别以 §5.6 分级表
  为准（进入管理模式/保存成功/失败=heavy，选中/切换/复制=selection）——
  research draft §5 触点表中 ugoira 成功=selectionClick 的写法按契约上提
  为 heavyImpact。
- D8 模式状态不跨页面共享：详情页 `_downloadMode` 本地态；查看器只有
  单页保存+保存全部，不建选择模式（risks.md R4 消解）。
- D9 compact header 用固定条（Column 布局槽）而非 pinned sliver——
  常驻语义由布局保证，零 sliver 机制与滚动状态机。
- D10 「返回第 n 话」内存态写入点 = 详情页系列 context 解析处（单写点），
  键含 accountId，多账号隔离正确。

## Out of scope

- 持久化章节阅读进度 schema/迁移（D4，另立任务）；追更页"继续阅读"语义
  统一（W6 R6 已按其 PRD 消费 seen 游标）。
- 详情页排版/信息架构调整（PR #50 基线不动）；双栏 meta 列结构改动；
  `IllustDetailPagerPage` 横向翻页行为。
- 评论页与输入器（W7）；对象组件/管理页/下载任务页 UI（W6）；排行/
  发现页（W2）；设置页信息架构与表单迁移（W8——本包只在浏览设置加
  一行开关，见 design.md §六协调点）。
- `TagChip` 共享组件签名变更、novel 侧 tag 菜单对齐（W5 页面范围）；
  PixEz 式 tag 收藏、pinch-zoom overlay。
- 新公共基件/第二套 MotionTokens/SnackBar/弹层/断点来源（§4.11）。
- 网络/API/持久化业务语义变更（`firstContentId` 为 wire 已有字段的
  解析扩展，已按 D4/父 §6 声明）。
