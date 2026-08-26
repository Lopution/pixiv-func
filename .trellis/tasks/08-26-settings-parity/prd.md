# 复刻完整 Settings

## Goal

补齐beta56 Settings信息架构和默认值，用版本化、类型安全的现代设置存储驱动主题、浏览、下载、历史、翻译和账号入口。

## Confirmed Facts

- 当前SettingsController仅持久化guide、language和theme，Home Settings tab仍是文字占位。
- beta56 SettingsPage包含账号卡、Account、Theme、Language、Translate、Browse、Download、History、Block Tag、Downloader、About。
- 历史默认固定image source只能作为legacy/emergency route，现代默认网络仍为系统DNS直连。

## Dependencies

- 08-26-secure-account-store、restore-icon-font完成。
- DownloadManager/History等服务通过稳定setting provider读取，不直接读SharedPreferences。

## Requirements

- R1: 复刻Settings页面分组、顺序、ListTile视觉、账号卡和各子页面；账号卡tap到Me，复制迁移long-press由独立clipboard task接管。
- R2: 定义版本化AppSettings schema和typed keys，保留guide、theme(0/1/-1)、language、preview/scale quality、local/Pixiv history、R18/AI block、translate、max downloads、naming等原版项。
- R3: 默认max downloads=3、local history=on、Pixiv history=on、preview high quality=on、theme=system；语言延续首次引导。
- R4: 历史210.140.92.148只作为显式legacy/emergency route选项，Normal系统DNS/HTTPS为现代默认；UI说明其风险。
- R5: 普通设置使用SharedPreferencesAsync/repository，翻译凭据等secret转入CredentialStore，不进入settings JSON。
- R6: 设置写入序列化、可等待、失败可观察；不使用fire-and-forget让UI显示未持久化成功。
- R7: 提供schema migration、未知/损坏值逐字段恢复和保留用户其他有效设置，不因单字段错误清空全部。
- R8: 设置变更通过typed providers通知Download/History/Image/Theme等服务，避免全局静态读取。
- R9: About显示版本、许可证/归属入口；Updater行为由独立task接入。

## Acceptance Criteria

- [ ] Settings主页面/子页面顺序和可见行为与beta56对应，无文字占位。
- [ ] 新安装默认值全部正确；theme/language/quality/history/max-download重启后保持。
- [ ] 单字段损坏/migration不会重置整个settings，写入失败UI回滚或显示错误。
- [ ] secret不出现在SharedPreferences/settings JSON/log；typed providers即时驱动相关服务。
- [ ] legacy image route非默认且有明确标识，Normal严格TLS不受影响。
- [ ] analyze、全量test、debug build及重启/迁移/四语言/三主题真机验证通过。

## Out of Scope

- 安全剪贴板账号迁移实现。
- Downloader/History业务实现本身。
- Updater下载/安装。
- 新增原版没有的设置。

## Risks and Deferred Items

- 设置项依赖多个尚未完成feature；页面只能在owning feature可用后显示真实能力，不能用空onTap占位。

## Source Anchors

- beta56 lib/pages/settings/*、lib/models/settings.dart、lib/app/services/settings_service.dart
- 当前 lib/core/settings/*、onboarding、Home Settings tab

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
