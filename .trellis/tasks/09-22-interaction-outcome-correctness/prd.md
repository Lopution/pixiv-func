# 确定性动作后果与返回行为修复（Roadmap W1）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图）。
规划基线：`main@8067b2d`；产品复核快照 `main@8215fa8`。全部代码断言已由本 leaf
`research/codebase-*.md` 在当前 HEAD 逐条复核。

## Goal

修复九处"动作名称/惯例与实际后果不一致"的确定性错误。本 leaf 只修行为与文案：
不收敛页面层级（W2/W5/W8/W9 的职责）、不预建后续工作包才会使用的公共基件
（父 design.md §4.11）、不改网络/数据层语义。

## Requirements

### R1. 在线小说阅读器显式返回真正离页

现状：`lib/features/novel/novel_page.dart` AppBar back 调 `maybePop()`（L352），被同页
`PopScope(canPop: !_chromeVisible)`（L229）拦死——chrome 可见时显式返回只关 chrome。

- 显式返回（AppBar back、错误/状态 scaffold 的返回）一律 `pop()` 离页。
- 系统返回（predictive back / `handlePopRoute`）保持"先关 chrome"语义不变。

### R2. 小说标签可操作

现状：`NovelInfoSheet` 的 `TagChip`（novel_page.dart L567-573）视觉可交互但无 `onTap`。

- 点 tag：关闭 sheet 并 `openSearchResults(context, NovelSearchQuery(keyword: tag.name))`，
  复用同文件作者 chip 的 `Sheet.of(context).close()` 模式与现有 import。

### R3. 本地小说打开恢复上次位置

现状：`local_novel_reader_page.dart` 持久化 `read_offset` 但从不回灌 `NovelReader`
的 `initialAnchor` 参数——每次打开都回到开头。

- 新增 `readOffset → NovelAnchor` 纯函数（段落边界语义与持久化侧互逆），进入阅读器时传入。
- null/0/负值 → 首页；越界/陈旧 offset → 夹取到末页（自愈语义，见决策 D5）。
- 无 schema 变更；`onAnchorChanged` 恢复后回写同值，幂等。

### R4. 搜索热门标签不再丢内容

现状：`search_page.dart` L153-155 `itemCount: tags.length - tags.length % 3` 为凑整行
丢弃末行不足 3 个的标签。

- `itemCount: tags.length`，删除裁剪逻辑；末行不满的观感让步于内容可达。
- Wrap 重排/层级收敛归 W2，本 leaf 只做最小修复。

### R5. 资料编辑无改动直接返回

现状：`profile_edit_page.dart` `_attemptPop`（L137-167）不区分脏/干净表单一律弹确认框；
leading tooltip 误用 `cancel` 文案。

- `session == null || !session.hasUnsavedChanges` → 直接 `pop()`；脏表单保持现有确认流。
- leading tooltip 改用 `MaterialLocalizations.backButtonTooltip`。
- W3 在该契约上完成布局，本 leaf 只修判定。

### R6. 浏览设置"测试"名称与后果一致

现状：`browse_settings_page.dart` 的"测试"先 `_applyCustomInput` 持久化再探测——
名为测试，实为应用。

- 采用方案 A（改名"应用并测试"，行为不变）；方案 B（真测试语义）见决策 D1，默认不采用。

### R7. 翻译凭据清除同步 UI 且三态可辨

现状：`translation_credentials_page.dart` `_clear`（L131-156）成功后不清输入框；
成功/失败同为 `primary` 色，处理中无反馈。

- 清除成功后清空当前 provider 对应的 controllers，置 success 态。
- `_status` 区分 success/error（error 用 `colorScheme.error`）；进行中按钮显示 spinner 且禁用。

### R8. 反向搜图区分"取消本次搜索"与"离开页面"

现状：进度卡"取消"按钮实际执行离页（`_cancelAndPop`），无法停搜保图。

- `ReverseImageController` 新增 `stopSearch()`：`++_generation` + cancel token +
  回到 `ready`（保留输入图）；`pick()` 补 generation 校验使 picking 中可取消。
- 进度卡取消 → `stopSearch()`（原地停止，不离页）；AppBar back → 离页 + 全量 `cancel()`。
- AppBar leading tooltip 改用 `backButtonTooltip`。

### R9. 授权 WebView 动作名称与后果一致

现状：两端 `login_webview*_page.dart` 可恢复错误无 reload 动作；致命错误"重新打开"
实际只是 pop（不复登）。

- 可恢复错误卡加"重新加载"（`reload()`，非重建会话）；致命错误卡改"重新登录"——
  原地 `beginSession()` 幂等重启 + `loadRequest(authorizeUrl)`。
- `create=true`（注册流，无 PKCE）致命态只给"重新加载"+关闭，不显示"重新登录"。
- 桌面端逻辑抽到共享 helper 或保持对称实现；Windows WebView2 路径无法本地验证，标"未验证"。

## Acceptance Criteria

- [ ] 九项各有 ≥1 个 focused test（新增或更新），既有受影响测试同步修正。
- [ ] 显式 back 与系统 back 双路径测试：novel 页用 `handlePopRoute` + 按钮 tap 钉住
      「命令式 `pop()` 绕过 `popDisposition`」的 SDK 行为。
- [ ] 本地小说锚点：空记录/有效记录/越界/陈旧记录分别测试，含换行边界与往返互逆断言。
- [ ] test/clear/cancel/reload 的持久化副作用断言为零或与文案一致。
- [ ] 每 commit 对应 implement.md 一个勾选框；分支 `task/09-22-interaction-outcome-correctness`。
- [ ] `flutter analyze --no-pub`、相关 `flutter test`、`git diff --check`、
      `task.py validate` 全绿；运行时矩阵中本机不可验项在 PR 显式标"未验证"。

## Decisions（规划定案）

- D1 R6 采用方案 A（改名），不扩大 image allowlist 信任面；若实现期用户改选 B，
  R6 拆为独立 stage。
- D2 R9 致命错误采用原地"重新登录"（`beginSession` 幂等已核实），不只改名保留 pop。
- D3 不加 AppBar 常驻 reload（非 §4.1 硬性要求），reload 在错误卡内可达即可。
- D4 reverse-image ready 态"取消"保留文案不改名（后果确为终止本次输入，可接受）。
- D5 越界/陈旧 readOffset 夹取到末页并回写夹取值（库自愈），不判失效回开头。
- D6 锚点函数放 `lib/core/localnovel/read_offset_anchor.dart`（跨包单测需要公共）。
- D7 桌面 login 页不建 InAppWebView platform fake；会话重启逻辑抽到
  `lib/features/login/` 共享 helper，手机端测试钉语义，桌面标"未验证"。

## Out of scope

- 阅读器设置/章节面板对齐（W5）；热门标签 Wrap 与搜索层级（W2）；设置页信息架构（W8）；
  WebView 错误层级布局统一（W9）；任何公共基件预建（§4.11）。
