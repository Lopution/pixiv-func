# 实现反向图片搜索 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start；中间父任务本身不作为实现目标。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 实时研究反向搜图 provider、API/ToS/隐私/限流、challenge 和凭据要求，为每个候选记录 `structuredApi / interactiveWebView / unavailable` 及观测日期。
2. 若仅 WebView 可行，先提交 beta56 结果体验差异与隐私边界审阅；未获批准不进入实现。
3. 定义 ImageInput/validator/preprocessor/owned-temp lifecycle、provider capability 和 typed result/terminal contracts。
4. 实现 picker+SEND 输入、预览/隐私提示、off-main content URI copy、stream upload/cancel/error；获批时再实现同源 WebView file chooser handoff。
5. 实现 structured response mapper、排序/去重、Pixiv hydration 和 safe URL policy。
6. 增加图片炸弹/格式/MIME/权限/云 content URI/大文件、challenge/429/malformed/origin change/cancel/cleanup 测试。
7. 运行 analyze/test/build，在真机分别验证 picker、SEND 及获准的真实 provider。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-reverse-image-search
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- MIME sniff、pixel/file limits、content URI permission。
- stream/resize memory、temp cleanup all terminal paths。
- provider malformed/429/TLS/cancel。
- provider capability/challenge、WebView origin/file chooser（仅在获批路径适用）。
- similarity sort、Pixiv ID dedupe/hydration。
- external URL scheme policy、secret/image logging audit。

## Risky Files and Rollback Points

- lib/features/reverse_image/、image picker/decoder依赖、Android content URI bridge、第三方 provider配置

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

## Execution Record

The common input, lifecycle and provider-capability boundary is implemented and
verified. The result-card/provider success portion remains an explicit external
blocker because the dated provider audit found no approved structured service:
SauceNAO's official API entry returned a managed challenge, TinEye requires a
product-owned `X-API-KEY` and reviewed terms/privacy boundary, and Google Lens
did not expose a stable result-card API at its upload entry. No image, cookie,
credential or provider key was sent during the audit.

The accepted implementation boundary is therefore the shared picker/SEND
preview flow plus a visible terminal unavailable state. Do not mark the
provider-success acceptance criteria complete or add HTML scraping, WebView
handoff, a bundled key, a direct third-party upload route or a mock result until
the missing product and credential decisions are approved.
