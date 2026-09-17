# 设计：多引擎以图搜图

## 架构

```
ReverseImageSearchPage                     lib/features/search/
  ├─ engine chips（ready/failure 态可切换）
  ├─ _ControlledResultWebView              webview_flutter（移动）/ InAppWebView（桌面）
  │     ← ReverseImageSearchWebView(html|resultUrl)   SauceNAO / IQDB 结果
  └─ _UploadWebView (InAppWebView, 全平台)
        ← ReverseImageSearchWebUpload(uploadPageUrl)  Ascii2D / TinEye
        └─ Android: armed content-Uri 一次性回传
        └─ Desktop: 系统文件选择器 + 提示

ReverseImageSearchSession                  lib/core/reverse_image/
  ├─ platform: ReverseImageInputPlatform
  ├─ engines: Map<ReverseImageEngine, ReverseImageProvider>
  └─ selectedEngine（可切换，search() 时取）

Provider impls
  ├─ SauceNaoWebViewProvider            现状保留（headless）
  ├─ IqdbWebViewProvider                headless multipart → HTML
  └─ WebViewUploadProvider(engine)      Ascii2D/TinEye：校验输入 → WebUpload outcome
```

## 引擎契约

| 引擎 | 模式 | 端点/上传页 | 约束 | CF |
|---|---|---|---|---|
| SauceNAO | headless | `POST saucenao.com/search.php` field `file` | 全局 10MB/4 格式 | 有（分类器覆盖） |
| IQDB | headless | `POST iqdb.org/` field `file` → 200 HTML | jpeg/png/gif、≤8MiB、≤7500×7500 | 无 |
| Ascii2D | webview-upload | 上传页 `ascii2d.net/`，站内 `*.ascii2d.net` 放行 | 全局 | 高 |
| TinEye | webview-upload | 上传页 `tineye.com/`，站内 `/search/{key}` 放行 | 全局 | 高 |

headless 统一语义（IQDB 沿用 SauceNAO 模式）：
- 200 且 `content-type: text/html` → 挑战页/限流分类 → `ReverseImageSearchWebView(html, baseUrl)`
- 3xx → location 在引擎 host 白名单 → `resultUrl`
- 429 → `rateLimited`；403 → `challenge`；其它 → `providerUnavailable`/`network` retryable
- 引擎级输入不满足 → `unsupportedInput`（新增失败码），不发请求

## WebView 自上传管线（Ascii2D/TinEye）

1. `search()` → provider 校验输入 → 返回 `ReverseImageSearchWebUpload(engine, uploadPageUrl, observedAt)`；图片文件继续由 controller 持有（webview 上传完成后随 close/cancel 释放）。
2. 页面渲染 `_UploadWebView`（InAppWebView）：加载引擎上传页 + 顶部提示条「点页面的上传按钮，已备好所选图片」。
3. 用户点页面自带的上传控件 → 平台 `onShowFileChooser`：
   - **Android**：`armReverseUpload(path)` channel 方法在打开 WebView 前置位——校验 path 必须位于 `cacheDir/reverse_image_inputs/`，`FileProvider.getUriForFile` 生成 content Uri 存入 armed 槽；vendored 插件 `InAppWebViewChromeClient` 新增 `armedFileChooserUris` 静态字段，onShowFileChooser 时一次性 take 回传（Shaft `reverseUploadArmed` 语义）；未 armed 走原系统选择器逻辑。
   - **桌面**：WebView2 无文件选择器拦截 API → 系统选择器正常弹出，用户重选同一图片（提示条说明）。headless 不在桌面重试——保持行为一致可见。
4. 上传后结果页在同一 WebView 内渲染，导航策略按引擎 host 白名单。

**不做的方案**：JS `input.files = DataTransfer` 注入（站点 DOM 耦合、脆）；JS 自动 click 触发选择器（Chromium 需用户手势，Shaft 真机证伪）。

## 关键改动

1. `reverse_image_provider.dart`：`ReverseImageEngine` 枚举 + `ReverseImageEngineSpec`（displayName、mode、uploadEndpoint/pageUrl、webViewHosts、resultBaseUrl、maxBytes、allowedMimes）+ `unsupportedInput` 失败码 + `ReverseImageSearchWebUpload` outcome。
2. `iqdb_provider.dart`（新）：headless multipart provider，挑战/限流分类逻辑与 saucenao 共享提取。
3. `webview_upload_provider.dart`（新）：校验输入 → `ReverseImageSearchWebUpload`。
4. `sauce_nao_navigation_policy.dart` → `reverse_image_navigation_policy.dart`：`decide(uri, allowedHosts)` 参数化，引用点迁移。
5. `reverse_image_controller.dart`：session 增 `engines`/`selectedEngine`；`selectEngine()`；`search()` 失败路径保留 `_input` 且 failure 态回写 `input`。
6. `reverse_image_platform.dart` + `ReverseImageInputChannel.kt`：`armReverseUpload(path)`/`disarmReverseUpload()`；`file_provider_paths` 加 `reverse_image_inputs/`。
7. vendored `plugins/flutter_inappwebview_android`：`InAppWebViewChromeClient` 静态 armed 槽 + onShowFileChooser 短路。
8. `reverse_image_search_page.dart`：引擎 chips、`_UploadWebView`、提示条。
9. `AppSettings.reverseImageEngine`（缺省 sauceNao，未知值宽容回落）。
10. l10n：引擎名专有名词不译；提示/不可用原因键四语言。

## 数据流

- 引擎选择 → `session.selectedEngine` → `search()` 按引擎 provider → outcome。
- webview-upload outcome → 页面先 `armReverseUpload(input.path)` 再建 WebView（armed 只被消费一次）。
- 输入保留：success/webview outcome 释放 `_input`；failure 保留（`prepare/cancel/close` 统一释放）。

## 回滚

引擎集合退回 `{sauceNao}` 即还原；armed 槽未置位时 vendored 插件行为与上游一致。
