# 实现安全剪贴板账号迁移 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 完成clipboard transfer威胁模型和标准envelope方案评审，明确残余风险。
2. 定义version/schema/size/time/nonce/integrity parser与ReplayStore。
3. 实现Android sensitive clipboard/conditional auto-clear adapter。
4. 实现export、explicit paste、strict validation、Pixiv verify和atomic import。
5. 实现原版入口/反馈并增加不宣称安全边界的说明。
6. 增加fuzz/tamper/expiry/replay/clear/account atomicity tests。
7. 运行analyze/test/build，在两台设备或等价设备矩阵验证。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-secure-clipboard-account-migration
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- version/size/schema/base64 parser fuzz。
- expiry/clock skew/nonce replay/tamper。
- conditional clear不覆盖后续clipboard。
- invalid credential/no half account。
- cross-device success、secret/log/storage audit。

## Risky Files and Rollback Points

- lib/core/auth/transfer/、Settings/Login入口、Android clipboard channel、真实refresh credential

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

