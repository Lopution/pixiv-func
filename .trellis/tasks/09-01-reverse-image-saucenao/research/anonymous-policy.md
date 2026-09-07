# SauceNAO 匿名调用复核（2026-09-07）

R1 硬前置：实现前必须复核 SauceNAO 当前是否仍允许不带 `api_key` 的调用。
本次复核在 child 开工时完成，结论：**D1 前提成立——HTML 表单路径匿名可用；JSON API 匿名不可用。**

## 方法

- 只发一次请求：本机生成 64×64 合成噪声 PNG（不是任何用户图片），以浏览器 UA 对
  `https://saucenao.com/search.php` 做 `multipart/form-data` POST（字段 `file`、`frame=1`、
  `hide=0`、`database=999`），不带 cookie、不带任何账号信息。
- 读取 `https://saucenao.com/`、`/status.html`、`/legal.html`；`user.php?page=search-api`
  仍需登录（规划阶段已知 403），限额数字取自 SauceNAO 自己渲染的超限页文案与公开
  客户端库对该页面的转录。

## 结果

| 项 | 观察 |
|---|---|
| `search.php` 匿名 POST | HTTP 200，`text/html; charset=UTF-8`，21,321 B，无重定向 |
| 页面内容 | 8 个 `resulttable` 结果块（`resultsimilarityinfo` 48.41% / 47.40% / 46.30% …），另有「Low similarity results have been hidden」折叠区；无登录、无 CAPTCHA、无 Cloudflare 挑战 |
| 页面第三方脚本 | `<!-- Cloudflare Web Analytics --><script defer src='https://static.cloudflareinsights.com/beacon.min.js' …>`（正常结果页就含 3 处 "cloudflare" 字样） |
| 结果链接形态 | 绝对 `https://` 外链（deviantart / imdb / 反查工具 yandex、tineye、trace.moe），站内相对链接 `donate.php`、`user.php`、`options.php`，缩略图 `/userdata/<id>.png`、`images/static/...` 相对路径 |
| JSON API（`output_type=2`）匿名 | 服务端返回 403 `{"header":{"status":-1,"message":"The anonymous account type does not permit API usage."}}`（公开客户端库 issue 记录，与 `user.php?page=search-api` 需注册一致）。因此零配置只能走 HTML/WebView 路径，`ReverseImageResultMapper.fromSauceNaoJson` 在匿名模式下没有数据源 |
| 未注册限额 | 按 IP 计：每 30 秒 4 次、每日 150 次（SauceNAO 超限页原文：`Daily Search Limit Exceeded.` / `your IP has exceeded the unregistered user's daily limit of 150 searches`；短窗文案 `Search Rate Too High.`）。正常结果页不显示剩余额度，只在超限时出现 |
| 服务条款 | `legal.html`：上传图片保存不超过约半小时；按 IP 统计搜索次数；多账号/多 IP 绕限会被封 |

## 对实现的影响

1. **修正一处会让所有真实搜索失败的启发式**：`SauceNaoWebViewProvider._classifyHtml` 原来把
   `contains('cloudflare')` 当作挑战页标记；真实结果页含 Cloudflare Web Analytics beacon，
   因此每次成功搜索都会被判成 `providerUnavailable`「challenge page」。现改为只匹配挑战页
   专有标记（`cf-chl-`、`cf_chl_opt`、`/cdn-cgi/challenge-platform/`、`<title>Just a moment...`、
   `Attention Required! | Cloudflare`、`Checking your browser before accessing`、captcha 组合）。
   脱敏后的真实结果页固定为 `test/fixtures/saucenao/anonymous_result_page.html`
   （上传文件名与 beacon token 已替换），回归测试在修复前失败、修复后通过。
2. 限流文案补 `search rate too high`（30 秒窗）；`search limit` 已覆盖 `Daily Search Limit Exceeded`。
3. 结果页以 `loadHtmlString(html, baseUrl: https://saucenao.com/search.php)` 加载，相对链接与
   缩略图按 saucenao.com 解析；`SauceNaoNavigationPolicy` 站内放行、Pixiv 走 `IntentRouter`、
   其它 HTTPS 交外部 launcher、非 HTTPS 拒绝——与观察到的链接形态一致。
4. 限额数字不进 UI（PRD Open Questions），只透出实际响应的等待信息（429 `retry-after`
   或超限页文案 → `rateLimited` 可重试）。

## 事实有效期

外部事实会变。若 SauceNAO 关闭匿名 HTML 搜索或对表单加挑战，`_classifyHtml` /
403 分支会把它变成可见的 `providerUnavailable`，不会伪装成空结果；此时回到用户重新决策 D1，
不得改成「让用户填 key」。
