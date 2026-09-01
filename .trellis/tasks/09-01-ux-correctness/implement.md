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

真机 / 模拟器验证（每阶段一次，产出截图）：

```bash
flutter build apk --debug --flavor github
adb install -r build/app/outputs/flutter-apk/app-github-debug.apk
adb exec-out screencap -p > .trellis/tasks/09-01-ux-correctness/research/screenshots/<名称>.png
```

截图命名沿用既有惯例：描述性 kebab-case，可带设备或时间后缀
（参考 `archive/2026-09/08-29-replica-v1-completion/research/screenshots/`）。
执行前先 `adb devices` 确认设备已连接。

## 阶段 1：低耦合垂直修复（U9 + C11 + C20）

- [ ] **U9**：`lib/features/illust/detail/illust_detail_page.dart:571-609` 的作者 `Row`
      外层加手势包裹，回调 `showUserPage(context, entity.user.id)`。
      不移动 `Key('illust-author-avatar')`，不改布局尺寸与间距 —— spec
      （`component-guidelines.md` 第 4 节）要求该头像保持 48px slot 与占位符不变。
- [ ] **C11**：`lib/features/login/login_page.dart:117-123` 的 loading 分支改为进度指示；
      error 分支改为错误说明 + 重试，重试执行 `ref.invalidate(settingsProvider)`。
      错误态文案走 `Localizations.localeOf(context)` + `ReplicaStrings.fromTag`，
      **不得**依赖 `settingsProvider`（该分支拿不到 settings）。
- [ ] **C20**：详情页 9 处硬编码中文迁入 `lib/core/i18n/replica_strings.dart`，
      行号 254 / 619 / 620 / 641 / 807 / 889 / 907 / 930 / 934。
      zh / en / ja / ru 四块全部补齐，带插值的用参数化形式，不拼接翻译片段。
- [ ] 为 U9 补一条 widget test：点击作者区域触发用户页导航。
- [ ] 为 C11 补一条 widget test：settings 出错时渲染错误态且重试可用。
- [ ] `flutter analyze` 与 `flutter test` 通过。
- [ ] 装机截图：详情页作者区域点击前后、登录页错误态、详情页日语与英语各一张。

**Review gate**：截图交用户确认后再进入阶段 2。
**Rollback point**：阶段 1 单独成一个提交。

## 阶段 2：热门标签代表图（U2）

- [ ] `lib/features/search/search_page.dart` 的 `_TrendingTagTile` 改为图文卡片，
      展示 `tag.representative` 的预览图。
- [ ] `onTap` 保持「搜索该标签」语义不变；进入代表作品保留为次要操作。
- [ ] `representative == null` 时退回纯文字形态与现有提示，不显示破图占位。
- [ ] `lib/core/search/search_trending_controller.dart` 加当天首次获取判定：
      同日重复进入搜索页不再请求，跨日重新获取。
      不引入通用页面缓存框架，不为单个 tag 单独发请求。
- [ ] 补测试：同日二次进入不触发第二次请求；跨日触发重新获取。
- [ ] `flutter analyze` 与 `flutter test` 通过。
- [ ] 装机截图：搜索页热门标签（有代表图）、点击后的搜索结果页。

**Review gate**：截图交用户确认后再进入阶段 3。
**Rollback point**：阶段 2 单独成一个提交。

## 阶段 3：Profile Header 连续过渡（D7 / U8）

- [ ] `lib/features/profile/profile_header_delegate.dart`：把 `_Avatar` 从
      `_ExpandedProfile`(line 173) 与 `_CollapsedProfile`(line 308) 内部移出，
      提升为 delegate `Stack` 中一个独立层。
- [ ] 头像层**不**被 `backgroundOpacity` 包裹（背景淡出，头像不淡出）。
- [ ] 半径改用 `geometry.avatarRadius`，展开端从 72 调到 48–56（直径 96–112dp），
      折叠端保持 20。具体档位按真机观感定，定完把选择理由写进本文件。
- [ ] 头像位置在展开锚点与折叠锚点之间按 `geometry.progress` 连续插值。
- [ ] 用户名在折叠端出现于 toolbar 居中位置，与展开态名字做交叉淡入淡出。
- [ ] 校验过渡全程头像不与 back button（`canPop` 为真/假两种起点）、
      restrict selector、edit / share 按钮重叠。
- [ ] 为 `ReplicaProfileHeaderGeometry` 的新插值补单测（该类是纯几何快照，已被测试引用）。
- [ ] `test/user_profile_test.dart` 现有 header 用例仍然通过。spec
      （`component-guidelines.md` 第 6 节）要求展开与折叠两态都断言
      `Icons.settings_outlined` 不存在 —— 重排头像层时不得引入设置入口。
- [ ] `flutter analyze` 与 `flutter test` 通过。
- [ ] 装机截图：展开态、过渡中间态、折叠态各一张；另拍一组无头像用户的同样三态。

**Review gate**：截图交用户确认后再进入阶段 4。中间态与无头像用户这两组是 U8 的核心证据，
不能省。
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

- [ ] 删除自建 scroll-notification 状态机：`_onScroll`、`_tracking`、`_dragOffset`、
      `_indicatorOffset`、`_viewportDimension` 及其辅助方法。
- [ ] 删除 `_handleRefresh` 中用 `_dragOffset` 否决框架刷新决定的分支（line 168-172）。
      阈值判定收敛为框架唯一持有。
- [ ] 指示器视觉进度改由框架暴露的刷新进度驱动；本文件不再监听 `ScrollNotification`。
- [ ] 若确认框架无法在不自建状态机的前提下实现「反向回滑连续跟随」，
      放弃该视觉细节、保留框架标准行为，并把该取舍写进本文件与 `design.md`。
      **不允许为这一处观感重新造状态机。**
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
- [ ] 装机验证并录证：慢拉、拉出后回滑、快速甩动、松手后惯性、连续第二次刷新。
      这五个场景是 U1 的真实复现路径，逐个确认。

**Review gate**：本阶段必须由用户在真机上亲自确认全部五个手势场景，通过后本 child 才算完成。
**Rollback point**：阶段 4 单独成一个提交（代码与 spec 修订同一个提交，不拆开）。

## 收尾

- [ ] 最后一次全量 quality check（覆盖四个阶段的全部改动，不只最后一段）。
- [ ] 走 Phase 3.3：判断本轮是否产生值得写进 `.trellis/spec/frontend/` 的约定
      （候选：删除错误 guard 时同步删掉固化该错误行为的测试）。
- [ ] Phase 3.4 提交，Phase 3.5 提示 `/finish-work`。

## 边界

- 不碰网络、下载、账号代码。
- 不新建共享 feed grid 组件 —— 那由 `settings-productization` 在自己的 design 阶段决定。
- 不处理详情页以外页面的硬编码中文。
- 不引入新的滚动 / 刷新第三方库。
