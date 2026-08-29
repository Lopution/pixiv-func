# 实现兼容网络与安全账号迁移 — Implementation Plan

## Start Gate

- 上级父任务规划已批准，Q1 已由用户明确允许 App 内部仅 Pixiv 域的兼容传输；任一 child 仍需自身 start approval。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 审阅两个叶子的威胁模型、关证书/反代反例、已修订的 Q1 scope 与 clipboard 的 authenticity/confidentiality 残余风险。
2. 当前 Ugoira 边界结束后先完成 P0 `restricted-compat-network`，再按独立审批完成 `secure-clipboard-account-migration`。
3. 核对 exact-host shared transport、DirectOnly/Automatic隔离、API/OAuth/image/download/WebView逐出口、WebView cleanup和CredentialStore写入。
4. 运行host/TLS/failure taxonomy/network-change/无外部代理大陆样本，以及clipboard reader/writer/checksum重算/target-local replay/expiry/account切换集成回归。
5. 记录父acceptance evidence并归档。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-compat-network-account-migration
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 两个child状态/evidence。
- DirectOnly vs Automatic traffic isolation；system proxy/VPN off 的 OAuth/API/pximg/download/WebView 矩阵。
- clipboard expiry/tamper/replay/clear。
- login/account store integration和secret audit。

## Risky Files and Rollback Points

- 两个child security contracts；父任务不直接编辑业务代码

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
