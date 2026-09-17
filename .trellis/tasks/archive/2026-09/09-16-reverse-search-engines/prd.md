# 多引擎以图搜图

## Goal

在 SauceNAO 之外接入 IQDB / Ascii2D / TinEye，四引擎可选可切，单引擎失败不阻塞其余。

## 现状核实（HEAD `main` @ PR #29 合并后）

已存在、直接复用的部分：

| 组件 | 位置 | 状态 |
|---|---|---|
| Provider 抽象 | `lib/core/reverse_image/reverse_image_provider.dart` | `ReverseImageProvider` 接口 + `interactiveWebView`/`structuredApi`/`unavailable` 三种 capability + `ReverseImageSearchOutcome`（Success/WebView/Failure）+ 8 类失败码 |
| SauceNAO headless 上传 | `sauce_nao_provider.dart` | multipart POST `search.php` → 服务渲染 HTML → `ReverseImageSearchWebView` outcome；含挑战页/限流分类器与 redirect 校验 |
| 流程状态机 | `reverse_image_controller.dart` | 8 状态 + CancelToken + generation 防串场；`ReverseImageSearchSession` 持有 platform + provider |
| 结果 WebView | `reverse_image_search_page.dart` | 受控 WebView（mobile webview_flutter / desktop InAppWebView），`SauceNaoNavigationPolicy`：站内放行、Pixiv 链接走 IntentRouter 进 app、其它 https 交外部 launcher、非 https 拒绝 |
| 输入管线 | `image_input.dart` | 10MB / 8192px / png·jpeg·gif·webp，MIME-格式一致性校验，临时文件所有权 |
| 传输 | `thirdPartyHttpClientProvider` | 共享 plain `http.Client`（第三方流量专用） |
| 入口 | `search_page.dart` 按钮 + Android ACTION_SEND | `/reverse-image` 路由 + `initialReference` |

## Requirements

- `ReverseImageEngine` 枚举 {sauceNao, iqdb, ascii2d, tinEye} + 每引擎 descriptor（显示名、上传端点、WebView 站内 host 集、baseUrl、输入约束、传输模式）
- 两种传输模式（引擎级属性，不是用户选项）：
  - **headless 上传**（IQDB、SauceNAO 沿用现状）：multipart POST → HTML/redirect → `ReverseImageSearchWebView`
    - IQDB：`POST https://iqdb.org/` field `file` → 200 HTML 结果页。约束：jpeg/png/gif、≤8MiB、≤7500×7500（比全局 10MB 严，需 per-engine 输入门控）
  - **WebView 自上传**（Ascii2D、TinEye——两引擎均在 Cloudflare 后，headless 会撞 JS 质询）：受控 WebView 打开引擎上传页，armed 图片在文件选择器回调时直接回传（Shaft `FragmentWebView` 语义：一次性、等用户点页面自己的上传按钮——Chromium 要用户手势，JS 自动点击证伪过）
    - Ascii2D 上传页 `https://ascii2d.net/`（站内 `/search/…` 放行）
    - TinEye 上传页 `https://tineye.com/`（站内 `/search/{key}` 放行）
- Android 平台管线：`reverse_image_input` channel 加 `armReverseUpload`（path → FileProvider content Uri → 静态 armed 槽）；vendored `flutter_inappwebview_android` 的 `InAppWebViewChromeClient.onShowFileChooser` 加一次性 armed-Uri 短路；`file_provider_paths` 暴露 `reverse_image_inputs/`
- 桌面端（WebView2 无文件选择器拦截 API）：上传页在同一受控 WebView 打开，用户经系统选择器重选图片 + 提示文案；不阻断流程
- 上传模式结果 WebView 用 `flutter_inappwebview`（两个平台统一）；SauceNAO 结果页维持 webview_flutter 现状
- 引擎选择 UI：ready 态引擎 chips + failure 态可切换引擎重试（输入保留，不强制重选图）
- 导航策略泛化：`SauceNaoNavigationPolicy` → 按引擎参数化 host 集，Pixiv 链接进站与外链安全策略不变
- 引擎选择持久化到 AppSettings（最近使用即默认）
- 失败引擎独立报错不阻塞其余；无静默降级、无空成功
- 四语言 l10n；provider 单测 + 控制器引擎切换测试 + widget 测试 + Android channel 单测

## Acceptance Criteria

- [ ] 四引擎可选可切；单引擎失败显示独立错误且可换引擎重试同一图片
- [ ] IQDB headless provider 单测覆盖请求构造/结果分类/错误映射
- [ ] IQDB 输入约束生效（webp/超限 → 该引擎不可用并说明原因，不发出请求）
- [ ] Ascii2D/TinEye 走 WebView 上传：armed 文件在首个文件选择器回调一次性回传；桌面降级为系统选择器 + 提示
- [ ] 结果/上传 WebView 只放行本引擎域名；Pixiv 链接仍进 app；其余 https 外链策略不变
- [ ] `armReverseUpload` 拒绝 reverse_image_inputs 目录外的路径

## Out of Scope

- SauceNAO 改 WebView 上传——现有 headless 路径保留，其 challenge 分类器已覆盖被拦场景；真机大面积被拦时再议
- 引擎并行 fan-out / 结果聚合——各引擎结果面是异构 WebView，不可聚合
- SauceNAO JSON API / TinEye 商用 API key 接入——零配置前提不变
- JS 注入 `input.files = DataTransfer` 方案——站点选择器耦合过脆，弃用（Shaft 已证伪 JS 自动点击，本方案等价风险）
- 新入口（详情页/查看器直搜）——入口维持搜索页 + ACTION_SEND

## 外部契约事实（2026-09-16 核实）

- IQDB 首页明示限制：JPEG/PNG/GIF、max 8192KB、max 7500×7500；无 Cloudflare
- Ascii2D：`/search/file` + `/search/uri`（PicImageSearch/YetAnotherPicSearch 确认）；结果页 `/search/color/{hash}` 与 `/search/bovw/{hash}`；有 headless 被 403 的公开记录（YetAnotherPicSearch #139）
- TinEye：站点自身前端即 POST `/api/v1/result_json/`（Mozilla bugzilla #1895666）；响应 `query.key`/`query.hash`（PicImageSearch 源码）；`searxng` 用 `GET /api/v1/result_json/?url=` 做 URL 搜索
- SauceNAO：本仓 2026-09-07 实测匿名 POST 可用；Shaft 后续观察（#733）称已遇 `cf-mitigated: challenge`——两边按时间/地区均可为真，挑战分类器已覆盖
- pximg 直链需 Referer，引擎侧 URL 拉取会 403——不支持「提交 Pixiv 图片 URL」模式，只上传本地文件

## References

- Shaft `ReverseImage.java`：headless 上传被 Cloudflare 质询的完整记录 → 改 WebView 自上传（本任务不采纳该路径，见 Out of Scope）
- PicImageSearch：ascii2d/tineye 端点契约
- 归档任务 `09-01-reverse-image-saucenao/research/anonymous-policy.md`

## Dependencies

无。
