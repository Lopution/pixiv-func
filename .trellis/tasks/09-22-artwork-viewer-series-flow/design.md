# 技术设计：作品浏览、查看器与系列流程（W4）

设计基线：`main@8067b2d`。逐项精确改动方案见
`research/implementation-draft.md`；现状核实（行号）见
`research/codebase-*.md`；决策点与运行时证据缺口见 `research/risks.md`。
本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

默认单分支 `task/09-22-artwork-viewer-series-flow`，按四阶段串行提交；
阶段边界即 stage 预拆线——若单 PR 超出可审查规模（查看器改造面最大），
切 `…-s1`（阶段 1+2：详情侧交互+触觉）、`…-s2`（阶段 3：查看器）、
`…-s3`（阶段 4：信息可达+Ugoira+系列），**串行合入**（三阶段都触碰
`illust_detail_page.dart` 与 arb 文件，并行会冲突）。

| 阶段 | 条目 | 耦合面 | 风险 |
|---|---|---|---|
| 1 | AppHaptics 封装 + `enableHaptics` 设置位 + bootstrap 注入 + 详情/卡片/Ugoira 下载触点接入 | `app/haptics/`（新建）、`app_settings.dart`、`browse_settings_page.dart`（W8 重叠，见 §六）、arb×4 | 低 |
| 2 | 下载模式显式化（选择语义）+ `downloadAll` 常显 + tag 长按菜单 | `illust_detail_page.dart`、`page_image.dart`、`detail_image_pager.dart`、`ugoira_viewer.dart`、`info_block.dart` | 中（旧测试改写、肌肉记忆变更） |
| 3 | 查看器：chrome/双击/页码/动作/空态/键鼠/PopScope + entity 透传 | `image_viewer_page.dart`、`routes.dart` | 中（手势冲突 R1、PopScope×replace R2） |
| 4 | 窄屏 compact header + InfoBlock 锚点 + `firstContentId` 解析 + 系列动作行 + 内存态「返回第 n 话」 | `illust_detail_page.dart`、`info_block.dart`、`series_models/repository/store`、`illust_series_page.dart`、`series_recent_open_store.dart`（新建）、`illust_series_section.dart` | 中（core 解析增量声明） |

阶段 1 必须先落：`AppHaptics` 与首个消费者（下载/保存触点）同阶段接入
——§4.11「禁止先落未使用的公共封装」、质量规范「无真实消费者的设置项
不得存在」双重门禁。查看器保存触点随阶段 3 的新动作接入（属后续消费者，
不受同阶段规则约束）。

## 二、关键契约

### 2.1 AppHaptics 契约面（W6/W7 消费面在此写死）

```dart
// lib/app/haptics/app_haptics.dart
/// 全库唯一触觉入口。禁止在 feature/widget 层直连
/// `HapticFeedback`（spec 增补条目随本包落地）。
abstract final class AppHaptics {
  /// bootstrap 注入：PixivFuncApp.build 一行
  /// `AppHaptics.configure(isEnabled: () =>
  ///     ref.read(settingsProvider).valueOrNull?.enableHaptics ?? true);`
  static void configure({required bool Function() isEnabled});

  /// 轻确认级：选中/切换、复制成功。最小间隔 50ms。
  static void selectionClick();

  /// 明确震动级：进入管理模式、危险确认、保存/发送成功、失败。
  /// 最小间隔 120ms。
  static void heavyImpact();

  /// @visibleForTesting：清空节流时间戳 / 注册触发观察器。
}
```

- 内部：`isEnabled()` false → 直接返回；距上次触发小于最小间隔 → 丢弃；
  `HapticFeedback.x()` 包 `try/catch`（吞 `MissingPluginException` 等）。
- enabled 读取失败（settings 未就绪/异常）→ 按 `true` 处理（默认开，
  与 `enableHaptics` 默认值一致）；`configure` 只挂读取闭包，wrapper
  自身不 import Riverpod。
- 分级语义以父 §5.6 为准：选中/切换/轻确认 → `selectionClick`；进入管理
  模式/危险确认/保存成功/失败 → `heavyImpact`；普通列表点击不调。
- W6/W7 消费约定：只调这两个公开方法；新增级别（success/warning 等）
  须先回到本契约面扩展，不得在消费侧直连 `HapticFeedback`。W10 验收
  「无第二来源」的静态断言 = `grep HapticFeedback lib/` 仅命中
  `app_haptics.dart`。

### 2.2 详情页：compact header / 锚点 / 选择模式 / tag 菜单

- **compact header**（窄屏 <1200）：置于 AppBar 与图片区之间的固定条
  （`Column` 布局槽，非 sliver——常驻语义由布局保证，不引入 sliver
  pinning 与滚动可见性状态机；D9）。内容：标题一行 ellipsize + 作者 +
  `n/共N页` + 「信息」图标钮（tooltip 本地化）。entity 为 null 的降级态
  不渲染该条。双栏分支不出现（`useTwoPaneDetail` 同处分派）。
- **信息锚点**：metaSlivers 中 `InfoBlock` 的 `SliverToBoxAdapter` 外包
  一层持 `GlobalKey` 的容器；「信息」动作 =
  `Scrollable.ensureVisible(key.currentContext,
  duration: MotionTokens.medium)`——用目标 context 定位，不接管
  `SmoothWheelScroll`/`PrimaryScrollController` 的 controller（risks R9）。
  目标不在视口时 ensureVisible 走最短滚动路径，长作品滚动时长由固定
  duration 天然钳制。
- **下载选择模式**（`_downloadMode` 语义改造）：
  - 进入：图片长按（ugoira 除外）/双栏 pager 长按 → `_toggleDownloadMode`
    + `AppHaptics.heavyImpact()`；退出：底栏「取消」/点空白/系统返回 →
    `selectionClick`。
  - chrome：`Scaffold.bottomNavigationBar` 槽挂 `AnimatedSwitcher`
    （`MotionTokens.fast`）底栏：「已选 n / 共 N · 全选 · 完成 · 取消」。
    `n=0` 时「完成」禁用。AppBar title 保持 `illustDetailTitle`（底栏已
    承载模式上下文，不再双份表达）。
  - 选择态：`Set<int> _selectedPages`（页 index）；`DetailPageImage` 增
    `selected`/`onToggleSelect` 参数，模式下 `onTap` 从「开查看器」改为
    「切选中」（placeholder 页仍不可点）；角标优先级 spinner >
    选中(check) > 未选中(空心圈)。
  - 完成：循环 `download(entity, i)`（`illustDownloadControllerProvider`，
    内部 retry/dedupe 安全）→ 成功 snackbar + `heavyImpact`，退出模式；
    失败 snackbar + `heavyImpact`，保留模式与选中态（失败输入保留，父 §6）。
  - 进行中任务不随退出取消；「全选」= 全部页 index 入集合。
- **下载全部常显**：AppBar `Icons.file_download_outlined` 解除
  `_downloadMode` 门控；成功/失败 snackbar 同点嫁接 `heavyImpact`。
- **tag 菜单**：`InfoBlock` 的 `TagChip.onLongPress` 从 `_blockMode` 翻转
  改为 `showAppBottomSheet` 菜单（`heavyImpact` 确认长按成立）：
  搜索该 tag（`openTagSearch`）/ 复制（`Clipboard.setData` + snackbar +
  `selectionClick`）/ 屏蔽或解除屏蔽（`muteStoreProvider.toggleTag`）/
  进入批量屏蔽模式（翻转 `_blockMode`）。`TagChip` 组件签名不动——菜单
  是调用点行为，非组件变体。屏蔽模式内 tap=切屏蔽态、菜单不再挂长按
  语义（模式内长按仍弹菜单无碍，但菜单项按 `blocked` 现实态命名）。

### 2.3 查看器状态机（`image_viewer_page.dart`）

- 组件形态：`StatefulWidget` → `ConsumerStatefulWidget`（动作须读
  `illustDownloadControllerProvider`/`shareServiceProvider`）；构造新增
  `IllustEntity? entity`（routes `_ImageViewerRoute` 把 L126 已解析实体
  透传——store 优先、extra 快照兜底，不建第二数据通道；entity null 时
  保存/分享/信息动作不渲染，页码/缩放/返回不受影响）。
- **chrome**：`_chromeVisible`（默认 true）。切换通道三选一等价：
  媒体区单击、底栏 `fullscreen`/`fullscreen_exit` 钮、键盘 `F`。
  隐藏 = AppBar + 底栏 + `SystemChrome.setEnabledSystemUIMode(
  SystemUiMode.immersiveSticky)`；恢复逆向。显隐动画
  `AnimatedOpacity/Slide` + `MotionTokens.fast`。
- **单击/双击消歧**：同一 `GestureDetector`（包 PageView 外侧）同时注册
  `onTap`（切 chrome）与 `onDoubleTap`（缩放循环）——框架手势竞技场使
  单击延迟 ~kDoubleTapTimeout 判定，无双击时立刻成立（risks R1 的定案；
  不引 PhotoView、不自造计时器）。双击循环：当前 scale <1.5 → 2.5
  （以双击点为 focal）；否则 → identity。`TransformationController.value`
  经 `Matrix4` tween + `MotionTokens.fast` 动画驱动。
- **页码**：底栏左侧 `n / total` 改 `InkWell`+tooltip →
  `showAppBottomSheet` 页码选择 → `jumpToPage`；翻页照旧走
  `onPageChanged → replaceImageViewerPage`（replace 重建 route/State 是
  现状语义：缩放/chrome 不跨页存活——既定事实，测试钉住不宣称跨页保留）。
- **底栏动作**：页码 | `fit_screen`（当前页复位，键盘 `0`）|
  `file_download_outlined`（保存当前页 → `download(entity, _activePage)`，
  复用 `stateFor` 判定的角标语义：downloading 禁用/exist 置底/success·
  fail snackbar 同详情）| `share_outlined`（`SharePayload.illust`）|
  `info_outline`（bottom sheet：标题/作者/日期/ID/页数 + 「保存全部」+
  「打开详情页」）。全 icon-only 带 tooltip。
- **空态**：`_pageCount==0` → 计数位 `SizedBox.shrink`（移除 `1 / 0`），
  body `viewerNoImages` 占位保留，保存/分享/跳页/适应/全屏动作
  visible-disabled（语义化禁用，不移除——读屏可感知「存在但不可用」）。
- **键鼠**：`CallbackShortcuts`（`←`/`→`/`J`/`K` 翻页；`+`/`-`/`0`；
  `F`；`Esc`/`Backspace` → `pop()`；`S` 保存；`I` 信息）+
  `Focus(autofocus: true)`；`Listener(onPointerSignal)`：滚轮 = 以指针
  位置为焦点缩放（focal-point matrix），Shift+滚轮翻页。
- **PopScope**（消费 W1 结论）：`canPop = !_activeZoomed && _chromeVisible`；
  `onPopInvokedWithResult` 在 `!didPop` 分支内：zoomed → 复位当前页；
  否则恢复 chrome。显式 AppBar back、`DragToDismiss` 下拉、键盘
  `Esc`/`Backspace` 均命令式 `pop()`——绕行 `popDisposition` 是 W1 已钉
  的 SDK 事实，不与系统返回混用。
- **DragToDismiss 不动**：`enabled: !_activeZoomed` 现状保留；放大态
  下拉仍归 InteractiveViewer pan。

### 2.4 系列诚实语义

- **wire 解析增量**：`_parseSeriesWorksPage` 读 `illust_series_first_illust`
  → `IllustSeriesEntity.firstContentId`（int?，payload 缺失保持 null，
  merge 保留旧值——与 `latestContentId` 同规则）。feed newest-first 语义
  不动；`SeriesWorksPage` 不加并行通道，first 只落在 canonical entity。
- **会话内存态**：`lib/core/series/series_recent_open_store.dart` —
  `NotifierProvider` 持 `Map<String, ({int illustId, int? contentOrder})>`，
  键 `'$accountId:manga:$seriesId'`。写入点唯一：`IllustSeriesSection`
  的 `illustSeriesContextProvider(illustId)` 解析出 context 时 post-frame
  `record(...)`（任何入口进详情都会解析 context → 单写点覆盖全部路径）。
  不落盘、无 TTL、进程死即清——「返回第 n 话」而非「继续阅读」。
- **系列页 header 动作行**：「开始阅读」`firstContentId != null` 才渲染
  → `openIllust(first)`；「返回第 n 话」memory 命中才渲染（contentOrder
  缺失用「返回上次作品」兜底文案）→ `openIllust(illustId)`。两动作与
  `WatchlistToggle` 同区。
- **`markSeen` 不动**：继续只驱动「有新内容」角标；门禁要求游标与进度
  用不同状态/断言——测试分别断言 markSeen 写游标、memory 写内存 Map、
  两者互不粘黏。

### 2.5 Ugoira 一致入口

- 导出钮脱离 `downloadMode && _asset != null` 双门控改为常显 overlay；
  三态同一入口：asset null=准备中（disabled+spinner 态，tooltip 说明）、
  running/finalizing=进度 spinner（tooltip 含百分比）、failed=error 图标
  点击即重试 `_export`。ugoira 作品长按不再进选择模式（无页可选；
  `onLongPress` 传 null）。
- 登录态/上下文缺失沿用 `ugoiraLoginRequired` 显式报错（risks R7）。

## 三、复用与禁止 / owning files

复用：`showAppBottomSheet`/`showAppDialog`、`MotionTokens`、
`FeedEmpty`/`FeedLoading` 族、`PixivImage` 质量交接、`DragToDismiss`、
`CallbackShortcuts`、`illustDownloadControllerProvider`
（`stateFor`/`download`/`downloadAll`）、`muteStoreProvider.toggleTag`、
`ShareService`、`openIllust`/`openTagSearch`/`openIllustComments`、
`AppBreakpoints.useTwoPaneDetail`、`TwoPane`、`SmoothWheelScroll`
（不接管其 controller）、`SettingsControl`/`persistSettings`。

owning files（本 leaf 可改）：

- `lib/features/illust/detail/**`（含 `widgets/`）、
  `lib/features/illust/viewer/image_viewer_page.dart`、
  `lib/features/series/illust_series_page.dart`；
- `lib/app/haptics/app_haptics.dart`（新建，唯一触觉 owner）；
- `lib/core/series/{series_models,series_repository,series_store,`
  `series_recent_open_store}.dart`（解析增量 + 内存态，§2.4 已声明）；
- `lib/core/settings/{app_settings,settings_controller}.dart`、
  `lib/features/settings/pages/browse_settings_page.dart`（一行开关，
  W8 协调见 §六）；
- `lib/app/navigation/routes.dart`（viewer 签名透传）、`lib/app/app.dart`
  （一行 configure）、`lib/app/widgets/card_actions/illust_card_actions.dart`
  （下载触点）、`lib/app/widgets/tag_chips.dart` 不改签名仅核对；
- `lib/l10n/app_{en,ja,ru,zh}.arb` + 生成物；`test/**`；
  `.trellis/spec/frontend/component-guidelines.md`（增补触觉条目）。

禁止：`HapticFeedback` 第二调用点；持久化进度 schema；`PhotoView` 等
新依赖；`TagChip`/`BookmarkSwitchButton` 签名变更；新 provider 滥用
（除 `seriesRecentOpenProvider` 已声明外无新增 store）；`firstContentId`
之外的 wire 字段扩张；W2/W3/W5/W6/W7/W8/W9 owning files。

## 四、测试策略

- 纯函数/模型单测：`firstContentId` 解析与 merge 保留；
  `seriesRecentOpenStore` record/read/多账号键隔离/不落盘；
  `AppHaptics` 节流窗口、开关关断、异常吞没（注入 debug 观察器断言
  级别序列）。
- widget 测试：详情（1/2/长多页/Ugoira/降级实体 × 窄屏 header +
  锚点跳转 + 双栏不出现）；选择模式全链路（进入→勾选→全选→完成提交
  →退出，进行中任务不取消，`n=0` 完成禁用）；tag 菜单四动作；
  查看器（chrome 三通道、双击循环、单击/双击序列、页码 sheet、
  动作 entity-null 分支、空态无 `1/0`+禁用、键鼠映射、PopScope 三级
  + replace 后拦截仍生效——用真实 `context.replace` 或等价 route
  替换复现 risks R2）；ugoira 三态；系列 header 动作行。
- 旧测试同步改写（同 commit）：`illust_detail_page_test.dart` 的
  「点角标即下载」断言、`1 / 0` 断言、ugoira long-press 交接用例——
  固化旧行为的断言随 guard 删除一并改写，不留 `skip:`。
- 回归锚点：Hero/PixivImage transitionKey、`DragToDismiss` enabled 边界、
  `replaceImageViewerPage` 路由形态、feed newest-first、
  `markSeen` 只前进。

## 五、运行时证据缺口（PR 标「未验证」）

Android predictive-back 真实手势链与 `immersiveSticky` 表现、真机触觉
（开关开/关双路径与节流体感）、TalkBack/Narrator 焦点顺序（查看器/
下载模式/compact header 优先）、1.3x 大字体底栏溢出、桌面滚轮缩放
手感、双击 ~300ms 单击延迟的实际观感。

## 六、跨包协调点

- **W6/W7 消费契约**：§2.1 即冻结面；两包只调公开两级，不得直连
  `HapticFeedback`（W6 PRD 已按此登记依赖）。本包 API 变更须回写
  通知两包。
- **W8 文件重叠**：`browse_settings_page.dart`/`app_settings.dart`/
  `settings_controller.dart` 同时是 W8 表单迁移面——本包只做最小增量
  （一个字段 + 一行 `SettingsControl`），若与 W8 并行，按合入顺序
  串行 rebase，不改表单结构（父 §3「不让两个并行分支编辑同一文件」
  的执行说明：必要时把开关行拆成独立小提交便于先合）。
- **arb 并行冲突**：W2–W9 多叶并行都会改 `app_{en,ja,ru,zh}.arb` 并
  重新生成 `lookup.dart`/`app_localizations*.dart`——arb 冲突按并集
  保留双方 key 后重新 `flutter gen-l10n` + `tool/gen_l10n_lookup.py`，
  生成物永不手解。
- **W1 已合入契约**：PopScope 命令式/征询式分派直接消费，
  不在本包重新论证。
- **W6 既有假设**：`WatchlistReadCursor` 语义本包保持原样（seen=见过
  最新 ID），W6 R6「继续阅读=seenId」的既有解读不受影响。
