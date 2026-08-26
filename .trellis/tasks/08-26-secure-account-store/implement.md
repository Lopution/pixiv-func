# 建立安全多账号与启动状态 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 定义账号、凭据、认证状态和 repository/store 接口及不变量。
2. 引入并配置安全存储插件，实施版本化 key 与错误分类。
3. 实现 AccountStore hydration、多账号操作和原子/可恢复写入顺序。
4. 将 PixivFuncApp/StartupGate 接入真实 provider，补充明确 loading/error UI。
5. 增加 store 单元测试、冷启动 Widget 测试和秘密泄漏检查。
6. 运行 analyze、test、debug APK；在 Android 上做安全存储重启与删除验证。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-secure-account-store
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 多账号 CRUD/current 恢复与并发操作序列。
- credential 写失败时 metadata 不暴露半成品账号。
- Keystore/存储读取失败产生明确错误和 re-auth。
- Startup 三态、语言/主题与账号 hydration 组合。
- 搜索 SharedPreferences 和日志输出，不含测试 token/cookie。

## Risky Files and Rollback Points

- pubspec.yaml、lib/core/auth/、lib/app/app.dart、lib/features/onboarding/startup_gate.dart、Android secure-storage 配置

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

