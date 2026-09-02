# 技术设计：真实高频 UX 正确性修复

设计基线：HEAD `70d470f`（产品代码等同 `9d1cb1b`）。

## 一、阶段划分与依赖

四个阶段严格串行，每阶段独立提交。人眼判定项统一留到全部完成后由用户真机验证。

| 阶段 | 内容 | 耦合面 | 风险 |
|---|---|---|---|
| 1 | U9 作者跳转 + C11 登录错误态 + C20 详情页 i18n | 三个互不相干的局部改动 | 低 |
| 2 | U2 热门标签代表图 | 搜索页 + trending controller | 低 |
| 3 | D7/U8 Profile header | profile header delegate | 中 |
| 4 | U1 下拉刷新 | **全部 feed 页共用的滚动基建** | 高 |

阶段 1–3 之间没有技术依赖，顺序按「风险递增」排。阶段 4 单独放最后，因为它是唯一一个
改动会同时影响所有列表页的项，且下游 child 的行为设计要以它的结果为准。

## 二、阶段 1

### 2.1 U9 详情页作者区域可点击

**现状**：`lib/features/illust/detail/illust_detail_page.dart:571-609` 是一个裸 `Row`：
`SizedBox.square(Key('illust-author-avatar'), CircleAvatar)` + `SizedBox(width: 20)` +
`Expanded(Column(name, account))`。没有任何手势包裹。

`showUserPage` 已在 line 26 import，并在 line 868 被 `_openPixivRoute` 使用（服务于简介
正文里的 pixiv 用户链接）。

**方案**：用一个手势层包裹整个 `Row`，回调 `showUserPage(context, entity.user.id)`。

约束：
- 包裹层加在 `Row` 外侧，**不移动** `Key('illust-author-avatar')` 的位置，
  `illust_detail_page_test.dart` 已按该 key 定位。
- 用带视觉反馈的形式（`InkWell` 一类）而非裸 `GestureDetector`，让「可点」在真机上可感知；
  但不得改变现有布局尺寸与间距。
- 不新增路由，不改 `showUserPage` 签名。

### 2.2 C11 登录页错误态

**现状**：`lib/features/login/login_page.dart:117-123`
```
ref.watch(settingsProvider).when(
  loading: () => const ReplicaScaffold(child: SizedBox.shrink()),
  error:   (error, stackTrace) => const ReplicaScaffold(child: SizedBox.shrink()),
  data:    (settings) { ... },
)
```

**方案**：
- loading → `ReplicaScaffold` 内居中进度指示。
- error → 错误说明 + 「重试」按钮，重试执行 `ref.invalidate(settingsProvider)`。

**设计陷阱**：`data` 分支靠 `settings.languageTag` 决定文案语言，而 error 分支拿不到
settings。错误态文案不能依赖 `settingsProvider`，需退回系统语言
（`Localizations.localeOf(context).toLanguageTag()` + `ReplicaStrings.fromTag`，
该组合已在 `profile_header_delegate.dart:145-149` 等处使用）。

不做的事：不「回退到默认设置继续渲染登录页」。那会把一次真实的存储读取失败伪装成正常
状态，用户会在一个语言、网络模式都不是自己设置的页面上登录。

### 2.3 C20 详情页 i18n

**现状**：`illust_detail_page.dart` 9 处硬编码中文，行号
254（页面标题回退）、619 / 620（投稿日期，未知与有值两种）、641（尺寸）、
807（打开链接失败）、889（作品受限）、907（作品不存在）、930（加载失败）、934（重试）。

**方案**：全部迁入 `lib/core/i18n/replica_strings.dart`。

注意成本：`ReplicaStrings` 按语言分块存放（zh / en / ja / ru 各一块），新增 key 必须
四块同时补齐。619/620/641/807/889 含插值，需要参数化形式，不能拼接翻译片段。

范围纪律：本 child 只处理详情页这一个文件。仓库其它页面的硬编码中文留给对应 child，
不在这里顺手扩散。

## 三、阶段 2：U2 热门标签代表图

**现状**：`lib/core/search/search_repository.dart:327-340` 已解析 `representative`
（`IllustEntity?`）；`lib/features/search/search_page.dart:150-195` 的 `_TrendingTagTile`
只渲染 `Icon + '#${tag.displayName}'`，`representative` 仅用于 `onLongPress`。

**方案**：
- 卡片改为图文形式，展示 representative 的预览图。
- `onTap` 维持现有语义 = 搜索该标签（不要因为加了图就改成打开作品）。
- 进入代表作品保留为次要操作。
- `representative == null` 时保留纯文字形态与现有提示，不显示破图占位。

**缓存**：现状是 `FutureProvider.autoDispose`（`search_trending_controller.dart:8`）——
最后一个 listener 移除即销毁，所以**每次进入搜索页都会重新请求一次** trending-tags。

**最小改法：去掉 `.autoDispose`。** provider 在 App 生命周期内常驻，进出搜索页不再重复
请求，这就是 U2 那条要求的真实诉求。

**不要**为此引入日期判定或持久化缓存：跨日刷新在实践中由 App 重启自然发生；即便 App
连续运行跨日，多显示一天的旧热门标签也没有任何产品后果。为这点收益新增一层缓存基础设施，
正是 parent PRD R6 要避免的。

副作用是 provider 不再随页面销毁，`ref.onDispose(token.cancel)` 也就不会在离开页面时
取消请求 —— 这反而是想要的：请求跑完并留下结果，下次进入直接有数据。
`ref.watch(accountStoreProvider.select(...))` 的账号切换重建行为保持不变。

## 四、阶段 3：D7/U8 Profile Header

**现状**（`lib/features/profile/profile_header_delegate.dart`）：头像和用户名分别由展开态
与折叠态子树渲染，滚动临界点会突然替换；展开态整块还跟随背景透明度一起消失，导致
无头像占位过大，并在滚动过程中与 toolbar 标题争抢空间。

**方案**：按 PixEz 的 `NestedScrollView` + `SliverAppBar` 分层方式，把背景、展开身份区和
pinned toolbar 分成独立层。

- 背景带独立按 `progress` 淡出。
- 展开身份区包含一个固定 52dp 半径头像、名字、账号、统计和操作按钮，整体向上移出
  header；头像不再缩进 toolbar，也不与背景共用透明度。
- 头像与名字之间使用固定间距，身份区在 toolbar 淡入前完全离场，避免任何中间帧重叠。
- pinned toolbar 只渲染返回、居中用户名和操作控件，不再放第二个头像。
- `topInset` 计入最小 header 高度，toolbar 控件放在状态栏安全区下方。

**回归面**：`test/user_profile_test.dart` 已有 header 相关 widget test
（如 `current profile header has no settings entry`），改动后必须仍然通过。
`ReplicaProfileHeaderGeometry` 是纯几何快照且已被测试引用，适合直接为新的插值补单测。

## 五、阶段 4：U1 下拉刷新

### 5.1 先决问题：旧 spec 契约曾固化当前行为

`.trellis/spec/frontend/component-guidelines.md` 的 *Artwork Detail Transition Contract*
第 3 节，曾经对 `PullToRefresh` 写过一条**已废弃的契约**：

> `PullToRefresh` tracks the leading-edge drag distance separately from Flutter's armed
> lifecycle. […] Once edge tracking has started, apply every vertical `scrollDelta` until
> the matching `ScrollEndNotification`; **a reverse update may have a null `dragDetails`
> while the scrollable bounces back.**

同文件结尾还曾有一条：

> Do not let the framework's armed visual state pin the indicator after the user has
> reversed the drag; **keep that correction in the shared wrapper** rather than creating a
> second per-page refresh implementation.

也就是说：当时的自建状态机**不是意外产物，是旧 spec 明确要求的**。「与框架 armed 生命周期
分开跟踪拖拽距离」正是 `_dragOffset` 那一套；「reverse update 的 `dragDetails` 可能为 null」
这一句，直接排除了用指针状态过滤 overscroll 的做法 —— 而那恰恰是「手指离开后惯性仍唤出
指示器」的成因。

旧第 6 节还规定了必需测试：
> Pull-to-refresh tests arm, reverse, and release a real scrollable, asserting linear
> indicator movement before release, dismissal below the threshold, and exactly one refresh
> after a valid release.

**因此阶段 4 不是「让实现回到 spec」，而是要推翻这条 spec 契约的一部分。** 必须先把
「哪部分是被真机证伪的」和「哪部分仍然有效」分清楚：

| 契约条款 | 判定 | 依据 |
|---|---|---|
| 反向回滑时指示器要跟随，不被框架 armed 状态 pin 住 | **保留**，这是正确的产品意图 | 用户抱怨的正是它没做到 |
| 低于阈值释放要取消、不调 `onRefresh` | **保留** | 行为正确 |
| `onRefresh` 开始后滚动通知不得重置刷新态 | **保留** | 行为正确 |
| 与框架 armed 生命周期**并行**维护第二套拖拽距离判定 | **推翻** | 见 5.2 根因 1 |
| edge tracking 开始后无条件应用每个 `scrollDelta` 直到 `ScrollEndNotification`；不得用 `dragDetails` 过滤 | **推翻** | 见 5.2 根因 2、3；这是惯性唤出与残留的直接成因 |

结论：spec 的**目标**是对的，spec 规定的**实现手段**被真机证伪。阶段 4 必须同步修订
`component-guidelines.md` 的对应条款与第 6 节的测试要求 —— 代码与 spec 一起改，不允许
只改代码留下一条已被推翻的契约，也不允许为了「符合 spec」而保留已证伪的实现手段。

这正是 parent PRD R1 写的：错误 guard 阻塞正常行为时删除或放松它，并同步删掉固化该错误
行为的测试与规范，而不是叠加下一层补丁。

#### 5.1.1 规范同步

本次实现同步更新了 `component-guidelines.md`：`PullToRefresh` 的构造签名增加可选的
`isNested`，刷新规则独立为 `Shared Pull-to-Refresh Contract`，并只保留可观察行为与
终态断言要求。实现采用 `easy_refresh` 的具体配置记录在 5.3，规范不再锁定某套自建
滚动状态机。

### 5.2 根因

`lib/app/pull_to_refresh.dart` 的类文档（line 9-12）声称框架仍然持有 refresh lifecycle 与
trigger semantics，这个 wrapper 只负责渲染指示器。**实现与该声明不符**，具体有四处：

1. **两套阈值判定并存。** `_handleRefresh`（line 164-172）在框架已经决定刷新之后，用自己
   维护的 `_dragOffset` 再判一次，不达标就 `return` 掉框架的刷新。触发语义因此是两套：
   框架的 armed 判定 + 这里的 `_dragOffset` 判定，两者依据不同的量，必然出现不一致。

2. **overscroll 分支不检查指针状态。** line 93-98 对 `OverscrollNotification` 无条件
   `_tracking = true`。而上方的 `ScrollStartNotification` 分支（line 73-75）是检查了
   `notification.dragDetails != null` 的。惯性（ballistic）阶段产生的 overscroll 同样满足
   line 93 的条件，于是在手指已经离开屏幕后重新打开跟踪并推进 `_progress` ——
   这正是「手已经离开、甩动仍能引出刷新图标」。

3. **滚动结束不收指示器。** line 100-102 的 `ScrollEndNotification` 只置
   `_tracking = false`，`_progress` 归零只发生在 `_onStatusChange(canceled)` 或
   `_handleRefresh` 的 `finally`。若 `_tracking` 是被上述 (2) 打开的，框架的
   `RefreshIndicator` 从未进入 armed，就不会产生 `canceled` 状态回调，`_progress` 保持非零
   —— 指示器停在屏幕上不消失，即用户报告的「卡住」。

4. **反向回滑时指示器不跟随。** 指示器位置只由 `_progress` 决定
   （line 200-201 `Transform.translate`），而 `_progress` 依赖 `_applyDragDelta` 收到负
   delta。框架 `RefreshIndicator` 进入 armed 后会接管 overscroll，列表位移由框架持有，
   此时反向拖动不再向本 listener 提供对应的负 delta，`_progress` 因此不变；同时框架仍在
   移动列表内容。观感就是「图标不动，跟列表一起上移」。

结论：问题不在某一行判断写错，而在于这个 wrapper 事实上重新实现了一遍 scroll lifecycle，
并与框架的那一套并行运行。修法是收敛到一套，而不是给 line 93 再加一个条件。

### 5.3 最终方案

实现采用 PixEz 当前使用的 `easy_refresh` 体系（版本 `3.5.1`），把刷新生命周期、
阈值和回弹都交给共享组件管理：

- `PullToRefresh` 是无状态共享 wrapper，使用 `EasyRefresh` + `MaterialHeader`。
- 普通列表的 `MaterialHeader` 固定为 `position: IndicatorPosition.above`、
  `safeArea: true`、`clamping: true`；嵌套 tab 使用 `position: locator`、
  `safeArea: false`，并由首项 `HeaderLocator` 提供定位。两条路径都取当前主题色；
  组件不另建 scroll-notification 状态机，不累加拖拽距离，也不否决组件已经触发的刷新。
- `clamping` header 会把指示器位移和内容位移分开保存。反向拖动时先收回 header，
  指示器完全离屏后剩余手势才交给列表，因此不会出现“指示器与内容一起上移”。
- 普通列表使用 `EasyRefresh` 的 child 构造；用户页保留外层
  `NestedScrollView`，每个实际展示的 tab body 使用一个 `isNested: true` 的
  `EasyRefresh` locator wrapper，并在列表首项放置 `HeaderLocator`（sliver 列表使用
  `HeaderLocator.sliver()`）。不再给整个 `NestedScrollView` 额外包一层，避免重复刷新
  生命周期。该嵌套路径有独立 widget test 覆盖。
- 业务页面已有的 `AlwaysScrollableScrollPhysics` 保留，由 `EasyRefresh` 的滚动作用域
  组合出组件 physics；其它刷新调用点不需要复制或改写。

这是一个依赖边界的选择：下拉时内容会随 header 产生组件定义的回弹位移，视觉上接近
PixEz，而不是以前自绘实现的悬浮图标。这个位移属于 `easy_refresh` 的滚动语义，业务层
不再对它做二次解释。

### 5.4 测试处置

刷新回归测试直接驱动真实 scrollable，覆盖：

- 未达阈值的下拉、反向回滑并释放；
- 已 armed 的反向回滑，断言 indicator 收完前列表 offset 不变，收完后剩余手势才滚动；
- 达到阈值释放，断言恰好一次刷新且最终 indicator 隐藏；
- 指针抬起后的 ballistic overscroll 不重新出现 indicator；
- `NestedScrollView` tab body 的 locator wrapper 刷新恰好一次且最终 indicator 隐藏。

所有用例都断言交互终态，不只断言中途轨迹；反向回滑用例必须越过组件 arm 阈值，避免
测试走到另一条未 armed 的框架分支。

## 六、不做的事

- 不再引入除已批准 `easy_refresh` 以外的滚动/刷新库。
- 不新建共享 feed grid 组件（那属于 `settings-productization` 的 C9 在 design 阶段
  自行判断的范围，本 child 不预先替它决定）。
- 不改路由架构、不碰网络/下载/账号代码。
- 不处理详情页以外页面的硬编码中文。
