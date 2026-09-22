# 执行计划：确定性动作后果与返回行为修复（W1）

需求见 `prd.md`，技术设计见 `design.md`，逐项精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实）。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` → `python3 tool/gen_l10n_lookup.py`。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-interaction-outcome-correctness
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：看似不相关的
> 测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec 判定噪声再排查。

本机不可验项（Windows WebView2、TalkBack/Narrator、真机 predictive back 手势、1.3x 大字体
溢出、真机 picker）在 PR body 标"未验证"，不得由 widget test 推断通过。

## 阶段 1：组 A — novel + profile 返回/恢复（R1, R2, R3, R5）

- [ ] **R1**：`lib/features/novel/novel_page.dart` L352 与 L133 的 `maybePop()` →
      `Navigator.of(context).pop()`；保留 L229 `PopScope` 系统返回拦截。
      测试：`test/novel_reader_chrome_test.dart` 补「显式 back → 路由弹出」用例，
      保留「系统返回先关 chrome」用例。
      提交：`fix(novel): 显式返回直接离页，系统返回仍先关 chrome`
- [ ] **R2**：同文件 `NovelInfoSheet` tags Wrap（L567-573）`TagChip` 接 `onTap`：
      `Sheet.of(context).close()` + `openSearchResults(context, NovelSearchQuery(keyword: tag.name))`。
      测试：info-sheet 用例断言 tag 点击关闭 sheet 并发起导航（或至少 onTap 非空+sheet 关闭）。
      提交：`fix(novel): 信息页标签可点击搜索`
- [ ] **R3**：新增 `lib/core/localnovel/read_offset_anchor.dart` 纯函数
      `novelAnchorForReadOffset`（互逆 `_persistCursor`，边界语义见 design.md §二）；
      `local_novel_reader_page.dart` `_LocalNovelReaderBody.build` 传
      `initialAnchor:`。
      测试：纯函数单测（null/0/负、段中、换行边界、越界夹取、内容变短）+
      `test/local_novel_page_test.dart` 恢复用例（回写同值幂等）。
      提交：`fix(localnovel): 打开本地小说恢复上次阅读位置`
- [ ] **R5**：`lib/features/profile/profile_edit_page.dart` `_attemptPop`（L137-167）
      干净表单短路 `pop()`；L183-186 leading tooltip → `backButtonTooltip`。
      测试：`test/profile_edit_test.dart` 新增干净/脏表单两分支用例，
      按文案定位的 finder 同步更新。
      提交：`fix(profile): 资料编辑无改动直接返回，修正返回 tooltip`

## 阶段 2：组 B — search + settings 语义（R4, R6, R7）

- [ ] **R4**：`lib/features/search/search_page.dart` L153-155 `itemCount: tags.length`，
      删除裁剪逻辑与注释。
      测试：`test/search_page_test.dart` 4/5 个 trending tag 全渲染断言。
      提交：`fix(search): 热门标签不再为凑整行丢内容`
- [ ] **R6**：`lib/features/settings/pages/browse_settings_page.dart` "测试"按钮改名
      "应用并测试"（新 l10n key，四语），行为不变。
      测试：`test/settings_test.dart` 断言点击后 repository 值按新文案语义更新。
      提交：`fix(settings): 镜像源测试改名应用并测试`
- [ ] **R7**：`lib/features/settings/pages/translation_credentials_page.dart` `_clear`
      成功后清当前 provider 的 controllers + success 态；`_status` 区分
      success/error（error 用 `colorScheme.error`）；进行中按钮 spinner+禁用。
      测试：`_FakeCredentials` override 下 填充→清除→断言输入为空+success 文案；
      抛 `TranslationCredentialsStoreException` → error 色 + 输入保留。
      提交：`fix(settings): 翻译凭据清除同步输入框并区分成败态`

## 阶段 3：组 C — reverse-image + login WebView（R8, R9）

- [ ] **R8**：`lib/core/reverse_image/reverse_image_controller.dart` 新增 `stopSearch()`
      （`_closed` 守卫 → `++_generation` → cancel token → `ready`（保图）或 `idle`）；
      `pick()` await 后补 generation 校验。页面：进度卡取消改 `stopSearch()`；
      AppBar leading tooltip → `backButtonTooltip`（onPressed 保持 `_cancelAndPop`）。
      测试：controller 单测（searching/preparing/picking 中 stopSearch 三分支 +
      迟到结果丢弃）；页面测试（进度卡取消不离页回 ready；AppBar back → pop+cancel）。
      提交：`fix(search): 反向搜图区分取消搜索与离开页面`
- [ ] **R9**：两端 `login_webview*_page.dart`：可恢复错误卡加"重新加载"（`reload()`）；
      致命错误卡"重新打开"改"重新登录"——`beginSession()` 幂等重启 +
      `loadRequest(authorizeUrl)`；`create=true` 致命态只给"重新加载"+关闭。
      会话重启逻辑抽 `lib/features/login/` 共享 helper（非公共基件）。
      新 l10n key：`loginReload`、`loginRestart`（四语）。
      测试：`test/login_navigation_test.dart` 三用例（reload 调用+未 pop；
      beginSession 二次调用+新 URL+`_fatal` 清除；create=true 无"重新登录"）。
      桌面路径标"未验证"。
      提交：`fix(login): 授权 WebView 动作名称与后果一致`

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec 判定）；
      `git diff --check` 干净；`task.py validate` 通过。
- [ ] PR：`gh pr create --fill`；CI 绿后 `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后提交）。

## 边界（不做）

- 阅读器设置面板/章节进度对齐（W5）；热门标签 Wrap/搜索层级（W2）；设置信息架构（W8）；
  WebView 错误层级布局统一（W9）；任何 `lib/app/` 公共基件；网络/数据层语义。
