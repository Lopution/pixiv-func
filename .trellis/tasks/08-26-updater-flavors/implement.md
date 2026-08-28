# 实现 GitHub 与 F-Droid Updater flavors — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 核验当前Flutter/AGP flavor和Android安装要求，提取beta56About/updater UI。
2. 定义flavors、UpdateCapability、signed manifest schema/public-key rotation、ReleaseInfo/channel policy和manifest overlays。
3. 实现bounded manifest+detached signature fetch、fail-closed verify、GitHub UI与F-Droid no-dependency/no-network/no-button behavior。
4. 接入single-flight DownloadManager、exact size/SHA-256/package/installed-signer verifier、download state reattach和InstallerAdapter。
5. 增加valid/invalid/missing signature、unknown schema/channel、oversize、version/flavor/redirect/download/install/cleanup tests。
6. 构建两个flavor debug/release，审计依赖与merged manifest，并在API36验证GitHub安装拒绝/允许流程。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-updater-flavors
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- version/prerelease/malformed/429/offline。
- flavor dependency and no F-Droid network path。
- merged manifest permission assertions。
- APK hash/package/signature/redirect。
- signed manifest/public key/size/hash/package/installed signer、concurrent check/apply和restart reattach。
- FileProvider/install permission/deny/cancel/cleanup。

## Risky Files and Rollback Points

- android flavors/manifests/build.gradle.kts、About UI、DownloadManager APK sink、FileProvider、release config

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
