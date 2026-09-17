# 执行计划：多引擎以图搜图

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] `feat(reverse-image): 引擎模型与 IQDB provider`——`ReverseImageEngine`/`ReverseImageEngineSpec`/`unsupportedInput`/`ReverseImageSearchWebUpload` outcome + 共享 headless 分类提取 + `IqdbWebViewProvider`（8MiB/jpeg-png-gif 门控）+ 单测
- [ ] `feat(reverse-image): WebView 上传通道`——`WebViewUploadProvider`（Ascii2D/TinEye descriptor）+ `armReverseUpload`/`disarmReverseUpload` channel + FileProvider 暴露 `reverse_image_inputs/` + vendored inappwebview armed-Uri 短路 + Kotlin 单测
- [ ] `feat(reverse-image): 引擎切换与导航策略泛化`——session engines map + selectedEngine、controller selectEngine、失败保留输入、`ReverseImageNavigationPolicy` 按引擎 host 参数化、AppSettings.reverseImageEngine + 单测
- [ ] `feat(reverse-image): 引擎选择与上传 UI`——ready/failure 引擎 chips（约束不满足禁用+原因）、`_UploadWebView`（armed + 提示条，桌面降级提示）、四语言 l10n、widget 测试
- [ ] `chore(09-16): reverse-search-engines journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test test/reverse_image_search_test.dart test/reverse_image_search_page_test.dart test/sauce_nao_provider_test.dart` + 新增测试
- `cd android && ./gradlew :app:testDebugUnitTest`（channel 单测，如环境可用）
- `flutter test`（全量）
- `git diff --check`

## 回滚点

每勾独立可 revert。引擎集合退回 saucenao 单引擎即还原；armed 槽不置位时插件行为同上游。

## 风险

- WebView 上传依赖站点表单形态（ascii2d/tineye 改版需跟进），armed 回传是站点无关的机制层，不耦合 DOM。
- 桌面无选择器拦截 → 用户重选图片，提示文案兜底；不静默退回 headless。
- vendored 插件改动极小（静态槽 + 短路），上游升级需保留此 patch。
