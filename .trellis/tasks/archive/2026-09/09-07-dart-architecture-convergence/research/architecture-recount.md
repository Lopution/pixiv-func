# Research: Dart 架构计数重跑（HEAD `1186ff2`）

- **Query**: 对 parent `dart-architecture-audit.md` / `modernization-gaps.md` 的计数在当前 HEAD 重跑，作为 child C（`09-07-dart-architecture-convergence`）开工 gate。
- **Scope**: internal（`/root/Pixiv-func-C2` 的 `lib/`、`test/`；脚本在 `/tmp/c-recount/`，不写入仓库）
- **Date**: 2026-09-08
- **HEAD**: `1186ff2e79c591bf68e9db6f3b066a3cd3db7da6`（`main` 合入 `09-01-behavior-correctness-cleanup` 后）
- **Baseline**: 两份 parent 研究写于 2026-09-07，测量的是**脏工作树**（大量未提交 `lib/`/`test/` 改动），不是 tag `baseline-2026-09-07` 的干净树。下表「baseline」一律取这两份文档当时写下的数字。
- **工具链**: Flutter 3.47.2（`/opt/flutter-3.47.2/bin`）；`flutter analyze --no-pub`；`flutter test -j 4`

统计口径：`wc -l`（按换行计 LOC）、`rg`、以及附录中的 Python（import 图按相对路径 + `package:pixiv_func/` 解析，与 parent B3 相同；方法长度按去字符串/注释后的花括号配对）。

---

## A. 结构 / 规模

### A1. 顶层文件数与 LOC

命令：

```bash
find lib -name '*.dart' | wc -l
find lib -name '*.dart' -print0 | xargs -0 wc -l | tail -1
# 子目录同理：lib/app lib/core lib/features；core/features 各一级子目录再各跑一遍
```

| 目录 | baseline 文件 / LOC | HEAD 文件 / LOC | Δ 文件 | Δ LOC |
|---|---|---|---|---|
| `lib/` 合计 | 200 / 51,772 | 199 / 51,843 | −1 | +71 |
| `lib/app/` | 14 / 855 | 14 / 855 | 0 | 0 |
| `lib/core/` | 135 / 33,254 | 135 / 33,487 | 0 | +233 |
| `lib/features/` | 50 / 17,629 | 49 / 17,467 | −1 | −162 |
| `test/` Dart | 72 / 21,903 | 75 / 23,901 | +3 | +1,998 |
| `test/*_test.dart` | 71 | 74 | +3 | — |

`features/` 少的 1 个文件是 `lib/features/login/login_intercept_controller.dart`（已不存在）。`lib/core/network/compat/` 仍为 9 文件，LOC 4,061 → 4,122（+61）。

`lib/core/` 子目录 HEAD（文件 / LOC）：

| 目录 | HEAD | Δ LOC vs baseline |
|---|---|---|
| `network/` | 16 / 4,888 | +61 |
| `download/` | 11 / 3,391 | 0 |
| `ugoira/` | 10 / 2,298 | +3 |
| `novel/` | 4 / 1,995 | 0 |
| `comments/` | 8 / 1,957 | +2 |
| `i18n/` | 1 / 1,900 | −24 |
| `profile/` | 7 / 1,898 | 0 |
| `updater/` | 6 / 1,937 | +99 |
| `auth/` | 10 / 1,702 | 0 |
| `platform/` | 9 / 1,319 | +29 |
| `widget/` | 6 / 1,106 | +23 |
| `history/` | 6 / 1,176 | +13 |
| `user/` | 7 / 1,179 | 0 |
| `entity/` | 4 / 1,066 | 0 |
| `settings/` | 5 / 993 | −7 |
| `search/` | 5 / 950 | 0 |
| `bookmark/` | 4 / 540 | 0 |
| `new/` | 3 / 300 | 0 |
| `mutation/` | 2 / 255 | 0 |
| `navigation/` | 2 / 27 | 0 |
| `paging/` | 2 / 834 | 0 |
| `reverse_image/` | 7 / 1,776 | +34 |

`lib/features/` 子目录 HEAD：

| 目录 | HEAD 文件 / LOC | Δ vs baseline |
|---|---|---|
| `settings/` | 3 / 2,904 | +23 |
| `profile/` | 7 / 2,687 | 0 |
| `illust/` | 8 / 2,680 | 0 |
| `novel/` | 3 / 1,885 | 0 |
| `search/` | 8 / 1,784 | +13 |
| `home/` | 5 / 1,575 | 0 |
| `comments/` | 4 / 1,051 | 0 |
| `login/` | **2** / 663 | −1 文件 / −198 |
| `history/` | 1 / 608 | 0 |
| `ranking/` | 2 / 461 | 0 |
| `new/` | 1 / 449 | 0 |
| `onboarding/` | 4 / 408 | 0 |
| `bookmark/` | 1 / 312 | 0 |

`features/` 后缀：`*_page.dart` 24（不变）、`*_controller.dart` **5**（baseline 6，少 `login_intercept_controller.dart`）、`*_repository.dart` 5（不变）。

### A2. `lib/` 最大 15 个文件

命令：`find lib -name '*.dart' -print0 | xargs -0 wc -l | sort -n | tail -16`

| # | 文件 | baseline LOC | HEAD LOC | Δ |
|---|---|---|---|---|
| 1 | `lib/features/settings/settings_page.dart` | 2,088 | 2,088 | 0 |
| 2 | `lib/core/i18n/replica_strings.dart` | 1,924 | 1,900 | −24 |
| 3 | `lib/core/novel/novel_entity.dart` | 1,559 | 1,559 | 0 |
| 4 | `lib/features/illust/detail/illust_detail_page.dart` | 1,388 | 1,388 | 0 |
| 5 | `lib/core/network/compat/network_policy.dart` | 1,198 | 1,197 | −1 |
| 6 | `lib/core/download/download_manager.dart` | 1,164 | 1,164 | 0 |
| 7 | `lib/features/profile/user_page.dart` | 959 | 959 | 0 |
| 8 | `lib/features/novel/novel_layout.dart` | 926 | 926 | 0 |
| 9 | `lib/core/network/compat/secure_resolver.dart` | 868 | 868 | 0 |
| 10 | `lib/core/ugoira/ugoira_zip.dart` | 741 | 744 | +3 |
| 11 | `lib/features/profile/profile_header_delegate.dart` | 713 | 713 | 0 |
| 12 | `lib/core/ugoira/ugoira_export.dart` | 698 | 698 | 0 |
| 13 | `lib/core/paging/paged_feed_controller.dart` | 629 | 629 | 0 |
| 14 | `lib/core/profile/web_profile_repository.dart` | 627 | 627 | 0 |
| 15 | `lib/core/network/compat/network_probe.dart` | 613 | 675 | +62 |

四个拆分目标（PRD R5 / C8）——均未拆，仍远高于 600 行：

| 文件 | LOC baseline → HEAD | 顶层 class 数 | 公开顶层 class | 页面类（`*Page`） |
|---|---|---|---|---|
| `settings_page.dart` | 2,088 → 2,088 | 27 | 14 | **14**（与 PRD 一致）：`SettingsPage` L89、`MePage` L349、`AccountSettingsPage` L422、`ThemeSettingsPage` L527、`LanguageSettingsPage` L571、`TranslateSettingsPage` L618、`TranslationCredentialsPage` L711、`BrowseSettingsPage` L989、`DownloadSettingsPage` L1150、`DownloadDestinationPage` L1360、`HistorySettingsPage` L1513、`BlockedTagsPage` L1570、`DownloadTasksPage` L1649、`AboutSettingsPage` L1766。第二个 `MePage` 仍在 `user_page.dart:48` |
| `illust_detail_page.dart` | 1,388 → 1,388 | 15 | 1（`IllustDetailPage`） | 1；`_GlobalRectClip` 仍私有于 L344 |
| `user_page.dart` | 959 → 959 | 16 | 2（`UserPage`、`MePage`） | 2 |
| `network_policy.dart` | 1,198 → 1,197 | 6 | 3：`NetworkAccessPolicy` L30、`PixivPolicyHttpClient` L1043、`PixivNetworkFactory` L1142 | — |

### A3. 方法 > 150 行

脚本：`/tmp/c-recount/methods_named.py`（去字符串/注释后配对 `{}`，参数列表允许 `{named}`）。

Parent 列出的 5 个仍在。命名参数感知扫描另外看到 3 个 >150 的方法（parent 文档未列）。

| 方法 | baseline 行数 | HEAD | Δ | 文件:行 |
|---|---|---|---|---|
| `UpdateService._checkOnce` | 228 | **239** | +11 | `lib/core/updater/update_service.dart:193-431` |
| `DownloadManager._run` | 187 | 188 | +1 | `lib/core/download/download_manager.dart:527-714` |
| `_LoginPageState.build` | 186 | 186 | 0 | `lib/features/login/login_page.dart:149-334` |
| `IllustCard.build` | 180 | 180 | 0 | `lib/features/home/recommended/recommended_illust_page.dart:243-422` |
| `ProfileEditController.submit` | 167 | 168 | +1 | `lib/core/profile/profile_edit_controller.dart:223-389` |
| `NetworkProbe._runner` | （未列） | **225** | 新入榜 | `lib/core/network/compat/network_probe.dart:85-309` |
| `DownloadManager._recover` | （未列） | **187** | 新入榜 | `lib/core/download/download_manager.dart:274-460` |
| `NovelContentMapper._parseMarker` | （未列） | **184** | 新入榜 | `lib/core/novel/novel_entity.dart:1026-1209` |

PRD「5 个 >150 行方法」：原 5 个都还在。C8 若按「所有 >150」则变成 **8**。`IllustCard` 仍定义在 `recommended_illust_page.dart:236`。

---

## B. 分层（C0 白名单初值）

脚本：`/tmp/c-recount/imports_only.py`（`import`/`export`/`part`；相对路径与 `package:pixiv_func/` 解析到真实文件）。

### B2–B3. 层间边

| 边类型 | baseline | HEAD | Δ |
|---|---|---|---|
| `features→core` | 269 | 268 | −1 |
| `features→app` | 74 | 74 | 0 |
| `features→features`（含同 feature） | 92 | 93 | +1 |
| `features→features` 且 `a ≠ b` | （未单列） | **44** | — |
| `core→core` | 392 | 396 | +4 |
| `app→core` | 10 | 10 | 0 |
| `core→features` | **1** | **1** | 0 |
| `app→features` | 1 | 1 | 0 |

#### `lib/core/**` → `lib/features/**`（C0 必须白名单，正好 1 条）

| from | line | uri | to |
|---|---|---|---|
| `lib/core/widget/widget_feed_loader.dart` | 21 | `../../features/home/recommended/recommended_repository.dart` | `lib/features/home/recommended/recommended_repository.dart` |

#### `lib/app` ↔ `lib/features/onboarding`（互引仍在）

`app→onboarding`（1）：

| from | line | to |
|---|---|---|
| `lib/app/app.dart` | 14 | `lib/features/onboarding/startup_gate.dart` |

`onboarding→app`（16；`startup_gate.dart` 本身不 import `lib/app`，它 import `home_page` / `login_page`）：

| from | line | to |
|---|---|---|
| `language_page.dart` | 3, 5–9 | `replica_route.dart`、`func_tokens.dart`、`replica_button.dart`、`replica_scaffold.dart`、`replica_switch_tile.dart`、`settings_load_error.dart` |
| `theme_page.dart` | 3, 5–9 | 同上 6 个 |
| `welcome_page.dart` | 2, 4–6 | `replica_route.dart`、`func_tokens.dart`、`replica_button.dart`、`replica_scaffold.dart` |

#### `lib/features/<a>` → `lib/features/<b>`（a ≠ b）— C0 白名单，44 条，稳定列表

| # | from | line | to |
|---|---|---|---|
| 1 | `comments/comment_item.dart` | 10 | `profile/user_page.dart` |
| 2 | `history/history_page.dart` | 16 | `illust/detail/illust_detail_page.dart` |
| 3 | `history/history_page.dart` | 17 | `novel/novel_page.dart` |
| 4 | `home/home_page.dart` | 17 | `profile/user_page.dart` |
| 5 | `home/home_page.dart` | 18 | `illust/detail/illust_detail_page.dart` |
| 6 | `home/home_page.dart` | 19 | `new/new_page.dart` |
| 7 | `home/home_page.dart` | 20 | `ranking/ranking_page.dart` |
| 8 | `home/home_page.dart` | 21 | `search/search_page.dart` |
| 9 | `home/home_page.dart` | 22 | `search/reverse_image_search_page.dart` |
| 10 | `home/home_page.dart` | 23 | `search/search_text.dart` |
| 11 | `home/home_page.dart` | 24 | `settings/settings_page.dart` |
| 12 | `home/recommended/recommended_home_page.dart` | 18 | `novel/novel_page.dart` |
| 13 | `home/recommended/recommended_home_page.dart` | 19 | `profile/user_page.dart` |
| 14 | `home/recommended/recommended_illust_page.dart` | 11 | `bookmark/bookmark_switch_button.dart` |
| 15 | `home/recommended/recommended_illust_page.dart` | 12 | `illust/detail/illust_detail_page.dart` |
| 16 | `illust/detail/illust_detail_page.dart` | 34 | `bookmark/bookmark_switch_button.dart` |
| 17 | `illust/detail/illust_detail_page.dart` | 35 | `comments/comments_page.dart` |
| 18 | `illust/detail/illust_detail_page.dart` | 36 | `profile/user_page.dart` |
| 19 | `illust/detail/illust_detail_page.dart` | 37 | `search/tag_search_page.dart` |
| 20 | `illust/detail/related_illusts_section.dart` | 6 | `home/recommended/recommended_illust_page.dart`（`IllustCard`） |
| 21 | `illust/detail/related_illusts_section.dart` | 9 | `search/search_text.dart` |
| 22 | `new/new_page.dart` | 13 | `home/recommended/recommended_illust_page.dart` |
| 23 | `new/new_page.dart` | 14 | `novel/novel_page.dart` |
| 24 | `novel/novel_page.dart` | 18 | `profile/user_page.dart` |
| 25 | `onboarding/startup_gate.dart` | 7 | `home/home_page.dart` |
| 26 | `onboarding/startup_gate.dart` | 8 | `login/login_page.dart` |
| 27 | `onboarding/theme_page.dart` | 13 | `login/login_page.dart` |
| 28 | `profile/user_page.dart` | 20 | `home/recommended/recommended_illust_page.dart` |
| 29 | `profile/user_page.dart` | 21 | `novel/novel_page.dart` |
| 30 | `ranking/ranking_page.dart` | 11 | `home/recommended/recommended_illust_page.dart` |
| 31 | `search/reverse_image_search_page.dart` | 16 | `illust/detail/illust_detail_page.dart` |
| 32 | `search/reverse_image_search_page.dart` | 17 | `profile/user_page.dart` |
| 33 | `search/search_page.dart` | 11 | `illust/detail/illust_detail_page.dart` |
| 34 | `search/search_result_page.dart` | 16 | `home/recommended/recommended_illust_page.dart` |
| 35 | `search/search_result_page.dart` | 17 | `novel/novel_page.dart` |
| 36 | `search/search_result_page.dart` | 18 | `profile/follow_switch_button.dart` |
| 37 | `search/search_result_page.dart` | 19 | `profile/user_page.dart` |
| 38 | `search/search_router.dart` | 5 | `illust/detail/illust_detail_page.dart` |
| 39 | `search/search_router.dart` | 6 | `novel/novel_page.dart` |
| 40 | `search/search_router.dart` | 7 | `profile/user_page.dart` |
| 41 | `settings/settings_page.dart` | 27 | `login/login_page.dart` |
| 42 | `settings/settings_page.dart` | 28 | `profile/profile_edit_page.dart` |
| 43 | `settings/settings_page.dart` | 29 | `profile/user_page.dart` |
| 44 | `settings/settings_page.dart` | 30 | `history/history_page.dart` |

### features/ 内 repository / controller / entity

PRD：5 repository、6 controller。HEAD：

**Repository 文件（仍 5，均在 `features/`）：**

| 文件 | 其中的类型 |
|---|---|
| `lib/features/home/recommended/recommended_repository.dart` | `RecommendedIllustRepository`；同文件 `RecommendedIllustController` L87 |
| `lib/features/illust/detail/illust_detail_repository.dart` | `IllustDetailRepository`（L18 仍手写 `host: 'app-api.pixiv.net'`） |
| `lib/features/illust/detail/related_illust_repository.dart` | `PixivRelatedIllustRepository`；同文件 `RelatedIllustController` L83 |
| `lib/features/ranking/ranking_repository.dart` | `RankingRepository`；同文件 `RankingFeedController` L112 |
| `lib/features/search/tag_search_repository.dart` | **没有 Repository 类**；`TagSearchController` L9（仅 `test/illust_detail_controller_test.dart` import） |

**`*_controller.dart` 文件：6 → 5**（`login_intercept_controller.dart` 已删除）：

| 文件 | 类 |
|---|---|
| `lib/features/home/recommended/recommended_feed_controller.dart` | `RecommendedFeedController` |
| `lib/features/illust/detail/illust_detail_controller.dart` | `IllustDetailController` |
| `lib/features/illust/detail/illust_download_controller.dart` | `IllustDownloadController` |
| `lib/features/profile/profile_feed_controller.dart` | `ProfileIllustFeedController`、`ProfileUserFeedController` |
| `lib/features/profile/user_detail_controller.dart` | `UserDetailController` |

另：`lib/features/novel/novel_reader.dart:135` `NovelReaderController extends ChangeNotifier`（状态范式，不是 `*_controller.dart`）。`lib/core/` 仍有 10 个 `*_controller.dart`。

**features/ entity/models：** `lib/features/profile/profile_models.dart`（`ProfileTab` L6、`MeProfileTab` L9，均为零引用声明）。

---

## C. 重复 / 散落契约（R3）

### C1. Pixiv 主机字面量（`pixiv_client_identity.dart` 之外）

`rg 'i.pximg.net|s.pximg.net|app-api.pixiv.net|oauth.secure.pixiv.net|www.pixiv.net|accounts.pixiv.net' lib --glob '*.dart'`

identity 文件自身仍持有 `app-api.pixiv.net` / `oauth.secure.pixiv.net` / `i.pximg.net` / `s.pximg.net`。其外：

| 字面量 | 非 identity 的 file:line（代码，注释另标） |
|---|---|
| `app-api.pixiv.net` | `network_contracts.dart:55,58`；`network_policy.dart:176`；`network_fast_route_store.dart:27`；`next_page_parser.dart:109`；`network_probe_page.dart:62`（及 L157 注释）；**`illust_detail_repository.dart:18`** |
| `oauth.secure.pixiv.net` | `network_contracts.dart:56`；`network_policy.dart:177`；`network_fast_route_store.dart:28`；`network_probe_page.dart:63` |
| `www.pixiv.net` | `network_contracts.dart:59`；`network_policy.dart:178`；`network_fast_route_store.dart:32`；`web_profile_repository.dart:40,41,43,423`；`intent_router.dart:9,18`（注释）、`:135`；`illust_detail_page.dart:1195`（注释）、`:1271`；`user_page.dart:937` |
| `accounts.pixiv.net` | `network_contracts.dart:58`；`login_webview_page.dart:64` |
| `i.pximg.net` | `network_contracts.dart:61`；`network_policy.dart:179`；`network_fast_route_store.dart:33`；`network_probe_page.dart:64`；`app_settings.dart:11,189`；`ugoira_export.dart:685`；`widget_feed_loader.dart:79,293`（注释）；`pixiv_image.dart:11`（注释）；`policy_download_transport.dart:68`（注释） |
| `s.pximg.net` | `network_contracts.dart:61`；`network_policy.dart:180`；`network_fast_route_store.dart:34`；`network_probe_page.dart:65`；`network_probe.dart:313`（注释）；`policy_download_transport.dart:68`（注释） |

相对 baseline：**`login_intercept_controller.dart` 上的主机再声明已随文件消失**。其余再声明点仍在（probe 页行号 56–59 → **62–65**）。`PixivHeaders` 仍无 `image()`；`'Referer'` 拼装仍在 `download_manager.dart:563`、`widget_feed_loader.dart:297`、`ugoira_repository.dart:138`、`pixiv_image.dart:67`。

### C2. SharedPreferences 直接构造

`rg 'SharedPreferencesAsync\(\)|SharedPreferences.getInstance\(\)' lib`

| | baseline | HEAD | Δ |
|---|---|---|---|
| `SharedPreferencesAsync()` | 7 处（审计写「6 处文件」、现代化写 7 次构造） | **7** | 0 |
| `SharedPreferences.getInstance()` | （未强调） | **0** | — |

HEAD 7 处：`blocked_tags.dart:18,36`；`settings_repository.dart:30`；`update_download.dart:163`；`network_fast_route_store.dart:40`；`download_recovery.dart:372`；`account_repository.dart:38`。

### C3. JSON helper 定义次数（同名各写一份）

定义点（不是调用点）：

| 名字 | baseline | HEAD 定义 | 文件:行 |
|---|---|---|---|
| `_map` | 5 | **5** | `user_entity.dart:249`；`novel_entity.dart:1538`；`novel_repository.dart:327`；`reverse_image_platform.dart:163`；`reverse_image_provider.dart:220` |
| `_firstString` | 3 | **3** | `user_entity.dart:282`；`novel_entity.dart:1541`；`search_repository.dart:355` |
| `_positiveInt` | 5 | **5** | `comment_entity.dart:176`；`user_entity.dart:254`；`novel_entity.dart:1510`；`update_manifest.dart:327`；`novel_repository.dart:319` |
| `_optionalString` | 4 | **4** | `user_entity.dart:269`；`novel_entity.dart:1535`；`novel_repository.dart:324`；`illust_entity.dart:424` |
| `_requiredString` | 2 | **2** | `user_entity.dart:264`；`novel_entity.dart:1528` |
| `_nonNegativeInt` | 2 | **2** | `user_entity.dart:259`；`novel_entity.dart:1523` |
| `_nextUrl` | 5 | **5** | `new_feed_repository.dart:191`；`user_repository.dart:355`；`comment_repository.dart:216`；`search_repository.dart:351`；`novel_repository.dart:330` |

### SnackBar / debugPrint / 内联 HTTP

| 项 | baseline | HEAD | Δ | 命令 / 位置 |
|---|---|---|---|---|
| `showSnackBar(` | 44 / 17 文件 | **44** | 0 | `rg -c 'showSnackBar\(' lib` 求和 |
| `SnackBar(` | 79（审计） | 仍与 44 次 show 成对出现 | — | 无 `showAppSnackBar` |
| `debugPrint(` | 23 / 5 文件 | **23** / 5 | 0 | `network_probe_page.dart` 2；`widget_channel.dart` 1；`widget_feed_loader.dart` 12；`widget_coordinator.dart` 2；`widget_background.dart` 6 |
| `http.Client()` | 5 默认参数 + 生产 `comment_translation` | 仍 5：`oauth_service.dart:98`、`pixiv_http_client.dart:60`、`sauce_nao_provider.dart:39`、`pixiv_download_transport.dart:55`、**`comment_translation.dart:537`（生产路径直接 `http.Client()`）** | 行号微调 | |
| `HttpClient()` | 3 | **3**：`network_probe.dart:475`、`secure_resolver.dart:790`、`update_service.dart:443` | 行号微调 | |
| `login_intercept` 上的 `http.Request` | 有 | **文件已不存在** | 已消失 | |

---

## D. 组件（R7）

| 项 | baseline | HEAD | Δ |
|---|---|---|---|
| `SliverMasonryGrid.count` | 8 | **8** | 0 |
| `crossAxisCount: 2` in `lib/features` | 8（全写死） | **8** | 0 |
| 私有 feed 态 widget（见下） | 32 | **32**（宽口径）/ 26（严格后缀） | 0 |
| `PixivImage(` 出现文件 | 13 文件 | **13** 文件 | 0 |
| 其中已传 `memCacheWidth` | 0（全仓库仅反查页 `cacheWidth: 1024`） | 仍 0；`reverse_image_search_page.dart:226` `cacheWidth: 1024` | 0 |
| `CachedNetworkImage(` | 仅 `pixiv_image.dart` | 仅 `lib/app/pixiv_image.dart:167` | 0 |
| `GestureDetector(` | 14 | **14** | 0 |
| `Duration(milliseconds:` | （散落） | 16 处 / 13 文件；`lib/app/motion/` **不存在** | — |
| `Curves.` | replica 两套路由 | 3 处：`replica_page_route.dart`、`replica_route.dart`、`novel_reader.dart:422` | — |
| `Navigator.of(context).push` | 28 | **28** | 0 |
| `replicaRoute(` | 3 | **3**（onboarding `welcome/language/theme`） | 0 |
| `ReplicaPageRoute` 构造调用 | 30 | **29**（另加类定义） | −1 |
| `PersonAvatar(` 调用 | 9（design §13） | **8** + 定义 | −1 |
| `HomeShellMetrics` | 2 个可写 static | 仍 `bottomNavTop` / `bottomNavHeight`（`home_shell_metrics.dart:15,19`） | 0 |

瀑布流 8 处（均紧跟 `crossAxisCount: 2`）：

- `history_page.dart:253`
- `ranking_page.dart:162`
- `user_page.dart:461`
- `new_page.dart:273`
- `recommended_illust_page.dart:107`
- `recommended_home_page.dart:264`
- `search_result_page.dart:143`
- `related_illusts_section.dart:127`

`PixivImage(` 调用点（定义除外）：`search_page.dart:193`、`image_viewer_page.dart:125`、`novel_page.dart:177`、`profile_edit_page.dart:470`、`profile_header_delegate.dart:339,351`、`comment_item.dart:216`、`history_page.dart:367,525`、`illust_detail_page.dart:832`、`recommended_illust_page.dart:302`、`ugoira_viewer.dart:199`、`recommended_home_page.dart:414`、`person_avatar.dart:62`；`pixiv_image.dart:154` 为内部再构造。PRD「13 个调用点」按**文件**仍成立。

`GestureDetector(`：`search_page.dart:179`、`login_page.dart:204`、`novel_reader.dart:294`、`history_page.dart:304`、`recommended_illust_page.dart:260`、`illust_detail_page.dart:603,806,958,1157,1221`、`ugoira_viewer.dart:127`、`comment_item.dart:187`、`bookmark_switch_button.dart:222,234`。

`HomeShellMetrics` 写入：`home_page.dart:196-197`。读取：`illust_detail_page.dart:301,306`（fallback `kHomeBottomNavHeight = 80.0` 于同文件 L55）。

### 私有 feed 尾部 / 空态 / 错误 widget（差异矩阵用）

严格后缀 `_*Tail|_*Error|_*Empty|_*Status|_*Placeholder`：**26**。parent「32」把近义名算进去后 HEAD 仍能凑满 32（无删除）。下表 32 项 + 2 个额外近邻（不计入 32）。

| # | 类 | 文件:行 | 主要参数 | 渲染要点 / i18n |
|---|---|---|---|---|
| 1 | `_CommentFeedTail` | `comments_page.dart:359` | `feed`, `onRetry` | spinner / 完结 / 重试；`commentText` |
| 2 | `_CommentError` | `comments_page.dart:387` | `error`, `onRetry` | 居中 Column + 重试 |
| 3 | `_HistoryError` | `history_page.dart:530` | `error`, `onRetry` | 居中 Column + 重试；`_historyText` |
| 4 | `_FeedTail` | `recommended_home_page.dart:475` | `feed`, `onRetry` | spinner / `_TailError` / 完结 |
| 5 | `_TailError` | `recommended_home_page.dart:507` | `onRetry` | `OutlinedButton.icon` refresh |
| 6 | `_RecommendedError` | `recommended_home_page.dart:527` | `error`, `onRetry` | 可滚动 Column |
| 7 | `_FeedTail` | `recommended_illust_page.dart:136` | `feed`, `onRetry` | 同上形态 |
| 8 | `_PagePlaceholder` | `illust_detail_page.dart:911` | （无） | `Icons.image_outlined` 占位 |
| 9 | `_NewStatus` | `new_page.dart:305` | `icon`, `title` | 居中 icon+title |
| 10 | `_NewEmpty` | `new_page.dart:326` | `onRefresh` | `Icons.inbox_outlined` + `_newText` |
| 11 | `_NewError` | `new_page.dart:352` | `error`, `onRetry` | 可滚动 Column |
| 12 | `_NewRefreshError` | `new_page.dart:383` | `onRetry` | `errorContainer` 横条 |
| 13 | `_NewFeedTail` | `new_page.dart:407` | `feed`, `onRetry` | spinner / 完结 / 重试 |
| 14 | `_NovelStatus` | `novel_page.dart:353` | `icon`, `title` | 居中 icon+title |
| 15 | `_NovelError` | `novel_page.dart:374` | `error`, `onRetry` | 404 → `novelNotFound` 否则 `novelLoadFailed` |
| 16 | `_StartupError` | `startup_gate.dart:54` | `error` | **无 onRetry**；局部 `text()` + `ReplicaStrings.fromTag` |
| 17 | `_ProfileFeedTail` | `user_page.dart:783` | `feed`, `onRetry` | padding 18 的 spinner |
| 18 | `_ProfileEmpty` | `user_page.dart:816` | `onRetry` | 固定高 240 |
| 19 | `_ProfileFeedError` | `user_page.dart:852` | `error`, `onRetry` | 可滚动 Column |
| 20 | `_RankingFeedTail` | `ranking_page.dart:190` | `feed`, `onRetry` | 与 illust `_FeedTail` 近 |
| 21 | `_RankingInitialError` | `ranking_page.dart:242` | `mode`, `error`, `onRetry` | 带 `RankingMode` |
| 22 | `_SearchInlineError` | `search_page.dart:465` | `title`, `error`, `onRetry` | 行内错误 |
| 23 | `_SearchStatus` | `search_result_page.dart:296` | `title` | `Icons.search` |
| 24 | `_SearchEmpty` | `search_result_page.dart:316` | `onRefresh` | `Icons.search` + 刷新 |
| 25 | `_SearchResultError` | `search_result_page.dart:342` | `error`, `onRetry` | 可滚动 Column |
| 26 | `_SearchFeedTail` | `search_result_page.dart:373` | `feed`, `onRetry` | spinner / 完结 |
| 27 | `_InitializationFailure` | `profile_edit_page.dart:182` | `error` | 初始化失败（非 PagedFeed） |
| 28 | `_RecommendedEmptyFeed` | `recommended_home_page.dart:324` | `message`, `onRefresh` | **已用** `ReplicaEmptyState`（`retry`） |
| 29 | `_InitialErrorView` | `recommended_illust_page.dart:188` | `error: String`, `onRetry` | 可滚动 Column |
| 30 | `_ErrorView` | `illust_detail_page.dart:1349` | `error`, `onRetry` | 详情页错误 |
| 31 | `_ErrorOverlay` | `ugoira_viewer.dart:571` | `message`, `onRetry` | 半透明黑底 overlay |
| 32 | `_ProfileStatusPage` | `user_page.dart:883` | `icon`, `title`, `detail?`, `onRetry?` | 整页状态 |
| + | `_LoadMoreFooter` | `related_illusts_section.dart:162` | `state`, `onLoadMore` | 相关作品 footer（未入 parent 32） |
| + | `_StatusBody` | `profile_edit_page.dart:529` | `state`, `onRetry` | 资料编辑状态（未入 parent 32） |

---

## E. 现代化

| 项 | baseline | HEAD | 注 |
|---|---|---|---|
| `ChangeNotifier` 子类 | 3 | **3** | `ProfileEditController` `profile_edit_controller.dart:65`；`ReverseImageSearchController` `reverse_image_controller.dart:64`；`NovelReaderController` `novel_reader.dart:135` |
| 页面 `new` 平台适配器 | 3 页 | 仍 3 页 | `profile_edit_page.dart:174` `MethodChannelReverseImageInputPlatform()`；`reverse_image_search_page.dart:46,50` 两个 MethodChannel 适配器；`home_page.dart:70` `MethodChannelAndroidIntentSource()` |
| `history_page` 手写分页 | 有 | **仍有** | `ScrollController` L113；`extentAfter < 400` → `_loadMore()` L142–145；自持 `_records/_hasMore/_loading/_generation` |
| `_xxxText` / fromTag 包装 | 27 | 同族仍在 | 见下 |
| `ReplicaStrings` 键 | 434 ×4（ru 433） | **432** ×3 + ru **431** | −2 键（webview intercept 文案已不在表内）；ru 仍缺 `detailQuality` |
| `networkDohEndpointsInvalid` | 未定义仍被调用 | **仍未定义** | `network_settings_page.dart:205`；四语言只有 `networkDohEndpoints` |
| `Colors.*` 于 `lib/` | 37 / 15 文件 | **37** / 15 | `replica_theme.dart` 5 处；其余 32 处在 feature。白/黑/透明偏 replica 视觉：`Colors.white`（login/onboarding/recommended/image_viewer/ugoira）、`Colors.black`/`black38`（viewer/webview）、`Colors.transparent`（follow/bookmark/header/theme） |
| `unreachable_from_main` | 未开；估 ≥13 声明 +4 文件 | 仍未开（`analysis_options.yaml` 仅 `flutter_lints`） | 现估 **≥12 声明 + 2 零引用 barrel 文件** |

i18n 包装（body 为 `ReplicaStrings.fromTag` / `.text` 的命名函数，不完全等于 PRD「27」的同一正则，但集合重叠）：

| 名字 | 文件:行 |
|---|---|
| `_settingsText` | `settings_page.dart:33` |
| `_networkText` | `network_settings_page.dart:11` |
| `_probeText` | `network_probe_page.dart:20` |
| `_historyText` | `history_page.dart:25` |
| `_recommendedText` | `recommended_home_page.dart:24` |
| `_newText` | `new_page.dart:446` |
| `_novelText` | `novel_page.dart:410` |
| `_profileText` | `user_page.dart:113`；`profile_header_delegate.dart:269` |
| `_profileEditText` | `profile_edit_page.dart:21` |
| `_detailText` | `illust_detail_page.dart:1380` |
| `_ugoiraText` | `ugoira_viewer.dart:24` |
| `_bookmarkText` | `bookmark_switch_button.dart:10` |
| `_loginText` | `login_page.dart:125` |
| `_text` | `login_webview_page.dart:51`；`follow_switch_button.dart:28`；`profile_header_delegate.dart:488,633` |
| `searchText` | `search/search_text.dart:5` |
| `commentText` | `comments/comment_text.dart:49` |
| 局部 `text(key)` | `settings_load_error.dart:29`；`image_viewer_page.dart:88`；`login_page.dart:164`；`theme_page.dart:43`；`startup_gate.dart:61` |
| 枚举→文案（亦名 `_*Text`） | `_qualityText:1120`、`_destinationText:1329`、`_namingPresetText:1345`、`_downloadStatusText:1751`（`settings_page.dart`） |
| 错误码映射（不是 fromTag） | `_transferErrorText:312`、`_loginTransferErrorText:130`、`_translationFailureText:160`、`_errorText` `related_illusts_section.dart:149` |

---

## F. 死代码

脚本：`imports_only.py`（文件级）+ `dead_tight.py`（词边界、lib+test）。

### F1. `lib/` 零引用文件

| 文件 | baseline | HEAD |
|---|---|---|
| `lib/core/network/compat_network.dart` | 7 行，lib/test 0 导入 | **仍 0 导入**（存在） |
| `lib/core/platform/android_platform.dart` | 16 行，0 导入 | **仍 0 导入**（存在；被 import 的是 `android_platform_interfaces.dart`） |
| `lib/features/login/login_intercept_controller.dart` | 198 行，0 导入 | **文件不存在**；`android/` 中 `login_webview_intercept` **0 命中** |
| `lib/features/search/tag_search_repository.dart` | 仅 1 个测试导入 | 仍仅 `test/illust_detail_controller_test.dart:16` |

零引用文件：4 → **2**（外加 1 个「仅测试引用」未变）。

### F2. 全仓库名字只出现 1 次的公共顶层声明

| | baseline | HEAD |
|---|---|---|
| 个数 | 13 | **12** |

| 声明 | HEAD | 状态 |
|---|---|---|
| `DnsRecordType` | `dns_message.dart:40` | 仍 once |
| `PageFetcher` | `paged_feed_controller.dart:91` | 仍 once |
| `ProfileFieldContract` | `profile_edit_models.dart:10` | 仍 once |
| `SearchResultTypeWire` | `search_models.dart:6` | 仍 once |
| `imageSourceProvider` | — | **已删除**（09-01 `e6ec619`） |
| `localBlockR18Provider` | `settings_controller.dart:226` | 仍 once（行号 234→226） |
| `localBlockAIProvider` | `settings_controller.dart:234` | 仍 once（242→234） |
| `UserRestrictWire` | `user_repository.dart:361` | 仍 once |
| `StableAnchor` | `novel_layout.dart:124` | 仍 once |
| `showProfileEditPage` | `profile_edit_page.dart:569` | 仍 once |
| `ProfileTab` | `profile_models.dart:6` | 仍 once |
| `MeProfileTab` | `profile_models.dart:9` | 仍 once |
| `showMePage` | `user_page.dart:105` | 仍 once |

### F3. 未用 `AppIcons`

`lib/` 引用仍只有 6 个 glyph / 8 处：`home`/`ranking`/`n`/`search`（`home_page.dart:276-279`）、`follow`/`friend`（`user_page.dart:640,645`；`profile_header_delegate.dart:438-439`）。

未在 `lib/` 使用的 8 个不变：`addFollow`、`filter`、`me`、`toggle`、`pawoo`、`twitter`、`web`、`blocked`。

`test/icon_font_test.dart:47-94` 仍用 switch 断言全部 14 个 codepoint（`0xe900`–`0xe90d`）与名字的对应。`home_page_test.dart` 另点 `AppIcons.ranking` / `home`。

### F4. 仅本文件使用的公共声明

| | baseline | HEAD |
|---|---|---|
| 个数 | 142 | **141** |

Δ −1 与 `LoginWebViewInterceptController` / `imageSourceProvider` 已不在树中一致。完整 141 条见附录。`settings_page.dart` 中仍仅本文件使用的 9 个页面类：`LanguageSettingsPage`、`TranslateSettingsPage`、`TranslationCredentialsPage`、`BrowseSettingsPage`、`DownloadSettingsPage`、`DownloadDestinationPage`、`HistorySettingsPage`、`BlockedTagsPage`、`DownloadTasksPage`。

`unreachable_from_main` 重估（PRD 要开的那条）：≥ **12** 个 once 声明 + **2** 个零引用 barrel 的全部导出内容（不再含 login intercept 文件）。

---

## G. 测试基线

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
cd /root/Pixiv-func-C2
flutter analyze --no-pub    # No issues found! (ran in 4.0s)
flutter test -j 4           # 00:32 +686: All tests passed!
```

| | baseline 文档 | HEAD `1186ff2` | Δ |
|---|---|---|---|
| `flutter analyze` | 0 issues（3.47.0） | **0 issues**（3.47.2） | 0 |
| `flutter test` 实跑 | 当时未跑（静态 `test(`/`testWidgets(` 626；`dart_test.yaml` 约 600） | **686 passed** | 验收「不减少」的数字是 **686** |
| PRD 预期 | ~674 | 686 | 已高于 674 |

`analysis_options.yaml` 仍只 `include: package:flutter_lints/flutter.yaml`，无 `unreachable_from_main`。

---

## 对 implement 的影响（parent C0–C9）

| 项 | 重跑后范围 |
|---|---|
| **C0** 分层测试 + 白名单 | 范围不变。初值：`core→features` 1 条；`features a≠b` **44** 条（上表）；`app↔onboarding` 1+16 条；features 内 5 repo 文件 + 5 `*_controller.dart` + repo 内嵌 controller。白名单必须抄 HEAD 行号（probe/settings 行号已变）。 |
| **C1** 删零引用 | **缩小**。`login_intercept_controller.dart` 与原生 `login_webview_intercept` **已不在**（C 只确认残留：无）。`imageSourceProvider` **已删**。仍要删：2 个 barrel、视测试处理 `tag_search_repository.dart`、**12** 个 once 声明、8 个未用 `AppIcons`、开 `unreachable_from_main`。 |
| **C2** 路由合并 | 不变。`replicaRoute()` 仍 3 处；`ReplicaPageRoute` 仍主体（29 次构造）；`HomeShellMetrics` 仍两个可写 static；`lib/app/motion/` 不存在。 |
| **C3** 组件层 | 不变。8 瀑布流、32 私有 feed 态、`IllustCard` 仍在 `recommended_illust_page.dart:236`。无一处已迁到 `lib/app/widgets/feed/`。 |
| **C3b** `PixivImage` 变体 | 不变。13 文件调用点仍无 `memCacheWidth`。 |
| **C3c** 重建边界 | 不变（静态规则，无计数变化）。 |
| **C4** repo/controller 归位 | **略缩**。迁入 core 的 `*_controller.dart` 是 **5** 不是 6。5 个 repository 文件仍在。`startup_gate.dart` 仍在 `features/onboarding/`。 |
| **C5** 单一 owner | **略缩**。主机再声明少了 login intercept 那一处；其余（registry / 预热 / bootstrap / probe / `illust_detail_repository:18` / 4 处 Referer）仍在。SharedPreferences 7、JSON helper 计数、SnackBar 44、debugPrint 23、内联 `http.Client`/`HttpClient` 均未变。 |
| **C6** i18n | 不变。432 键（不是 434）；ru 仍缺 `detailQuality`；`networkDohEndpointsInvalid` 仍未定义；包装函数仍在。 |
| **C7** 状态范式 | 不变。3 个 `ChangeNotifier`；history 手写分页；3 页仍 `new` 平台适配器。 |
| **C7b** 持久化增长 | 未重测行为；文件仍在原处。无范围变化信号。 |
| **C8** 文件拆分 | **四个大文件均未拆**。>150 行：原 5 个都在，且 `_checkOnce` 变长；扫描还看到 `_runner`/`_recover`/`_parseMarker`。C8 正文仍按 PRD 5 个即可，附录多 3 个候选。 |
| **C9** 颜色 + 私有化 | `Colors.*` 37 未变。仅本文件公共声明 142→**141**。 |

`.trellis/spec/frontend/directory-structure.md` 仍是模板；`index.md` 状态列仍全是 "To fill"。C 仍要写规则文本。

---

## 意外发现

1. **`login_intercept_controller.dart` 与 `login_webview_intercept` 已从 HEAD 消失**（09-01 已落地）。parent / PRD 仍按「可能还在」写；C1/C5 主机表要删这一行，不要再找该文件。
2. **`imageSourceProvider` 已删除**，once 声明 13→12。
3. **`replica_strings` 434→432 键**（−24 LOC）。ru 缺 `detailQuality`、`networkDohEndpointsInvalid` 未定义——这两条与 parent 一致，不是新问题。
4. **`network_probe.dart` +62 LOC**，进入最大 15 文件；出现 225 行 `_runner`。parent 的 5 个超长方法清单不完整（若用命名参数感知扫描，HEAD 有 **8** 个 >150 行方法）。
5. **测试实跑 686**，高于 PRD 口头「~674」。验收「测试数不减少」应以 **686** 为基线。
6. **parent 两份研究互不一致处仍在 HEAD 上成立、且未被拆分消掉**：审计写 settings「14 个公开页面类」，现代化写「10 个」但列出 14 个名字——HEAD 是 14。审计 SnackBar「79 个 `SnackBar(`」vs 现代化/PRD「44 处 showSnackBar」——HEAD `showSnackBar` 是 44。审计 i18n 包装 17 vs 现代化 27——HEAD 两种口径都能对上（17 个顶层 `_*Text(` vs 含方法内 `_text`/`text()` 的更大集合）。
7. **`lib/` 相对 baseline 少 1 文件、LOC 反而 +71**：login intercept −198 行，被 `network_probe`/`updater`/`platform`/`reverse_image` 等增长抵消。四个拆分目标 **0 行拆出**。
8. **`PersonAvatar(` 调用 8 处**，design §13 写 9。
9. 脏工作树 caveat 已兑现：数字与 tag 树/当时磁盘不完全相同，但分层违规与四大文件规模与 parent 描述仍同构。

---

## Related Specs

- `.trellis/spec/frontend/directory-structure.md` — 仍为模板；C 负责写入分层规则。
- `.trellis/spec/frontend/component-guidelines.md` — Detail Transition / Tab / Pull-to-Refresh 契约仍在；C 拆 `IllustCard`/`_GlobalRectClip` 时不得改这三个契约。
- `.trellis/spec/frontend/state-management.md` — `PagedFeedController`、Profile Edit、History 契约仍是 C7 的对照物。
- `.trellis/spec/frontend/quality-guidelines.md` — Flutter 3.47.2 路径；禁止在框架状态机旁再建一套。
- parent `design.md` §2–§3、§13–§14；`implement.md` Child C。

---

## 附录 A. 仅本文件使用的 141 个公共声明

```
lib/core/auth/account_transfer_service.dart:200 TransferImportResult
lib/core/auth/account_transfer_service.dart:343 transferCredentialVerifierProvider
lib/core/auth/oauth_service.dart:27 PkceSession
lib/core/auth/oauth_service.dart:55 OAuthUserProfile
lib/core/bookmark/bookmark_actions.dart:12 BookmarkActions
lib/core/comments/comment_actions.dart:18 CommentActions
lib/core/comments/comment_feed_controller.dart:10 CommentFeedController
lib/core/comments/comment_repository.dart:38 PixivCommentRepository
lib/core/comments/comment_store.dart:15 CommentStoreState
lib/core/comments/comment_store.dart:53 CommentStore
lib/core/comments/comment_translation.dart:48 CommentTranslationService
lib/core/comments/comment_translation.dart:520 DisabledCommentTranslationService
lib/core/comments/comment_translation.dart:558 translationSelectionProvider
lib/core/download/download_manager.dart:15 kProgressThrottle
lib/core/download/download_manager.dart:17 DownloadSubmissionContextProvider
lib/core/download/download_manager.dart:1061 DownloadOwnershipException
lib/core/download/download_manager.dart:1070 DownloadStorageException
lib/core/download/download_manager.dart:1076 DownloadPermissionException
lib/core/download/download_manager.dart:1082 DownloadDecodeException
lib/core/download/download_manager.dart:1088 DownloadResourceLimitException
lib/core/download/download_manager.dart:1094 classifyDownloadFailure
lib/core/download/download_request.dart:74 kDownloadExtensions
lib/core/download/download_request.dart:84 mimeTypeForExtension
lib/core/download/download_request.dart:213 kUpdateCdnHosts
lib/core/download/pixiv_download_transport.dart:32 RawHopHeaders
lib/core/download/pixiv_download_transport.dart:261 HopDownloadResponse
lib/core/entity/illust_caption.dart:18 CaptionSpan
lib/core/entity/illust_caption.dart:63 IllustCaptionParse
lib/core/history/history_repository.dart:17 PixivApiHistoryRemote
lib/core/history/history_repository.dart:382 historyDatabaseProvider
lib/core/history/history_tracker.dart:16 StopwatchHistoryClock
lib/core/history/history_visibility.dart:10 HistoryErrorCallback
lib/core/network/compat/dns_message.dart:10 DnsQuestion
lib/core/network/compat/dns_message.dart:20 DnsAnswer
lib/core/network/compat/dns_message.dart:47 DnsResponse
lib/core/network/compat/dns_message.dart:179 kSvcParamKeyEch
lib/core/network/compat/dns_message.dart:183 kSvcParamKeyIpv4Hint
lib/core/network/compat/dns_message.dart:186 HttpsSvcParam
lib/core/network/compat/network_contracts.dart:287 NetworkFailure
lib/core/network/compat/network_contracts.dart:322 NetworkRouteProbeException
lib/core/network/next_page_parser.dart:33 kNextPageEndpoints
lib/core/new/new_feed_controller.dart:10 NewFeedController
lib/core/new/new_feed_repository.dart:45 PixivNewFeedRepository
lib/core/novel/novel_entity.dart:16 NovelInlineMark
lib/core/novel/novel_entity.dart:40 NovelMarkupToken
lib/core/novel/novel_entity.dart:118 NovelJumpTarget
lib/core/novel/novel_entity.dart:122 NovelPageJumpTarget
lib/core/novel/novel_entity.dart:128 NovelUriJumpTarget
lib/core/novel/novel_entity.dart:167 NovelImageLoadRequest
lib/core/novel/novel_entity.dart:263 NovelMarkupIssueKind
lib/core/novel/novel_entity.dart:273 NovelMarkupDiagnostic
lib/core/novel/novel_entity.dart:334 NovelMarkupProgressCallback
lib/core/novel/novel_entity.dart:337 NovelMarkupParseResult
lib/core/novel/novel_entity.dart:350 NovelBlock
lib/core/novel/novel_entity.dart:462 NovelTag
lib/core/novel/novel_feed_controller.dart:10 UserNovelFeedController
lib/core/novel/novel_repository.dart:11 NovelContentUnavailableException
lib/core/novel/novel_repository.dart:45 NovelRepository
lib/core/novel/novel_repository.dart:72 PixivNovelRepository
lib/core/novel/novel_store.dart:8 NovelStore
lib/core/platform/android_intent_channel.dart:9 AndroidIntentMethods
lib/core/platform/android_intent_channel.dart:22 OutboundUrlOpener
lib/core/platform/android_intent_channel.dart:28 MethodChannelOutboundUrlOpener
lib/core/platform/intent_router.dart:4 DeepLinkRoute
lib/core/platform/media_store_channel.dart:10 MediaStoreMethods
lib/core/platform/media_store_channel.dart:151 MediaStoreChannelException
lib/core/platform/saf_tree.dart:9 SafTreePicker
lib/core/platform/saf_tree.dart:38 MethodChannelSafTree
lib/core/platform/saf_tree.dart:92 safTreeProvider
lib/core/platform/saf_tree.dart:104 SafTreeChannelException
lib/core/platform/shared_image.dart:4 SharedImage
lib/core/profile/profile_edit_controller.dart:21 ProfileEditFailure
lib/core/profile/profile_edit_repository.dart:15 PixivProfileEditRepository
lib/core/profile/web_profile_session.dart:42 WebProfileMethods
lib/core/reverse_image/reverse_image_controller.dart:22 ReverseImageFlowFailure
lib/core/reverse_image/reverse_image_platform.dart:18 ReverseImagePlatformFailureCode
lib/core/reverse_image/reverse_image_platform.dart:37 ReverseImageInputMethods
lib/core/search/search_autocomplete_controller.dart:12 SearchAutocompleteState
lib/core/search/search_repository.dart:90 PixivSearchRepository
lib/core/settings/app_settings.dart:151 SecretSettingRef
lib/core/settings/blocked_tags.dart:7 BlockedTags
lib/core/ugoira/ugoira_export.dart:666 UgoiraExportCanceledException
lib/core/ugoira/ugoira_metadata.dart:8 UgoiraFrameMetadata
lib/core/ugoira/ugoira_recovery.dart:14 UgoiraRecoveryReport
lib/core/ugoira/ugoira_zip.dart:23 UgoiraZipEntry
lib/core/updater/update_download.dart:134 UpdateDownloadStateStore
lib/core/updater/update_manifest.dart:15 updateReleaseAbis
lib/core/updater/update_service.dart:37 PlatformUpdateSignatureVerifier
lib/core/user/follow_actions.dart:9 FollowActions
lib/core/user/follow_repository.dart:21 PixivFollowRepository
lib/core/user/user_repository.dart:12 userWorkTypeWire
lib/core/user/user_repository.dart:95 PixivUserRepository
lib/core/widget/widget_background.dart:17 widgetBackgroundChannel
lib/core/widget/widget_feed_loader.dart:36 widgetCoverMaxBytes
lib/core/widget/widget_feed_loader.dart:40 widgetSnapshotMaxTotalImageBytes
lib/core/widget/widget_snapshot.dart:11 widgetSnapshotMaxTextLength
lib/core/widget/widget_snapshot.dart:12 widgetSnapshotMaxAccountKeyLength
lib/core/widget/widget_snapshot_store.dart:18 widgetSnapshotDirectory
lib/core/widget/widget_snapshot_store.dart:251 WidgetSnapshotWriteError
lib/features/comments/comment_input.dart:6 CommentComposerPanel
lib/features/comments/comments_page.dart:29 showCommentReplies
lib/features/history/history_page.dart:32 HistoryPage
lib/features/home/recommended/recommended_feed_controller.dart:14 RecommendedFeedKey
lib/features/home/recommended/recommended_feed_controller.dart:18 RecommendedFeedController
lib/features/home/recommended/recommended_home_page.dart:151 RecommendedFeedView
lib/features/illust/detail/illust_detail_controller.dart:44 IllustDetailController
lib/features/illust/detail/illust_detail_controller.dart:94 illustDetailRepositoryProvider
lib/features/illust/detail/illust_detail_page.dart:55 kHomeBottomNavHeight
lib/features/illust/detail/illust_download_controller.dart:15 IllustDownloadController
lib/features/illust/detail/related_illust_repository.dart:12 RelatedIllustPage
lib/features/illust/detail/related_illust_repository.dart:83 RelatedIllustController
lib/features/illust/detail/related_illust_repository.dart:128 relatedIllustRepositoryProvider
lib/features/new/new_page.dart:168 NewFeedBody
lib/features/novel/novel_layout.dart:101 NovelLayoutProgressCallback
lib/features/novel/novel_layout.dart:127 NovelLayoutKey
lib/features/novel/novel_layout.dart:174 NovelPageLine
lib/features/novel/novel_page.dart:36 novelDetailProvider
lib/features/novel/novel_page.dart:47 novelSeriesProvider
lib/features/novel/novel_reader.dart:14 NovelReaderLayoutContext
lib/features/novel/novel_reader.dart:32 NovelReaderDiscardReason
lib/features/profile/profile_feed_controller.dart:13 ProfileIllustFeedController
lib/features/profile/profile_feed_controller.dart:89 ProfileUserFeedController
lib/features/profile/user_detail_controller.dart:42 UserDetailController
lib/features/ranking/ranking_repository.dart:40 RankingIllustPage
lib/features/ranking/ranking_repository.dart:48 RankingRepository
lib/features/ranking/ranking_repository.dart:107 rankingRepositoryProvider
lib/features/ranking/ranking_repository.dart:112 RankingFeedController
lib/features/search/search_filter_sheet.dart:17 SearchFilterSheet
lib/features/search/search_page.dart:111 SearchPage
lib/features/search/search_page.dart:416 SearchAutocompletePanel
lib/features/search/tag_search_page.dart:9 TagSearchPage
lib/features/search/tag_search_repository.dart:9 TagSearchController
lib/features/settings/settings_page.dart:571 LanguageSettingsPage
lib/features/settings/settings_page.dart:618 TranslateSettingsPage
lib/features/settings/settings_page.dart:711 TranslationCredentialsPage
lib/features/settings/settings_page.dart:989 BrowseSettingsPage
lib/features/settings/settings_page.dart:1150 DownloadSettingsPage
lib/features/settings/settings_page.dart:1360 DownloadDestinationPage
lib/features/settings/settings_page.dart:1513 HistorySettingsPage
lib/features/settings/settings_page.dart:1570 BlockedTagsPage
lib/features/settings/settings_page.dart:1649 DownloadTasksPage
```

词边界计数，同名不同实体可能误判；C9 私有化前需逐项看。

---

## 附录 B. 复现脚本

脚本只存在于 `/tmp/c-recount/`（不进仓库）。LOC/简单模式用正文里的 `find`/`wc`/`rg`。下面三份是 import 图、死声明、方法长度。

### B.1 `imports_only.py`

```python
#!/usr/bin/env python3
"""Import graph on unmasked source (package: + relative), parent B3 method."""
from __future__ import annotations

import json
import re
from collections import Counter, defaultdict
from pathlib import Path

ROOT = Path("/root/Pixiv-func-C2")
LIB = ROOT / "lib"
TEST = ROOT / "test"
OUT = Path("/tmp/c-recount/out")
PACKAGE = "pixiv_func"
IMPORT_RE = re.compile(r"""^\s*(import|export)\s+['\"]([^'\"]+)['\"]""", re.M)
PART_RE = re.compile(r"""^\s*part\s+['\"]([^'\"]+)['\"]""", re.M)


def dart_files(base: Path) -> list[Path]:
    return sorted(p for p in base.rglob("*.dart") if p.is_file())


def rel(p: Path) -> str:
    return str(p.relative_to(ROOT)).replace("\\", "/")


def resolve_uri(from_file: Path, uri: str) -> Path | None:
    if uri.startswith(f"package:{PACKAGE}/"):
        cand = LIB / uri[len(f"package:{PACKAGE}/") :]
        return cand if cand.exists() else None
    if uri.startswith(("package:", "dart:")):
        return None
    cand = (from_file.parent / uri).resolve()
    return cand if cand.exists() else None


def top(path: str) -> str:
    parts = path.split("/")
    if not path.startswith("lib/"):
        return "test"
    if len(parts) >= 2 and parts[1] in ("core", "features", "app"):
        return parts[1]
    return "lib-root"


def feat(path: str) -> str | None:
    parts = path.split("/")
    if len(parts) >= 3 and parts[0] == "lib" and parts[1] == "features":
        return parts[2]
    return None


def main() -> None:
    files = dart_files(LIB) + dart_files(TEST)
    edges = []
    imported_by: dict[str, set[str]] = defaultdict(set)
    for p in files:
        src = p.read_text(encoding="utf-8")
        fr = rel(p)
        for m in IMPORT_RE.finditer(src):
            kind, uri = m.group(1), m.group(2)
            tgt = resolve_uri(p, uri)
            if tgt is None:
                continue
            to = rel(tgt)
            ln = src.count("\n", 0, m.start()) + 1
            edges.append({"from": fr, "to": to, "kind": kind, "uri": uri, "line": ln})
            imported_by[to].add(fr)
        for m in PART_RE.finditer(src):
            uri = m.group(1)
            tgt = resolve_uri(p, uri)
            if tgt is None:
                continue
            to = rel(tgt)
            ln = src.count("\n", 0, m.start()) + 1
            edges.append({"from": fr, "to": to, "kind": "part", "uri": uri, "line": ln})
            imported_by[to].add(fr)

    layer = Counter()
    core_to_feat = []
    feat_cross = []
    app_onb = []
    onb_app = []
    lib_edges = 0
    for e in edges:
        if not (e["from"].startswith("lib/") and e["to"].startswith("lib/")):
            continue
        lib_edges += 1
        a, b = top(e["from"]), top(e["to"])
        layer[f"{a}→{b}"] += 1
        if a == "core" and b == "features":
            core_to_feat.append(e)
        if a == "features" and b == "features":
            fa, fb = feat(e["from"]), feat(e["to"])
            if fa and fb and fa != fb:
                feat_cross.append({**e, "from_feat": fa, "to_feat": fb})
        if a == "app" and e["to"].startswith("lib/features/onboarding/"):
            app_onb.append(e)
        if e["from"].startswith("lib/features/onboarding/") and b == "app":
            onb_app.append(e)

    print("lib_edges", lib_edges)
    print("layer", dict(layer))
    print("core→feat", len(core_to_feat), core_to_feat)
    print("feat×feat", len(feat_cross))
    print("app→onb", app_onb)
    print("onb→app", len(onb_app))


if __name__ == "__main__":
    main()
```

### B.2 `dead_tight.py`

```python
#!/usr/bin/env python3
"""Public top-level decls: names appearing once in lib+test, or only in own file."""
from __future__ import annotations

import json
import re
from collections import defaultdict
from pathlib import Path

ROOT = Path("/root/Pixiv-func-C2")
LIB = ROOT / "lib"
TEST = ROOT / "test"
OUT = Path("/tmp/c-recount/out")

TYPE_RE = re.compile(
    r"^(?:abstract\s+|base\s+|final\s+|interface\s+|sealed\s+|mixin\s+)*"
    r"(?:class|enum|mixin)\s+([A-Za-z_][A-Za-z0-9_]*)",
    re.M,
)
TYPEDEF_RE = re.compile(r"^typedef\s+([A-Za-z_][A-Za-z0-9_]*)", re.M)
EXT_RE = re.compile(r"^extension\s+([A-Za-z_][A-Za-z0-9_]*)\s+on\s+", re.M)
VAR_RE = re.compile(
    r"^(?:const|final)\s+(?:[A-Za-z_][\w.<>,\s?]*\s+)?"
    r"([A-Za-z_][A-Za-z0-9_]*)\s*[;=]",
    re.M,
)
FN_RE = re.compile(
    r"^(?:[A-Za-z_][\w.<>,\s?]*\s+)([A-Za-z_][A-Za-z0-9_]*)\s*(?:<[^>\n]*>)?\s*\([^;]*\)\s*(?:async\s*)?\{",
    re.M,
)
SHOW_RE = re.compile(r"^(?:void\s+)?(show[A-Za-z0-9_]+)\s*\(", re.M)
SKIP = {
    "if", "for", "while", "switch", "catch", "main", "void", "int", "bool",
    "double", "String", "dynamic", "widget", "build", "createState",
}


def dart_files(base: Path):
    return sorted(p for p in base.rglob("*.dart") if p.is_file())


def rel(p: Path) -> str:
    return str(p.relative_to(ROOT)).replace("\\", "/")


def decls_in(src: str, path: str):
    out, seen = [], set()
    for pat, kind in (
        (TYPE_RE, "type"),
        (TYPEDEF_RE, "typedef"),
        (EXT_RE, "extension"),
        (VAR_RE, "var"),
        (FN_RE, "fn"),
        (SHOW_RE, "show"),
    ):
        for m in pat.finditer(src):
            name = m.group(1)
            if not name or name.startswith("_") or name in SKIP:
                continue
            ln = src.count("\n", 0, m.start()) + 1
            line = src.splitlines()[ln - 1].strip()
            if line.startswith(("import ", "export ", "part ", "library ")):
                continue
            if kind == "fn" and name[0].isupper():
                continue
            if name in seen:
                continue
            seen.add(name)
            out.append({"file": path, "name": name, "line": ln, "kind": kind})
    return out


def main() -> None:
    lib_files = dart_files(LIB)
    texts = {rel(p): p.read_text(encoding="utf-8") for p in lib_files + dart_files(TEST)}
    decls = []
    for p in lib_files:
        decls.extend(decls_in(texts[rel(p)], rel(p)))
    needed = {d["name"] for d in decls}
    index = defaultdict(lambda: defaultdict(int))
    ident = re.compile(r"\b[A-Za-z_][A-Za-z0-9_]*\b")
    for fr, src in texts.items():
        for tok in ident.findall(src):
            if tok in needed:
                index[tok][fr] += 1
    once, own = [], []
    for d in decls:
        files = index.get(d["name"], {})
        total = sum(files.values())
        if total == 1:
            once.append(d)
        elif total >= 2 and list(files.keys()) == [d["file"]]:
            own.append({**d, "occurrences": total})
    print("once", len(once), "own", len(own))
    (OUT / "dead_tight.json").write_text(
        json.dumps({"once": once, "own_file_only": own}, indent=2),
        encoding="utf-8",
    )


if __name__ == "__main__":
    main()
```

### B.3 `methods_named.py`

```python
#!/usr/bin/env python3
"""Method length; braces after stripping strings/comments; supports {named params}."""
from __future__ import annotations

import json
import re
from pathlib import Path

ROOT = Path("/root/Pixiv-func-C2")
OUT = Path("/tmp/c-recount/out")
STRING_OR_COMMENT = re.compile(
    r"(//[^\n]*)|(/\*.*?\*/)|(r?'''(?:\\.|[^\\])*?''')|(r?\"\"\"(?:\\.|[^\\])*?\"\"\")"
    r"|(r?'(?:\\.|[^'\\])*')|(r?\"(?:\\.|[^\"\\])*\")",
    re.S,
)
SKIP = {
    "if", "for", "while", "switch", "catch", "on", "assert", "return", "throw",
    "await", "yield", "new", "super", "this", "case", "default", "else", "try",
    "finally", "do", "in", "is", "as", "rethrow", "break", "continue",
}
HEAD = re.compile(
    r"""(?mx)^[ \t]*(?:(?:static|external|covariant|factory|const|final|late|override)\s+)*
    (?:[\w.<>,\s?]+\s+)?(?P<name>~?[A-Za-z_][A-Za-z0-9_]*)\s*(?:<[^>\n]*>)?\s*\(""",
)


def mask(src: str) -> str:
    return STRING_OR_COMMENT.sub(
        lambda m: "".join("\n" if c == "\n" else " " for c in m.group(0)), src
    )


def match_braces(src: str, open_idx: int):
    depth = 0
    i, n = open_idx, len(src)
    while i < n:
        if src[i] == "{":
            depth += 1
        elif src[i] == "}":
            depth -= 1
            if depth == 0:
                return i
        i += 1
    return None


def methods(path: Path):
    masked = mask(path.read_text(encoding="utf-8"))
    rel = str(path.relative_to(ROOT))
    out = []
    for m in HEAD.finditer(masked):
        name = m.group("name")
        if name in SKIP:
            continue
        i, depth, n = m.end() - 1, 0, len(masked)
        while i < n:
            if masked[i] == "(":
                depth += 1
            elif masked[i] == ")":
                depth -= 1
                if depth == 0:
                    i += 1
                    break
            i += 1
        else:
            continue
        rest = masked[i : i + 80]
        mm = re.match(r"\s*(?:async\s*)?(?:sync\*\s*|async\*\s*)?\{", rest)
        if not mm:
            continue
        end = match_braces(masked, i + mm.end() - 1)
        if end is None:
            continue
        start_line = masked.count("\n", 0, m.start()) + 1
        end_line = masked.count("\n", 0, end) + 1
        nlines = end_line - start_line + 1
        if nlines > 150:
            out.append({"file": rel, "name": name, "start": start_line, "end": end_line, "lines": nlines})
    return out


def main() -> None:
    allm = []
    for p in sorted((ROOT / "lib").rglob("*.dart")):
        allm.extend(methods(p))
    allm.sort(key=lambda x: -x["lines"])
    for m in allm:
        print(f"{m['lines']:4} {m['file']}:{m['start']}-{m['end']} {m['name']}")
    (OUT / "long_methods3.json").write_text(json.dumps(allm, indent=2), encoding="utf-8")


if __name__ == "__main__":
    main()
```

跑法：`python3 /tmp/c-recount/imports_only.py`；`python3 /tmp/c-recount/dead_tight.py`；`python3 /tmp/c-recount/methods_named.py`。JSON 落在 `/tmp/c-recount/out/`。
