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

- [ ] **先做最简方案**：`lib/app/pull_to_refresh.dart` 换成标准 `RefreshIndicator`
      （非 `noSpinner`），用 `color` / `backgroundColor` / `strokeWidth` / `displacement`
      做主题化，采用框架自带指示器。
      已核实框架**没有**暴露连续刷新进度的公开 API（`RefreshIndicatorStatus` 是离散枚举），
      所以「用框架进度驱动自定义图标」这条路不成立 —— 不要尝试，见 `design.md` 5.3。
- [ ] 删除全部自建状态机与自绘指示器：`_onScroll`、`_tracking`、`_dragOffset`、
      `_indicatorOffset`、`_viewportDimension`、`_progress`、`_dismissController`、
      `_buildIndicator`、外层 `Stack` 及其辅助方法。
- [ ] `_handleRefresh` 直接透传 `widget.onRefresh`，删除用 `_dragOffset` 否决框架刷新
      决定的分支（line 168-172）。阈值判定收敛为框架唯一持有。
- [ ] **换完先跑真机验收**（见下方五个手势场景）。框架标准实现原本就跟随反向拖拽回落
      并在拖拽结束后收起指示器，很可能直接全部通过。
- [ ] 仅当某条验收确实不通过时，才讨论自定义外观 —— 届时只能用离散 `onStatusChange`
      （做得到淡入淡出，**做不到跟手连续位移**），并重新评估该视觉细节是否值得。
      **任何情况下都不允许重新引入 scroll-notification 状态机**；
      若某个视觉诉求只能靠它实现，放弃该视觉诉求。取舍结果写回本文件与 `design.md`。
- [ ] **修订 spec**：`.trellis/spec/frontend/component-guidelines.md`。
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
- [ ] 重写 `test/pull_to_refresh_test.dart` 中
      `reverse pull follows the finger before dismissing`：保留意图，按新实现改写；
      若上一条取舍被触发，按新的真实行为改写用例，不得为让它变绿而保留旧状态机。
- [ ] 保留 `a released armed pull still refreshes once`。
- [ ] 新增：反向回滑至阈值以下并释放 → 不触发刷新且指示器归零。
- [ ] 新增：指针抬起后的 ballistic overscroll → 不得重新唤出指示器。
- [ ] 全量回归：`flutter test`（下拉刷新是共享基建，必须跑全量而非单文件）。
- [ ] `flutter analyze` 通过。
- [ ] 把五个手势场景写进交付说明，交用户真机验证（U1 的真实复现路径）：
      1. 慢拉引出指示器；
      2. 拉出后反向回滑 —— 指示器跟随回落，**且回滑期间页面内容不上滚**，
         指示器完全消失后剩余手势才滚动列表；
      3. 快速甩动；
      4. 松手后惯性 —— 不得重新唤出指示器；
      5. 连续第二次刷新 —— 指示器不残留。
      场景 2、4、5 有 widget test 覆盖，但手感只能由真机判定。

**Review gate**：不停下。四个阶段全部完成后，把上述五个场景连同阶段 3 的过渡观感
一起交用户在手机上验证。
**Rollback point**：阶段 4 单独成一个提交（代码与 spec 修订同一个提交，不拆开）。

## 收尾

- [ ] 最后一次全量 quality check（覆盖四个阶段的全部改动，不只最后一段）。
- [ ] 走 Phase 3.3：判断本轮是否产生值得写进 `.trellis/spec/frontend/` 的约定
      （候选：删除错误 guard 时同步删掉固化该错误行为的测试）。
- [ ] Phase 3.4 提交，Phase 3.5 提示 `/finish-work`。
- [ ] 交付时给出用户真机验证清单：阶段 3 的过渡三态（含无头像用户）+ 阶段 4 的五个手势场景。

## 边界

- 不碰网络、下载、账号代码。
- 不新建共享 feed grid 组件 —— 那由 `settings-productization` 在自己的 design 阶段决定。
- 不处理详情页以外页面的硬编码中文。
- 不引入新的滚动 / 刷新第三方库。
