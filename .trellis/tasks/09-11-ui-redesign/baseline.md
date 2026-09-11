# 阶段 0 基线：全项目 UI 视觉与交互重构

记录于任务分支 `task/09-11-ui-redesign` HEAD（`f66b96b`，main 含 PR #15/#16 合并）。`baseline-2026-09-07` tag 仅作历史参照，本文件为当前回归基线。

## 质量命令基线

| 命令 | 结果 |
| --- | --- |
| `flutter analyze --no-pub` | No issues found（11.7s） |
| `flutter test` | 718 全部通过（~32s） |
| `git diff --check` | clean |
| `dart format --set-exit-if-changed .` | clean（本分支提交前已验） |

## Golden / 视觉证据现状

- 唯一 golden：`test/goldens/home_bar.png`（`icon_font_test.dart:122`，`matchesGoldenFile`）。
- 无 golden harness、无跨主题/宽度/语言样例目录——阶段 5 需要先补基建。

## 路由与状态恢复清单

- Shell branches（5 个，各带 `restorationScopeId`）：`/recommended`、`/ranking`、`/new`、`/search`、`/settings`（settings 带子路由树 + `includeHistory: false`）。
- Root-level stack roots：`_stackRoots = {/recommended, /ranking, /new, /search, /settings, /reverse-image}`。
- `_commonBranchRoutes` 挂于每个 root：`illust/:id`、`user/:id`、`novel/:id`、`tag/:keyword`、`history`、`me` 等。
- 恢复：`restorationScopeId` 'pixiv-func'（MaterialApp）→ 'router' → 'home-shell' → 各 branch；各 feed 使用 `PageStorageKey`（recommended/ranking/new/search/history/profile/search-result 均有）。
- 测试面：`navigation_router_test.dart`、`navigation_restoration_test.dart`、`intent_router_test.dart`、`login_navigation_test.dart`、`home_page_test.dart`。

## 共享组件盘点

详表见 `design.md` §0。要点：`feed/`（IllustFeedGrid/IllustCard/FeedTail/FeedEmpty/FeedError，**无 FeedLoading**）、`replica_*`、`settings/` tiles、`novel_card`、switch 按钮、`app_snack_bar`、`person_avatar`、`pull_to_refresh`、`pixiv_image`。列数函数 `_illustColumnsFor` 为 private 且吃 `MediaQuery` 整窗宽。

## 高风险业务消费者（只许改呈现/入口，不许改语义）

- 设置行为与 `/settings/...` 深链（11 个子页路由树）。
- 下载恢复与 MediaStore/SAF 落盘（`download_manager`、`saf_tree`）。
- Ugoira 播放/时间轴与可见性检测（`ugoira_viewer`）。
- 阅读位置与历史持久化（`history_database`）。
- OAuth/凭据（`oauth_service`、secure storage）。
- Hero flight 依赖 `homeShellMetricsProvider` 的底栏实测值。
- 账号迁移 payload（`account_transfer_*`）。

## 视觉矩阵命名规则（阶段 5 用）

- 目录：`test/goldens/<family>/<page>/`，family ∈ `sample-chain`（推荐→详情→作者→我的/收藏，走完整矩阵）、`feed`、`detail`、`profile`、`search`、`settings`、`novel`、`misc`。
- 文件：`<page>_<theme>_<width>_<lang>_<state>.png`，theme ∈ `light|dark`，width ∈ `320|390|600|840|1200`（横屏另记），lang ∈ `zh|en|ja|ru`，state ∈ `content|loading|empty|error|restricted|ugoira`。
- 非样板链路页面只取代表状态子集；新样例必须写明覆盖的回归风险。

## 不触碰清单（本任务全程）

Pixiv API/网络策略、OAuth/TLS、账号凭据、缓存语义、下载恢复、Ugoira 播放、阅读位置、业务数据流与 mock 边界。
