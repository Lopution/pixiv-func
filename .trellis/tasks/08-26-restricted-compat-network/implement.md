# 实现受限兼容网络模式 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 实时核验AndroidX WebKit ProxyController/reverse bypass、DoH和当前Pixiv域集合，写威胁模型。
2. 定义host canonicalization、allowlist、route policy、resolver和tunnel contracts。
3. 实现DNS/DoH race、TTL cache、health/取消与strict CONNECT parser/limits。
4. 实现original-host TLS connector和WebView proxy session set/clear/refcount。
5. 接入Login/HTTP client compatibility开关及明确failure UI。
6. 增加域绕过、proxy abuse、TLS、DNS、lifecycle/capability tests。
7. 运行analyze/test/build，在受控网络和API36 WebView验证。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-restricted-compat-network
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- Unicode/suffix/trailing-dot/IP host allowlist corpus。
- CONNECT parser/443/rate/size/loopback only。
- SNI/cert original host and bad cert failure。
- DNS/DoH race/TTL/cancel/fallback。
- ProxyController capability/set/clear/refcount/non-Pixiv bypass。

## Risky Files and Rollback Points

- lib/core/network/compat/、Android local service/WebKit adapter、Login switch、network route policy

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

