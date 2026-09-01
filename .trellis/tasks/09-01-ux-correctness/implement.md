# 执行计划：真实高频 UX 正确性修复

需求见 `prd.md`，技术设计与根因分析见 `design.md`。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.0/bin:$PATH
```

不要手改 `android/local.properties` —— flutter tool 会用全局配置覆盖它。若构建报
`LicenceNotAcceptedException` 或 SDK 路径解析到 `/usr/lib/android-sdk`，先修 flutter
全局配置（详见该 spec）。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze
flutter test
```

> **全量 `flutter test` 的已知环境噪声**：该 spec 的 Testing Requirements 记录了本机
> WSL 环境约 20% 的 `dart:io` loopback 连接丢失，表现为**看似不相关**的测试文件抛
> `TimeoutException after 0:00:30`，单文件重跑却是绿的。遇到这种失败先按该 spec 判定
> 是否为环境噪声，不要误判成本 child 改坏了共享代码 —— 阶段 4 尤其容易撞上，因为它
> 要求跑全量。

真机验证（2026-09-01 用户决定）：**本 child 不做逐阶段模拟器装机截图留证**，
四个阶段全部实现完成后，由用户在自己的手机上一次性验证。理由：模拟器缺少可用的
Pixiv 网络与真实账号数据，逐阶段截图既拍不到关键状态，也不如用户真机使用直观。

因此每阶段的验收证据 = `flutter analyze` + `flutter test` + 可复现的行为断言；
需要人眼判定的部分（过渡观感、手势跟手）留到用户真机验证时一并确认。
构建命令保留备用：

```bash
flutter build apk --debug --flavor github
```

## 阶段 1：低耦合垂直修复（U9 + C11 + C20）— 已完成 `e6a8fd7`

- [x] **U9**：`lib/features/illust/detail/illust_detail_page.dart:571-609` 的作者 `Row`
      外层加手势包裹，回调 `showUserPage(context, entity.user.id)`。
      不移动 `Key('illust-author-avatar')`，不改布局尺寸与间距 —— spec
      （`component-guidelines.md` 第 4 节）要求该头像保持 48px slot 与占位符不变。
- [x] **C11**：`lib/features/login/login_page.dart:117-123` 的 loading 分支改为进度指示；
      error 分支改为错误说明 + 重试，重试执行 `ref.invalidate(settingsProvider)`。
      错误态文案走 `Localizations.localeOf(context)` + `ReplicaStrings.fromTag`，
      **不得**依赖 `settingsProvider`（该分支拿不到 settings）。
- [x] **C20**：详情页 9 处硬编码中文迁入 `lib/core/i18n/replica_strings.dart`，
      行号 254 / 619 / 620 / 641 / 807 / 889 / 907 / 930 / 934。
      zh / en / ja / ru 四块全部补齐，带插值的用参数化形式，不拼接翻译片段。
- [x] 为 U9 补一条 widget test：点击作者区域触发用户页导航。
- [x] 为 C11 补一条 widget test：settings 出错时渲染错误态且重试可用。
- [x] `flutter analyze` 与 `flutter test` 通过。

**Review gate**：无需停下等确认，直接进入阶段 2。本阶段三项都由 widget test 覆盖。
**Rollback point**：阶段 1 单独成一个提交。

## 阶段 2：热门标签代表图（U2）— 已完成 `f6c9da9`

- [x] `lib/features/search/search_page.dart` 的 `_TrendingTagTile` 改为图文卡片，
      展示 `tag.representative` 的预览图。
- [x] `onTap` 保持「搜索该标签」语义不变；进入代表作品保留为次要操作。
- [x] `representative == null` 时退回纯文字形态与现有提示，不显示破图占位。
- [x] `lib/core/search/search_trending_controller.dart:8`：去掉 `.autoDispose`，
      让 provider 在 App 生命周期内常驻，进出搜索页不再重复请求。
      **不加日期判定，不加持久化缓存**（理由见 `design.md` 三）。
- [x] 补测试：二次进入搜索页不触发第二次 trending-tags 请求。
- [x] `flutter analyze` 与 `flutter test` 通过。

**Review gate**：无需停下等确认，直接进入阶段 3。本阶段正确性由 widget test 覆盖。
**Rollback point**：阶段 2 单独成一个提交。

## 阶段 3：Profile Header 连续过渡（D7 / U8）— 已完成 `af44831`

- [x] `lib/features/profile/profile_header_delegate.dart`：把 `_Avatar` 从
      `_ExpandedProfile` 与 `_CollapsedProfile` 内部移出，
      提升为 delegate `Stack` 中一个独立层。
- [x] 头像层**不**被 `backgroundOpacity` 包裹（背景淡出，头像不淡出）。
- [x] 半径改用 `geometry.avatarRadius`，展开端从 72 调到 **52**（直径 104dp），
      折叠端保持 20。
      取 52 的理由：PRD 给的区间是 96–112dp，52 正是中点；而且它让头像底边仍然落在
      `backgroundHeight + 24`（沿用旧布局的 24px 悬垂量），所以与背景带的重叠观感、
      到名字行的间距都和改动前一致，变的只有头像本身的大小。取 48 会让头像在 252dp
      高的背景带里显得空，取 56 又太接近旧的偏大观感。
      **该档位属于需用户真机确认项。**
- [x] 头像位置在展开锚点与折叠锚点之间按 `geometry.progress` 连续插值。
      头像的行程提前在 progress 0.9 结束（`_avatarSettleProgress`），理由见下条。
- [x] 用户名在折叠端出现于 toolbar 居中位置，与展开态名字做交叉淡入淡出。
      实现：折叠层不再由 `isFullyCollapsed` 二选一渲染，改为按 `collapsedOpacity`
      淡入，并钉在顶部 `minExtent` 高的 toolbar 带内 —— 若让它填满收缩中的盒子，
      标题会从画面中部往上滑，并与移动中的头像相撞。命中测试维持原状：折叠层在
      完全折叠前套 `IgnorePointer`，展开层在完全折叠时依然不构建。
- [x] 校验过渡全程头像不与 back button（`canPop` 为真/假两种起点）、
      restrict selector、edit / share 按钮重叠。
      做法不是靠肉眼：把「头像停位」与「折叠层淡入起点」绑成同一个常量
      （`_avatarSettleProgress == _collapsedFadeStart == 0.9`），折叠层一出现，
      头像必定已停在 `[collapsedLeftInset, +40]`，而标题的 96px 内缩正是为这个位置
      留的。单测按 400 步细采样断言该不变式 —— 粗采样会直接跨过窄重叠窗口。
- [x] 为 `ReplicaProfileHeaderGeometry` 的新插值补单测（该类是纯几何快照，已被测试引用）。
      另加一条 widget test：任意滚动位置上有且只有一个 `CircleAvatar`，且它到 header
      之间不存在 opacity < 1 的祖先（有头像与无头像两种用户各跑一遍）。
- [x] `test/user_profile_test.dart` 现有 header 用例仍然通过（`avatarRadius`
      的期望值按新档位更新）。spec
      （`component-guidelines.md` 第 6 节）要求展开与折叠两态都断言
      `Icons.settings_outlined` 不存在 —— 重排头像层时不得引入设置入口。
- [x] `flutter analyze` 与 `flutter test` 通过。

**Review gate**：不停下。过渡观感（展开态 / 中间态 / 折叠态，以及无头像用户的同样三态）
无法由 widget test 判定，列入交付时的用户真机验证清单。几何插值本身由单测覆盖。
**Rollback point**：阶段 3 单独成一个提交。

## 阶段 4：下拉刷新收敛（U1，P0）

> 本阶段单独实施、单独验收。`lib/app/pull_to_refresh.dart` 被全部 feed 页共用，
> 且 `09-01-settings-productization` 的 C9 续拉行为要以本阶段结果为地基。
>
> **先读 `design.md` 5.1**：现行 spec 契约
> （`.trellis/spec/frontend/component-guidelines.md` 的 Artwork Detail Transition
> Contract）明确规定了当前这套自建状态机，其中「无条件应用每个 `scrollDelta`、
> 不得用 `dragDetails` 过滤」正是惯性唤出与指示器残留的成因。本阶段要**同时改代码和
> 改 spec**，不是让实现回到 spec。

- [x] **先做最简方案**：`lib/app/pull_to_refresh.dart` 换掉自建状态机。
      最终实现是三步，前两步单独都不够（`design.md` 5.3 记了完整推导与反证表）：
      1. 换成 `RefreshIndicator`，阈值 / 指针状态 / 惯性 / 触发全部交还框架；
      2. `ScrollConfiguration` 把 ambient physics 换成 `BouncingScrollPhysics`
         并关掉边缘辉光 —— 下拉距离必须真实存在于滚动坐标系里（负 `pixels`），
         反向手势才有可抵消的余量，这是 PRD 阶段 4 第二条的前提；
      3. 改用 `RefreshIndicator.noSpinner`，指示器由本 wrapper 渲染：位置取自
         `metrics.pixels` 的 overscroll（镜像，不累加），是否绘制取自
         `onStatusChange` 的 `drag` / `armed`。
      框架自带指示器有 armed 地板（`refresh_indicator.dart:521`），越过阈值后
      最低只能收到满程 2/3，所以第 2 步之后指示器仍会与列表一起上移 —— 第 3 步
      是为此而做，不是为了好看。
- [x] 删除全部自建状态机与自绘指示器：`_onScroll`、`_tracking`、`_dragOffset`、
      `_indicatorOffset`、`_viewportDimension`、`_progress`、`_dismissController`、
      `_buildIndicator`、外层 `Stack` 及其辅助方法。
      现文件仍是 `StatefulWidget`，但持有的只有两个 `ValueNotifier`：一个是
      `pixels` 的镜像，一个是框架状态的转存，**都不是判定依据**。
      用 `ValueListenableBuilder` 而非 `setState`，避免每帧重建整个 feed。
- [x] `_handleRefresh` 直接透传 `widget.onRefresh`，删除用 `_dragOffset` 否决框架刷新
      决定的分支（line 168-172）。阈值判定收敛为框架唯一持有。
- [x] **换完先跑真机验收**（见下方五个手势场景）。
      真机验收按用户 2026-09-01 的决定推迟到整个 parent task 完成后统一做；
      本阶段用探针测试在 widget 层逐条核实了实际行为（见下条）。
- [x] 自定义外观的取舍：最终**确实**自绘了指示器，但不是为了外观，而是因为框架
      自带指示器的 armed 地板与 PRD 第二条直接冲突。
      **探针实测（Flutter 3.47）**：
      - `ClampingScrollPhysics` 下 `pixels` 恒为 0，反向 20px → `pixels=20`，
        指示器回落与列表上滚同时发生。
      - 换 bouncing 后，分步下拉 6/8/10/14 × 40px 全部进入 armed（`refresh=1`），
        反向全程 `goneAt=-1` —— 指示器从未离开屏幕，而 `pixels` 照常转正。
        armed 门槛低到 240px 分步下拉即触发，**是真机常态路径**。
      - 改 `noSpinner` + overscroll 驱动后，指示器可见期间 `pixels` 恒 ≤ 0，
        指示器收完才转正；armed 释放仍 `refresh=1`；手指抬起后的回弹不唤出指示器
        （这条需要「只在 `drag`/`armed` 才绘制」，只看 overscroll 会复现该缺陷）。
- [x] **修订 spec**：`.trellis/spec/frontend/component-guidelines.md`。
      改法已在 `design.md` 5.1.1 定稿，**直接照抄那五段**，不要现场重新拟措辞：
      - (1) 第 3 节 `PullToRefresh` 整条替换为纯行为契约 + 「阈值判定只有一处」硬约束
        （含两条新补的契约：指针抬起后的运动不得开始下拉；每次下拉结束指示器必须隐藏）。
      - (2) 第 4 节错误矩阵 `Armed pull reverses before release` 一行扩为三行。
      - (3) 第 6 节测试要求改为必须断言指示器的**终态**可见性，而不只是 release 前的运动。
      - (4) 文件结尾那句保留意图，去掉已证伪的实现指向。
      - (5) 把上述条款连同第 2 节 Signatures 里的 `PullToRefresh` 构造签名，
        一并拆出为独立小节 `## Shared Pull-to-Refresh Contract`；
        *Artwork Detail Transition Contract* 只保留 Hero / 预览 / 头像相关内容。
      逐条对照 5.1 的判定表，不要整节删除。
      五段全部照抄落地，并在其上补了两处 —— 都是本阶段实测后才知道必须写的：
      - 第 3 节增加「指示器在屏期间列表不滚动」这条行为契约（PRD 第二条），
        并说明它要求下拉被存成真实 overscroll；同时写清「读 `metrics.pixels`
        用于渲染是允许的，累加距离 / 判阈值 / 否决框架不允许」——
        这个区分不写明，下次改的人会像我一样把唯一可行解当成违规。
      - 第 5 节测试要求增加「反向回滑用例必须越过 arm 阈值」，
        因为未 armed 的下拉走的是另一条框架分支，那样写的用例是假绿。
- [x] 重写 `test/pull_to_refresh_test.dart`：五条用例，全部断言指示器终态可见性。
- [x] 保留 `a released armed pull still refreshes once`（改名并补上终态断言）。
- [x] 新增：反向回滑至阈值以下并释放 → 不触发刷新且指示器归零。
- [x] 新增：指针抬起后的 ballistic overscroll → 不得重新唤出指示器。
      额外断言「甩动确实越过了顶边」（最深 `pixels < 0`）且最终停在 `pixels == 0`，
      否则甩动没够到顶边时用例会空过、什么都没证明。
- [x] 新增（PRD 第二条）：`reversing an armed pull retracts the indicator before
      the list scrolls` —— 指示器在屏期间断言 `pixels <= 0`，收完后断言 `pixels > 0`。
      **该用例必须用越过阈值的下拉**（10×40=400px）。初版用 160px 写过一次，
      那是未 armed 的另一条框架分支，属于假绿；发现后已改。
- [x] 全量回归：`flutter test`（下拉刷新是共享基建，必须跑全量而非单文件）。
      539 tests all passed，本次没有出现 WSL loopback 噪声。
- [x] `flutter analyze` 通过（No issues found）。
- [x] 把五个手势场景写进交付说明，交用户真机验证（U1 的真实复现路径）：
      1. 慢拉引出指示器；
      2. 拉出后反向回滑 —— 指示器跟随回落，**且回滑期间页面内容不上滚**，
         指示器完全消失后剩余手势才滚动列表；
      3. 快速甩动；
      4. 松手后惯性 —— 不得重新唤出指示器；
      5. 连续第二次刷新 —— 指示器不残留。
      五条全部有 widget test 覆盖，但手感只能由真机判定。
      真机需额外留意**本次引入的观感变化**：下拉时列表内容会跟着手指下移
      （iOS 式回弹），不再是「内容不动、指示器悬浮」。这是实现场景 2 的必要代价。

**测试对旧实现与中间版本的反证**（确认用例不是装饰）：

| 临时换回的实现 | 结果 |
|---|---|
| `HEAD` 版自建状态机 | 反向回滑用例「Found 0」（指示器中途消失）、ballistic 用例「Found 1」（手指离开后被唤出）—— 正是用户报告的两个缺陷 |
| 标准 `RefreshIndicator` + clamping | 核心用例失败：`pixels` 恒为 0，下拉没被存成 overscroll |
| 标准 `RefreshIndicator` + bouncing | 核心用例失败：`pixels=7.47` 时指示器仍在屏上 |

三个设计决策（交还框架 / 换 physics / 换指示器驱动源）因此各自都被证明是必要的。


**Review gate**：不停下。四个阶段全部完成后，把上述五个场景连同阶段 3 的过渡观感
一起交用户在手机上验证。
**Rollback point**：阶段 4 单独成一个提交（代码与 spec 修订同一个提交，不拆开）。

## 收尾

- [x] 最后一次全量 quality check（覆盖四个阶段的全部改动，不只最后一段）：
      `flutter analyze` No issues found；`flutter test` 539 tests all passed。
- [x] 走 Phase 3.3：判断本轮是否产生值得写进 `.trellis/spec/frontend/` 的约定。
      产出了，写进 `quality-guidelines.md` 已有的三个空小节（不新建文件）：
      - Forbidden：不得在框架已持有的生命周期旁再跑一套状态机，阈值 / 指针状态 / 进度
        只能有一处；错误 guard 要删不要叠补丁。
      - Required：spec 只写可观察结果不写实现指令；实测证伪 spec 条款时代码与 spec 在
        同一个提交里改；框架表达不了的视觉就放弃视觉、不放弃正确性。
      - Testing：回归测试必须先对旧实现跑一遍确认会失败；断言终态而非只断言轨迹；
        依赖前置条件的用例要断言该条件真的发生过，否则会空过。
      原候选「删除错误 guard 时同步删掉固化该错误行为的测试」已并入上述条目 ——
      本轮的真正教训比它更靠前：**是 spec 里的实现指令先把错误行为锁死的**。
- [x] Phase 3.4 提交，Phase 3.5 提示 `/finish-work`。
- [x] 交付时给出用户真机验证清单：阶段 3 的过渡三态（含无头像用户）+ 阶段 4 的五个手势场景。

## 边界

- 不碰网络、下载、账号代码。
- 不新建共享 feed grid 组件 —— 那由 `settings-productization` 在自己的 design 阶段决定。
- 不处理详情页以外页面的硬编码中文。
- 不引入新的滚动 / 刷新第三方库。
