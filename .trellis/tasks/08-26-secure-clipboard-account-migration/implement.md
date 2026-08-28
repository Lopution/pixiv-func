# 实现安全剪贴板账号迁移 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 完成clipboard reader/writer、accidental corruption和target-local replay威胁模型，明确零交互方案不提供authenticity/confidentiality。
2. 定义version/schema/size/time/nonce/checksum parser与ReplayStore，错误类型不得超出实际保证。
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
- checksum重算无法抵御malicious writer的文档/测试边界；不得出现“authenticated/encrypted transfer”伪声明。
- conditional clear不覆盖后续clipboard。
- invalid credential/no half account。
- cross-device success、secret/log/storage audit。

## Risky Files and Rollback Points

- lib/core/auth/transfer/、Settings/Login入口、Android clipboard channel、真实refresh credential

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

## Execution Record

### Implemented

- Added bounded version-1 `TransferEnvelope`, strict parser, explicit
  unkeyed-corruption/replay threat boundary and secure target-local replay
  storage.
- Added exact-host pre-import verification with supplied credential, optional
  refresh, server-authoritative profile metadata and atomic account import.
- Added Android sensitive clipboard channel, five-minute conditional clear,
  explicit Settings/Login entry points, localized residual-risk wording and
  regression tests for account-store rollback.

### Compiled and tested

- Focused transfer/account/network tests: 39 passed.
- Changed-file `flutter analyze`: passed with no issues.
- Full test run: 347 passed and the existing VM-only
  `test/icon_font_test.dart` event-channel `MissingPluginException` failed;
  the isolated reproduction is documented in
  `research/implementation-evidence.md`.
- `flutter build apk --debug`: passed; APK SHA-256 is
  `1b982a7fd811d635dc371c517ad426f0419393f1969cd48b5fd2d7f8bfb11bc8`.
- `./gradlew :app:compileDebugKotlin --no-daemon`: passed.

### Device-tested

- `MuMu emulator-tested, not physical-device-tested` on verified MuMu
  `127.0.0.1:16384`, API/SDK 35, proxy `null`, validated Wi-Fi/NAT and no
  VPN according to the required preflight.
- Installed and cold-launched the debug APK. Existing real signed-in account
  exported through Settings long press; Login help displayed the warning and
  paste entry; a real-account recognized transfer attempt was exercised with
  no credential output, and the account remained after force-stop/relaunch.
- API 36, a second independent device and physical-device coverage remain
  unverified; API 35 results are not represented as API 36 evidence. The real
  account login/read chain is working and is not a blocker.

See `research/clipboard-threat-model.md` and
`research/implementation-evidence.md` for the exact residual-risk and
verification boundaries.
