# 复刻完整 Settings — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 建立beta56 settings inventory、页面顺序、default和当前consumer矩阵。
2. 定义versioned AppSettings/Repository/migration/secret references。
3. 迁移现有guide/language/theme并实现可靠awaited writes/error recovery。
4. 实现Settings主/子页面、账号卡、typed providers和service integration。
5. 增加defaults/migration/corruption/persistence/provider/UI/i18n/theme测试。
6. 运行analyze/test/build，在真机验证重启、迁移和consumer即时更新。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-settings-parity
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- defaults和current-key migration。
- per-field corruption/write failure。
- secret separation和日志扫描。
- theme/language/restart以及consumer provider updates。
- 页面顺序/路由，无空onTap。

## Risky Files and Rollback Points

- lib/core/settings/、SharedPreferences keys/migration、Settings UI/routes、多个service provider contracts

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

