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

**现状**（`lib/features/profile/profile_header_delegate.dart`）：

- line 35 `avatarRadius => lerp(72, 20)` —— **计算了但全文件无人使用**。
- line 173 `_ExpandedProfile` 内 `_Avatar(radius: 72)`（直径 144dp），硬编码。
- line 308 `_CollapsedProfile` 内 `_Avatar(radius: 20)`，硬编码。
- line 91-112 由 `isFullyCollapsed` 在两个子树之间二选一。
- line 37 / 92-93 `backgroundOpacity => 1 - progress` 包裹**整个** `_ExpandedProfile`。

三者叠加的实际观感 = 一个 144dp 大头像保持原尺寸整体淡出，到临界点突然被 20 半径小头像
替换。默认「无头像」占位（`Icon(Icons.person_outline, size: radius)`）因此呈现为一个
巨大的、正在淡出的灰色图标，看起来像背景水印。这与 U8 的用户描述完全吻合。

**方案**：把头像从两个互斥子树里提出来，作为 delegate `Stack` 中一个独立的、由 geometry
驱动的层。

- 头像层不被 `backgroundOpacity` 包裹 —— 背景图淡出，头像不淡出。
- 半径改用已有的 `geometry.avatarRadius`，并把展开端从 72 调到 **48–56**
  （直径 96–112dp，取值在实现时按真机观感定档）。折叠端保持 20。
- 位置在展开锚点（当前 `top: backgroundHeight - 120` 居中）与折叠锚点
  （`_CollapsedProfile` 中 back button 右侧）之间按 `geometry.progress` 连续插值。
- `_ExpandedProfile` / `_CollapsedProfile` 各自去掉内部的 `_Avatar`，其余布局不动。
- 用户名：折叠端出现在 toolbar 居中位置。展开端名字与 toolbar 标题之间做交叉淡入淡出，
  不要求两者共用同一个 widget。

**必须注意的重叠**：`_CollapsedProfile` 折叠态左侧有 back button、右侧有
restrict selector / edit / share。头像提为独立层后要保证在过渡的任意 progress 上都不与
它们重叠 —— 特别是 `canPop` 为真与为假时左侧起点不同（line 298）。

**回归面**：`test/user_profile_test.dart` 已有 header 相关 widget test
（如 `current profile header has no settings entry`），改动后必须仍然通过。
`ReplicaProfileHeaderGeometry` 是纯几何快照且已被测试引用，适合直接为新的插值补单测。

## 五、阶段 4：U1 下拉刷新

### 5.1 先决问题：现行 spec 契约固化了当前行为

`.trellis/spec/frontend/component-guidelines.md` 的 *Artwork Detail Transition Contract*
第 3 节，对 `PullToRefresh` 有一条**现行契约**：

> `PullToRefresh` tracks the leading-edge drag distance separately from Flutter's armed
> lifecycle. […] Once edge tracking has started, apply every vertical `scrollDelta` until
> the matching `ScrollEndNotification`; **a reverse update may have a null `dragDetails`
> while the scrollable bounces back.**

同文件结尾还有一条：

> Do not let the framework's armed visual state pin the indicator after the user has
> reversed the drag; **keep that correction in the shared wrapper** rather than creating a
> second per-page refresh implementation.

也就是说：当前的自建状态机**不是意外产物，是 spec 明确要求的**。「与框架 armed 生命周期
分开跟踪拖拽距离」正是 `_dragOffset` 那一套；「reverse update 的 `dragDetails` 可能为 null」
这一句，直接排除了用指针状态过滤 overscroll 的做法 —— 而那恰恰是「手指离开后惯性仍唤出
指示器」的成因。

第 6 节还规定了必需测试：
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

#### 5.1.1 spec 具体改法（阶段 4 的执行依据）

这条 spec 的根本问题不是「写错了」，而是把行为契约（what）与实现指令（how）绑在同一段里，
导致实现方无法只丢弃被证伪的那一半。下面是定稿的改法，阶段 4 照此执行。

**(1) 第 3 节 —— 拆成纯行为契约 + 一条硬约束**

删除原有的 `PullToRefresh` 整条（含 "tracks the leading-edge drag distance separately
from Flutter's armed lifecycle"、"apply every vertical `scrollDelta` until the matching
`ScrollEndNotification`"、"a reverse update may have a null `dragDetails`" 三处 how），
替换为：

```markdown
- `PullToRefresh` is the single shared refresh wrapper. Feeds must not add a
  second per-page refresh implementation.
- Indicator behavior, stated as observable outcomes:
  - A pull that reverses before release moves the indicator back with the
    finger; releasing below the threshold cancels without calling `onRefresh`.
  - Motion produced after the pointer has lifted (ballistic settle, bounce,
    overscroll) never starts or resumes a pull.
  - Every pull ends with the indicator hidden — whether it refreshed or cancelled.
  - Once `onRefresh` starts, no scroll activity resets the refreshing state
    until that Future completes. Exactly one `onRefresh` per qualifying pull.
- The refresh threshold is decided in exactly one place. The wrapper must not
  maintain a drag-distance judgement in parallel with the framework's, and must
  not veto a refresh the framework has already triggered.
```

三处实质变化：
- 全部 how 移除 —— 它们是实现路径，不属于 spec，且正是它们把 bug 锁死。
- **补两条原本缺失的契约**：「指针抬起后的运动不得开始下拉」与「每次下拉结束指示器必须
  隐藏」。用户报告的两个缺陷恰好落在旧 spec 的空白区 —— 旧条款只规定了下拉过程，
  从未规定终态。
- **新增「阈值判定只有一处」** —— 这是本次事故的真正教训，比修某一行更值得沉淀。

**(2) 第 4 节错误矩阵 —— 一行扩为三行**

原有的 `Armed pull reverses before release` 覆盖不到用户实际遇到的两种情况，替换为：

```markdown
| Armed pull reverses before release | Indicator follows the finger back; releasing below threshold cancels without calling `onRefresh`. |
| Scroll motion continues after the pointer lifts | No pull starts or resumes; indicator stays hidden. |
| Refresh completes or cancels | Indicator returns to hidden; nothing residual on screen. |
```

**(3) 第 6 节测试要求 —— 必须断言终态**

原文只要求断言 "linear indicator movement **before release**"。「指示器卡在屏幕上」这个
缺陷之所以能通过既有测试，正是因为**没有任何用例断言下拉结束后的可见性**。替换为：

```markdown
- Pull-to-refresh tests drive a real scrollable and cover: reverse-then-release
  below threshold, a valid release, and pointer-up ballistic overscroll.
  Assert the indicator's **final** visibility in every case, not only its
  motion before release.
```

**(4) 文件结尾 —— 保留意图，去掉已证伪的实现指向**

```markdown
Do not let the framework's armed visual state pin the indicator after the user
has reversed the drag. Correct that in the shared wrapper — but not by running
a second scroll-notification state machine alongside the framework's. If the
framework cannot express the visual, drop the visual, not the correctness.
```

**(5) 结构调整 —— 把这些条款移出详情页契约**

`PullToRefresh` 与「作品详情页过渡」没有任何关系，它是历史上某个 task 顺手并入的。
后果很实际：以后改下拉刷新的人不会想到去 *Artwork Detail Transition Contract* 里找规则
（本次规划也是靠通读整个 spec 文件才发现）。

把上述 (1)(2)(3)(4) 的条款，连同第 2 节 Signatures 中的 `PullToRefresh` 构造签名，
一并拆出为独立小节 `## Shared Pull-to-Refresh Contract`。详情页契约中只保留与 Hero /
预览 / 头像相关的内容。

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

### 5.3 方案

**先决核查（2026-09-01，已在 Flutter 3.47 源码确认）**：
`packages/flutter/lib/src/material/refresh_indicator.dart` 的公开成员里，与状态有关的
只有 `onStatusChange`（`ValueChanged<RefreshIndicatorStatus?>`），而
`RefreshIndicatorStatus` 是**离散**枚举：`drag / armed / snap / refresh / done / canceled`。
全文件不存在任何暴露连续进度（0..1）的公开 getter 或 `Animation<double>`，
内部 `_positionController` 是私有的。

因此「用框架暴露的刷新进度驱动一个自定义指示器」这条路**在 API 层面就不成立**，
不能作为方案写下去。

**首选方案：直接换成标准 `RefreshIndicator`，放弃自定义指示器外观。**

- 用标准构造（非 `noSpinner`），采用框架自带的 Material 指示器，通过 `color` /
  `backgroundColor` / `strokeWidth` / `displacement` 做主题化。
- 删除全部自建状态机与自绘指示器：`_onScroll`、`_tracking`、`_dragOffset`、
  `_indicatorOffset`、`_viewportDimension`、`_progress`、`_dismissController`、
  `_buildIndicator` 以及外层 `Stack`。
- `_handleRefresh` 直接透传 `widget.onRefresh`，不再用 `_dragOffset` 否决框架的刷新决定。
- 结果：阈值判定、指针状态、惯性处理、指示器进退全部交还框架，这些正是 5.2 四条根因的
  所在地。

**先做这个，然后按 R6 的五条验收在真机上检验。** 框架标准实现原本就在 armed 后跟随反向
拖拽回落、并在拖拽结束后收起指示器 —— 5.2 的四条根因都源自 wrapper 的并行状态机，
而不是框架本身。所以首选方案有很大概率直接满足全部验收，包括「指示器消失前页面不上滚」。

**只有当真机验收发现框架标准行为确实不满足某条**时，才讨论自定义外观。届时的可用手段
只有离散 `onStatusChange`（可做淡入淡出与状态切换，**做不到跟手连续位移**），
并且必须重新回答「这个视觉细节值不值得」这个问题。

**硬约束**：阈值判定只有一处且必须是框架的；本文件不得再监听 `ScrollNotification`
来推导下拉状态。如果某个视觉诉求只能靠自建状态机实现，**放弃该视觉诉求，不放弃正确性**。

### 5.4 测试处置

`test/pull_to_refresh_test.dart` 现有两条：

- `reverse pull follows the finger before dismissing` —— 它固化的是当前自建实现的具体
  行为路径。**意图**（反向回滑要收起指示器）保留，实现细节按新方案重写；若 5.3 末尾的
  取舍被触发，该用例按新的真实行为改写，不允许为了让它变绿而保留旧状态机。
- `a released armed pull still refreshes once` —— 意图正确，保留。

新增两条（审计明确要求）：

- 反向回滑至阈值以下并释放 → 不触发刷新，且指示器归零。
- 指针抬起后的 ballistic overscroll → 不得重新唤出指示器。

**全部用例的共同要求**：每一条都必须断言指示器的**终态可见性**，而不只是 release 之前的
运动轨迹。旧用例只断言了 release 前的位移，这正是「指示器卡在屏幕上」能通过既有测试的
原因。该要求与 5.1.1 (3) 修订后的 spec 第 6 节一致。

## 六、不做的事

- 不引入新的滚动/刷新第三方库。
- 不新建共享 feed grid 组件（那属于 `settings-productization` 的 C9 在 design 阶段
  自行判断的范围，本 child 不预先替它决定）。
- 不改路由架构、不碰网络/下载/账号代码。
- 不处理详情页以外页面的硬编码中文。
