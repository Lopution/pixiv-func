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

- [x] `profile_header_delegate.dart` 按 PixEz 的 flexible-space 分层：背景、展开身份区
      和 pinned toolbar 分开渲染。
- [x] 展开身份区使用固定 52dp 半径头像（直径 104dp），头像与名字固定留白并一起
      滚出；不再把头像缩进 toolbar，也不在临界点重复挂第二个头像。
- [x] 背景图单独按 `backgroundOpacity` 淡出，头像不继承背景透明度；身份详情在 toolbar
      淡入前退出，避免中间帧头像/名字冲突。
- [x] toolbar 标题独立交叉淡入，保留返回、分享、限制和编辑操作；`topInset` 计入
      pinned header 的最小高度，控件不会进入状态栏区域。
- [x] `test/user_profile_test.dart` 覆盖有头像与无头像用户的滚动过程，断言头像与名字
      不重叠、折叠后展开头像消失且 toolbar title 存在；现有无 settings 入口断言保留。
- [x] `flutter analyze` 与 `flutter test` 通过。

**Review gate**：不停下。过渡观感（展开态 / 中间态 / 折叠态，以及无头像用户的同样三态）
无法由 widget test 判定，列入交付时的用户真机验证清单。几何插值本身由单测覆盖。
**Rollback point**：阶段 3 单独成一个提交。

## 阶段 4：下拉刷新收敛（U1，P0）

> 本阶段单独实施、单独验收。`lib/app/pull_to_refresh.dart` 被全部 feed 页共用，
> 且 `09-01-settings-productization` 的 C9 续拉行为要以本阶段结果为地基。
>
> **先读 `design.md` 5.1**：这里记录了旧契约为何被真机行为证伪；当前可执行契约位于
> `.trellis/spec/frontend/component-guidelines.md` 的 `Shared Pull-to-Refresh Contract`。
> 本阶段同时更新代码与 spec，最终实现以 `easy_refresh` 共享 wrapper 为准。

- [x] **先做最简方案**：`lib/app/pull_to_refresh.dart` 删除自建 refresh 状态机，改为
      `EasyRefresh` + `MaterialHeader`，普通列表采用 `position: above`、`safeArea: true`，
      nested tab 采用 `position: locator`、`safeArea: false` 并配套 `HeaderLocator`，两者
      都启用 `clamping: true` 与主题色配置；阈值、指针生命周期和回弹由组件统一管理。
- [x] 普通 feed 继续使用共享 `PullToRefresh`；用户页保留外层 `NestedScrollView`，每个
      实际 feed tab 使用一个 `isNested: true` 的 locator wrapper，并在列表首项放置
      `HeaderLocator`（sliver 使用 `HeaderLocator.sliver()`），不再重复包裹整个外层。
- [x] 反向回滑、有效刷新、未达阈值取消、指针抬起后的 ballistic overscroll，以及嵌套
      `NestedScrollView` tab body locator 路径各有回归断言；所有场景都检查最终 indicator
      已隐藏。
- [x] **修订 spec**：`.trellis/spec/frontend/component-guidelines.md`。
      `PullToRefresh` 签名记录可选 `isNested`；共享契约改为可观察行为，明确阈值只能由
      一处组件持有、反向回滑先收 indicator、指针抬起后的运动不得重新唤起，并要求测试
      断言终态可见性。普通列表与 `NestedScrollView` tab body locator 的接法均记录在该契约中。
- [x] 重写 `test/pull_to_refresh_test.dart`：覆盖普通列表与嵌套滚动，所有用例断言指示器终态可见性。
- [x] 保留 `a released armed pull still refreshes once`（改名并补上终态断言）。
- [x] 新增：反向回滑至阈值以下并释放 → 不触发刷新且指示器归零。
- [x] 新增：指针抬起后的 ballistic overscroll → 不得重新唤出指示器。
      额外断言「甩动确实越过了顶边」（最深 `pixels < 0`）且最终停在 `pixels == 0`，
      否则甩动没够到顶边时用例会空过、什么都没证明。
- [x] 新增（PRD 第二条）：`reversing an armed pull retracts the indicator before
      the list scrolls` —— 指示器在屏期间断言 `pixels <= 0`，收完后断言 `pixels > 0`。
      **该用例必须用越过阈值的下拉**（10×40=400px）。初版用 160px 写过一次，
      那是未 armed 的另一条框架分支，属于假绿；发现后已改。
- [x] 全量回归：`flutter test`（下拉刷新是共享基建，必须跑全量而非单文件）；末轮
      `/opt/flutter-3.47.0/bin/flutter test` 结果为 `+541: All tests passed!`。
- [x] `/opt/flutter-3.47.0/bin/flutter analyze` 通过（`No issues found!`）。
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

**Review gate**：不停下。四个阶段全部完成后，把上述五个场景连同阶段 3 的过渡观感
一起交用户在手机上验证。
**Rollback point**：阶段 4 单独成一个提交（代码与 spec 修订同一个提交，不拆开）。

## 收尾

- [x] 最后一次全量 quality check（覆盖四个阶段的全部改动，不只最后一段）：
      `/opt/flutter-3.47.0/bin/flutter analyze` 输出 `No issues found!`；
      `/opt/flutter-3.47.0/bin/flutter test` 输出 `+541: All tests passed!`；
      `git diff --check` 通过。
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
- 刷新只使用已批准的 `easy_refresh` 依赖，不再引入其它滚动 / 刷新库。
