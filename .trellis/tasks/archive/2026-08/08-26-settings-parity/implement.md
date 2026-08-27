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

## Completion Record — 2026-08-27

- 已完成 `AppSettings` v2、`PreferencesSettingsRepository` 迁移/逐字段恢复、序列化写入和 typed providers；翻译凭据仅保留 `SecretSettingRef`，不进入设置 JSON。
- 已完成 Settings 主页面及账号、主题、语言、翻译、浏览、下载、历史、屏蔽标签、下载任务、About 子页面；Home 的设置 tab 已接入真实页面，未保留文字占位或空操作入口。
- 已接入主题、图片源、预览/查看质量、历史、屏蔽和最大并发下载设置的 consumer；旧版固定 IP 仅为显式 legacy/emergency 选项，默认使用 `i.pximg.net`。
- 自动化验证：`flutter pub get` 通过；`flutter analyze` 通过（No issues found）；`flutter test --concurrency=1` 通过（183 项）；`flutter build apk --debug` 通过；`git diff --check` 通过。
- MuMu 验证：使用 `127.0.0.1:7555` 安装 `build/app/outputs/flutter-apk/app-debug.apk`；Settings 首页、主题页和浏览设置页可打开并显示对应入口、三种主题、图片源选项及质量/历史/屏蔽开关。保留应用数据，未改账号或凭据。
- 范围边界：安全剪贴板、History/Downloader 业务实现和 Updater 仍由对应后续子任务负责；本任务只提供真实设置入口和已存在能力的接线。
