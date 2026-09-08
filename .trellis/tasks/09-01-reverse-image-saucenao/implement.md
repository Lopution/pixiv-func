# 执行计划：SauceNAO 反向搜图 transport

## 开工条件

- [x] 匿名 policy 已核实（2026-09-02 决策记录；**2026-09-07 开工复核** `research/anonymous-policy.md`）：
      HTML `search.php` 匿名可用（实测 200 结果页）；JSON API 匿名被拒（403 "anonymous account
      type does not permit API usage"）；未注册按 IP 每 30 秒 4 次 / 每日 150 次；超限为 429
      `retry-after` 或超限页文案。
- [x] 用户确认 D1：零配置 WebView 方案，不要求用户 key/注册。
- [x] 读取 parent R4/R5/R6 与现有 reverse-image 测试；本 task 不新建 provider registry。
- [x] `task.py start` 前完成 `design.md`/`implement.md` review。

## 阶段 0：事实与 fixture

- [x] 固定并记录匿名 endpoint、multipart 字段、成功 HTML、无匹配页面、限流响应和 HTML 挑战页。（真实成功页 fixture `test/fixtures/saucenao/anonymous_result_page.html`；限流 / 挑战 / 无匹配为文案级 fixture，见 `sauce_nao_provider_test.dart`；复核发现并修复 `cloudflare` 误判）
- [x] 所有 fixture 脱敏，不提交用户图片、cookie、token 或 key。（上传文件名、beacon token、CDN `auth`/`exp` 值已替换为 `REDACTED`；探针图为合成噪声；2026-09-08 check 复核后补齐）
- [x] 为响应大小、Content-Type、取消和超时建立可注入的 fake transport。（`MockClient` 注入；大小 / Content-Type / 取消用例已在测试中）

## 阶段 1：WebView transport

- [x] `interactiveWebView` capability（enabled、observedAt、reason 可诊断）。
- [x] `OwnedReverseImageInput` 流式 multipart 上传（file 字段），取消/文件缺失/大小边界覆盖。
- [x] 受控 WebView：站内导航放行；Pixiv 作品/作者链接 → 原生详情/用户页；其它 HTTPS →
      外部 launcher；非 HTTPS 拒绝（`SauceNaoNavigationPolicy` 单测覆盖）。
      **2026-09-08 补**：SauceNAO 实际输出 `member_illust.php?mode=medium&illust_id=` / `member.php?id=`，
      Pixiv 自身链接可带 `/en/` 前缀——`IntentRouter` 原先都不认，全部外跳；已补映射并加测试。
- [x] 429/retry-after、非 HTML 挑战、超时、取消、provider 拒绝均有分类映射。
      **2026-09-08 补**：`retryAfter` 透传到 UI（429 精确秒数；`Search Rate Too High` 页 = 30 s 窗口）；
      `Daily Search Limit Exceeded` → `dailyLimit`（可重试、无倒计时）；挑战页/403 → `challenge`，
      文案不再声称「图片未上传」；无匹配页 → 空成功（显示「无结果」而非失败）。
- [x] 空成功禁止：非 HTML/超限/无内容 → 分类失败（无匹配页除外：显式空成功）；输入 exactly-once 释放沿用 controller（成功路径亦有测试）。
- [x] 阶段门：sauce_nao_provider_test（16）+ navigation policy test（6）+ intent_router_test（9）通过。

## 阶段 2：Controller/UI 接线

- [x] 生产 wiring 替换为 `SauceNaoWebViewProvider`（`reverse_image_search_page.dart`
      默认 provider）。
- [x] picker/ACTION_SEND 两入口、隐私提示、取消流程保留；i18n 四语言补齐
      （reverseIntro/rateLimited/pageLoadFailed）。
- [x] 限流/凭据/网络失败分类可见；外链走安全 launcher。
- [x] widget/route 测试：reverse_image_search_page_test（5，全部注入离线 provider——原先会构造真实 provider 打到 saucenao.com）、reverse_image_search_test（12）全过；
      WebView 导航策略纯函数单测覆盖 Pixiv/外链/非 HTTPS。
- [x] 阶段门：无空成功路径，失败原因可见。

## 最终验证

- [x] `flutter analyze`（No issues）
- [x] `flutter test`（全量 570+；2026-09-08 check：654+381 通过，仅 WSL loopback 已知超时项单跑通过）
- [x] `git diff --check`（提交前统一执行；fixture 尾随空白已清）
- [ ] 真机：Pixiv 截图命中并进详情、非 Pixiv 结果打开浏览器、限流等待、挑战页失败、隐私提示。

## 回滚点

1. 事实/fixture；2. transport；3. controller/UI wiring。事实推翻 D1 时保留明确的
`UnavailableReverseImageProvider` 失败路径，不伪装成可用。
