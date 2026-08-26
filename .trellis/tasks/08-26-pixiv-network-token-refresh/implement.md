# 建立 Pixiv 网络客户端与 Token 刷新 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 实时核验客户端身份、base URI、headers 和当前可信实现，写 research/provenance。
2. 定义 transport、identity、headers、route policy、ApiError 和 next-page parser。
3. 实现 AccountStore 集成、认证拦截、single-flight refresh 和一次重试。
4. 增加日志脱敏、取消、超时、限流和解析错误路径。
5. 编写并发/多账号/恶意 URL/TLS/取消测试以及 test transport。
6. 运行 analyze、全量测试、debug build，并用真实账号做受控 API/refresh 验证。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-pixiv-network-token-refresh
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 20 并发单 refresh；多账号并行 gate。
- 旧 token 比较、retry 上限和 invalid refresh。
- host/endpoint/query allowlist corpus。
- 取消、timeout、429/retry-after、JSON/schema 错误映射。
- 敏感 header/body 日志审计与严格 TLS。

## Risky Files and Rollback Points

- pubspec.yaml、lib/core/network/、lib/core/auth/token_refresh_gate.dart、AccountStore token 更新接口

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

