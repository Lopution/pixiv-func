# W1 实施草案 — 确定性动作后果与返回行为修复

范围：parent design.md §4.1 九项。原则：只修行为/术语，不预建未来工作包基件（§4.11）；复用现有 app 层控件与状态所有者；可观测错误不吞。

l10n 通用流程（凡涉及新 key）：编辑 `lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` → `python3 tool/gen_l10n_lookup.py`。新 key 一律英文语义命名。

---

## 1. 在线阅读器显式返回（novel_page.dart）

- 目标文件：`lib/features/novel/novel_page.dart`
- 改动点：L352 `onPressed: () => Navigator.of(context).maybePop()` → `Navigator.of(context).pop()`。L133 `_NovelStatusScaffold` 的 `maybePop` 可保留（无 PopScope，语义等价）；若该页可能作为分支首路由（canPop=false 时 maybePop 死按钮），统一改 `pop()` 更稳——**建议两处都改 `pop()`**。
- 不改动：L229–233 PopScope（系统返回拦截保留）；`_hideChrome`；ReaderHandle。
- 状态设计：无新增状态；`_chromeVisible` 继续只服务于系统返回。
- 测试：`test/novel_reader_chrome_test.dart` 新增「chrome 可见时点 AppBar back → 路由弹出、页面销毁」；保留既有「系统返回先关 chrome」用例（`handlePopRoute` 路径）。
- 迁移：显式返回由「关 chrome」→「离页」；系统返回不变。无持久化/迁移。
- 边界：`context.pop()`（go_router）与 `Navigator.pop()` 等价（无 onExit 路由），用 `Navigator.of(context).pop()` 与 profile_edit 保持一致。

## 2. 小说标签可操作（novel_page.dart）

- 目标文件：`lib/features/novel/novel_page.dart`（L567–573 `NovelInfoSheet` tags Wrap）
- 改动点：`TagChip(label: tag.name, translated: tag.translatedName, onTap: () { Sheet.of(context).close(); openSearchResults(context, NovelSearchQuery(keyword: tag.name)); })`
  - `Sheet.of(context).close()` 模式复制自 L557–559 作者 chip；`NovelSearchQuery`/`openSearchResults` 均已 import 于本文件（搜索页 L181 在用同型 query）。
- 复用：`TagChip`（app/widgets/tag_chips.dart）、`Sheet`、`openSearchResults`、`NovelSearchQuery` —— 零新组件。
- 测试：`test/novel_reader_chrome_test.dart` info-sheet 用例补「点 tag → sheet 关闭 + 路由 push」断言（需可观测的 navigator/router；若 push 难断言，至少断言 TagChip.onTap 非 null + sheet 关闭）。
- 迁移：非交互 chip → 可导航 chip；视觉不变（TagChip 自带 ink 响应）。
- 备选（不推荐）：改纯 `Text` 去掉 Card——放弃导航价值，且与同文件 tag 视觉不一致。

## 3. 本地小说 initialAnchor（local_novel_reader_page.dart）

- 目标文件：
  - `lib/features/novel/local_novel_reader_page.dart`（接线）
  - `lib/core/localnovel/` 新增纯函数或就地私有 —— **建议** `local_novel_reader_page.dart` 同文件内静态函数 + 若要跨包单测可放 `lib/core/localnovel/read_offset_anchor.dart`（决策点：私有够用则私有，避免新增公共 API）。
- 改动点：`_LocalNovelReaderBody.build` 传 `initialAnchor: _anchorForOffset(novel.readOffset, text)`。
- 纯函数契约：
  ```dart
  NovelAnchor? novelAnchorForReadOffset(int? readOffset, String text) {
    if (readOffset == null || readOffset <= 0 || text.isEmpty) return null;
    if (readOffset >= text.length) {
      // 陈旧/越界：夹取到末段末尾 → 末页
      final lines = text.split('\n');
      return NovelAnchor('p${lines.length - 1}', lines.last.length);
    }
    var acc = 0;
    final lines = text.split('\n');
    for (var i = 0; i < lines.length; i++) {
      final end = acc + lines[i].length;
      if (readOffset <= end) return NovelAnchor('p$i', readOffset - acc);
      acc = end + 1; // + '\n'
    }
    return NovelAnchor('p${lines.length - 1}', lines.last.length);
  }
  ```
  （与 `_persistCursor` L86–95 正映射互逆；`readOffset` 落点恰在换行处时归下一段 `('p$i',0)`——上述 `<= end` 已实现该语义，与持久化侧「锚点只产生在段首」一致，往返测试验证。）
- 状态设计：无新持久化字段；`read_offset` 列语义不变（字符偏移）。`onAnchorChanged` 恢复后立即回写同一 offset，幂等。
- 测试（`test/local_novel_page_test.dart` + 纯函数单测）：
  - null / 0 / 负值 → 从首页起；
  - 段中 offset → 对应段落所在页；
  - 恰好换行边界 → 下一段首页；
  - `>= text.length` → 末页（且回写被夹取后的 offset，或保持原值——决策点，建议回写夹取值使库自愈）；
  - 内容变短后的陈旧 offset → 不崩、落末页；
  - 恢复后 `updateReadOffset` 被以同值/夹取值调用一次。
- 迁移：首次打开本地小说从「总是开头」→「恢复上次页」。无 schema 变更。
- 注意：`initialAnchor` 仅在首次布局消费（novel_reader.dart L433）——body 是 ConsumerWidget，text 已加载后同步计算即可，无异步竞态；`NovelReaderCommitGate` 无需触碰。

## 4. 热门标签不丢内容（search_page.dart）

- 目标文件：`lib/features/search/search_page.dart` L153–155
- 改动点：`itemCount: tags.length`（删整行裁剪逻辑与注释）。
- 复用：现有 `SliverGrid`/`_TrendingTagTile`。
- 测试：`test/search_page_test.dart` 构造 4/5 个 trending tag → 断言全部渲染；点末位 tag → `NovelSearchQuery` 路由。
- 迁移：末行可能不满 3 格——观感让步于内容可达；若评审要求无空位观感，转 Wrap 方案留给 W2（见 codebase-search-pages.md §4）。
- 注意区分：此 grid 的 `itemExtent`/尺寸由 gridDelegate 控制，末行不满不改变 tile 尺寸。

## 5. 资料编辑返回（profile_edit_page.dart）

- 目标文件：`lib/features/profile/profile_edit_page.dart`
- 改动点：
  - L137–167 `_attemptPop`：`session == null || !session.hasUnsavedChanges` → `Navigator.of(context).pop()`；否则现有 `showAppDialog` 确认路径不变。
  - L183–186 leading tooltip：`context.l10n.cancel` → `MaterialLocalizations.of(context).backButtonTooltip`（与 novel_page L351 同模式）。
- 状态设计：脏状态继续以 `profileSessionProvider`/`hasUnsavedChanges` 为唯一事实源，不复制比较。
- 测试：`test/profile_edit_test.dart`：
  - 现有 dirty 分支用例保留（finder 若按 'Cancel' tooltip 定位需同步改为 backButtonTooltip 定位）；
  - 新增「干净表单点返回 → pop 且无对话框」；
  - 新增「脏表单点返回 → 对话框；确认 → pop；取消 → 停留」完整分支。
- 迁移：干净表单点返回少一次确认弹窗；对话框文案（Discard/Cancel）不动。

## 6. 浏览设置「测试」（browse_settings_page.dart）

- 目标文件：`lib/features/settings/pages/browse_settings_page.dart`；若选方案 B 另触 `lib/core/settings/settings_controller.dart`。
- 方案 A（推荐，最小诚实）：仅改名。`imageSourceTest` key → 新 `imageSourceApplyAndTest`（"Apply and test"/"应用并测试"等四语），`_testMirror` 行为不变。_applyCustomInput 的持久化变为显式后果。
- 方案 B（真测试语义）：
  - `settings_controller.dart` 加 `saveCustomImageSource(String normalized)`（只写 `customImageSource`，不动 `imageSource`——走现有 `copyWith`+`queueSettingsWrite`）；
  - `imageMirrorAllowlistProvider`（L336–343）放行 `settings.customImageSource` 的 host 于 `extraHosts`（与 `imageSource` 的 host 合并）；
  - `_testMirror` 改为：`normalize` → `saveCustomImageSource` → 走 `pixivNetworkFactoryProvider.client(image)` 探针（候选已在 allowlist，保留完整降级梯）。
  - 成功后才引导用户点「保存」真正切换；UI 需要表达「候选已测未启用」——可能需额外状态文案，工作量翻倍。
- 决策点：方案 B 扩大 allowlist 信任面（未选中的 host 也可直连 image purpose）且需新 UI 状态；**默认建议 A**，B 需用户确认。
- 测试：`test/settings_test.dart` 新增「点测试按钮 → repository 值按文案语义变化」：方案 A 下断言 imageSource 已更新（行为=名称一致）；方案 B 下断言 imageSource 不变、customImageSource 更新、探针发起。网络探针需 `pixivNetworkFactoryProvider` override（fake client 返回 bytes）。
- 迁移：方案 A 纯文案；方案 B 下已存 `customImageSource` 自动进入 allowlist（无迁移脚本，provider 读时生效）。

## 7. 翻译凭据清除（translation_credentials_page.dart）

- 目标文件：`lib/features/settings/pages/translation_credentials_page.dart`
- 改动点：
  - `_clear()`（L131–156）成功后：按当前 provider tab 清空对应 controllers（baidu → `_appIdController`+`_appKeyController`；llm → `_llmKeyController`），`_status` 置 success。
  - `_status` 建模：`_status` 由 `String?` 改为 `({String text, bool isError})?` 或并列 `_statusIsError` bool；错误态 `colorScheme.error`，成功态 `primary`（L206–213）。
  - 处理中：触发中的按钮 icon 位换 `SizedBox(16,16,CircularProgressIndicator(strokeWidth:2))`；`_saving` 改名或保留（覆盖 save+clear 两动作）。
  - （可选）`_clear` 成功后重新 `read` 校验 store 已空——不必，删 API 已抛异常。
- 状态设计：三态 = 按钮 spinner（in-progress）/ primary 文案（success）/ error 色文案（failure）；互斥由 `_saving` 保证。
- 测试：新建或并入现有测试文件（`translationCredentialStoreProvider` override `_FakeCredentials`，参照 `test/comment_translation_test.dart` L10–34）：
  - 填充→清除→断言 TextField 为空 + success 文案；
  - fake store 抛 `TranslationCredentialsStoreException` → error 色文案 + controllers 保留输入；
  - 清除进行中按钮禁用/显示 spinner。
- 迁移：纯 UI 行为；无持久化变化。不显示/不回填密钥本体之外的任何秘密。

## 8. 反向搜图取消（reverse_image 两文件）

- 目标文件：`lib/core/reverse_image/reverse_image_controller.dart`、`lib/features/search/reverse_image_search_page.dart`
- controller 改动点：
  - 新增 `Future<void> stopSearch()`：`_closed` 守卫 → `++_generation` → `_cancelToken?.cancel()` → `_input != null ? setState(ready(保留 input/engineFailures)) : setState(idle)`。见 codebase-search-pages.md §8 代码草案。
  - `pick()`（L158–176）：`platform.pickImage()` await 后补 `if (_closed || generation != _generation) return;`（入口捕获 `final generation = _generation`）——让 picking 中的 stopSearch 生效。
  - `prepare`/`search` 的既有 gen 校验已兼容，不改。
- page 改动点：
  - `_progress` 卡取消按钮（L296–299）：`_cancelAndPop` → `() => _controller.stopSearch()`。
  - AppBar leading（L167–171）：tooltip `searchReverseCancel` → `MaterialLocalizations.of(context).backButtonTooltip`；onPressed 保持 `_cancelAndPop`（离页销毁输入正确）。
  - `ready` 态取消按钮（L343–346）：保留 `_controller.cancel`；文案 `searchReverseCancel` 语义上是「丢弃图片」——可改名 `removeImage`/`clear`（决策点；不改名也可接受，因后果确实终止本次输入）。
- 状态设计：`stopSearch` 是 `searching|preparing|picking → ready|idle` 的合法转换；`canceled` 态保留给全量 `cancel()`。
- 测试：
  - controller 单测（`test/reverse_image_search_test.dart`）：searching 中 stopSearch → ready + `_input` 保留 + 迟到 provider 结果被丢弃；preparing 中 stopSearch → idle + copy 完成时删文件；picking 中 stopSearch → picker 返回后状态停 idle。
  - 页面测试（`test/reverse_image_search_page_test.dart`）：searching 进度卡点取消 → 页面未 pop + 回到 ready；AppBar back → pop + cancel 被调。
- 迁移：进度卡「取消」由离页 → 原地停止；ready 态不变。无持久化。

## 9. 授权 WebView 动作统一（login 两文件）

- 目标文件：`lib/features/login/login_webview_page.dart`、`lib/features/login/login_webview_desktop_page.dart`
- 改动点（两端对称）：
  - 可恢复错误卡：加「重新加载」动作（`l10n` 新 key `loginReload`/"Reload"/"重新加载"）→ `_controller.reload()`（桌面 `controller.reload()`）或 `loadRequest(_mainFrameUri)`；保留 `dismiss`。
  - 致命错误卡：`reopen`（pop）→ 「重新登录」（新 key `loginRestart`/"Sign in again"/"重新登录"）原地重启：
    ```dart
    setState(() { _fatal = false; _error = null; _exchanging = false; });
    final session = widget.oauthService.beginSession();      // 内部已 discardSession
    _mainFrameUri = session.authorizeUrl;
    _controller.loadRequest(session.authorizeUrl);           // 桌面: loadUrl(URLRequest(url: WebUri.uri(...)))
    ```
    `create==true` 时无 PKCE：致命态给「重新加载」(signup URL) + 关闭；不显示「重新登录」。
  - 关闭 X 保留 `pop(false)`（离开=销毁会话，dispose 兜底）。
  - （可选决策点）AppBar 加常驻 reload IconButton——对齐 PixEz 模式；若加，两端同加。
- 状态设计：`_fatal/_error/_exchanging/_mainFrameUri` 已够；重启只是复用 `beginSession` 幂等性。不引入新 provider。
- 测试（`test/login_navigation_test.dart`）：
  - HTTP 错误 → 点「重新加载」→ fake controller 记录 reload/loadRequest 调用 + 页面未 pop；
  - 非法回调 → 致命卡 → 点「重新登录」→ `beginSession` 第二次被调（oauthServiceProvider fake/间谍）+ 新 authorizeUrl loadRequest + `_fatal` 清除；
  - `create=true` 致命态不显示「重新登录」。
  - 桌面页：Linux 无 WebView2 → 桌面 widget 测试需要 InAppWebView platform fake；若成本高，至少把「重启会话」逻辑抽到两端共享的纯 Dart helper（如 `login_navigation_decision.dart` 旁 `login_session_restart.dart` 或私有顶层函数无法共享——决策点：允许一个小的 `lib/features/login/` 共享 helper，非公共基件）+ 手机端测试钉语义，桌面标「未验证」。
- 迁移：错误卡按钮文案/后果变化；无持久化。

---

## 横切验收（实现期）

1. 九项各有 ≥1 个 focused test；受影响既有测试同步更新。
2. `flutter analyze --no-pub`、相关 `flutter test`、`git diff --check`、`task.py validate`。
3. 运行时矩阵（320/390/600/840/1200dp + 横屏、1.3x 大字体、中英长文案、触摸/系统返回/键盘/鼠标、TalkBack/Narrator、reduced motion）——本机不可验项在 PR 标「未验证」。
4. 每个 commit 对应 implement.md 一个勾选框；分支 `task/09-22-interaction-outcome-correctness`。

## 显式排除（防 scope 蔓延）

- 本地小说阅读器设置/章节进度面板对齐在线版 → W5。
- 热门标签 Wrap 重排、tag 视觉统一 → W2。
- 设置页信息架构/分组 → W8。
- WebView 常驻动作栏视觉 → 沿用 AppBar，不新造。
- `ResumeAnchor`（下载恢复）不用于阅读进度。
