# 用户原始需求追踪表

## Purpose

本文件把用户提供的 40 节项目指令映射到父 PRD 与实际 Trellis 任务，确保委派时不丢失范围。原始材料中出现的历史 OAuth 客户端值不在仓库中复制；实现时必须从当前可信来源重新核验，并集中维护。

## Actual task registry

| 编号 | 实际任务目录 | 类型或叶子 |
|---|---|---|
| 0 | `08-26-flutter-android-scaffold` | 已归档叶子 |
| 1 | `08-26-restore-icon-font` | 叶子 |
| 2 | `08-26-secure-account-store` | 叶子 |
| 3 | `08-26-oauth-pkce-webview-login` | 叶子 |
| 4 | `08-26-pixiv-network-token-refresh` | 叶子 |
| 5 | `08-26-recommended-feed-paging` | 叶子 |
| 6 | `08-26-illust-detail-viewer` | 叶子 |
| 7 | `08-26-bookmark-state-sync` | 叶子 |
| 8 | `08-26-android-platform-parity` | 叶子 |
| 9 | `08-26-discovery-search-reverse-image` | 中间父任务：`ranking-feed`、`new-content-feeds`、`search-catalog`、`reverse-image-search` |
| 10 | `08-26-profile-follow-social` | 中间父任务：`user-profile-follow`、`profile-edit` |
| 11 | `08-26-comments-history-settings` | 中间父任务：`settings-parity`、`comments-replies`、`history-persistence` |
| 12 | `08-26-novel-reader` | 叶子 |
| 13 | `08-26-downloads-ugoira-media` | 中间父任务：`download-manager-mediastore`、`ugoira-player-export` |
| 14 | `08-26-compat-network-account-migration` | 中间父任务：`restricted-compat-network`、`secure-clipboard-account-migration` |
| 15 | `08-26-live-widgets-updater` | 中间父任务：`live-player`（已移出范围）、`android-home-widgets`、`updater-flavors` |
| 16 | `08-26-replica-v1-integration-release` | 叶子 |

状态注记：截至 2026-08-27，原 Replica 树的 `08-26-ugoira-player-export` 是当前唯一 `in_progress` 实现叶子；`08-26-restricted-compat-network` 是下一个 P0 `planning` 候选，另有 8 个原树实现叶子仍为 `planning`。审查后新增的 5 个 hardening leaves 属于独立 top-level `08-27-replica-v1-hardening`，不计入本表 17 项矩阵。

## Traceability

| 原始章节 | 约束主题 | 父任务位置 | 计划执行任务 |
|---|---|---|---|
| 一 | 现代内部实现、原样复刻可见体验 | PRD R1；Design §2、§7 | 所有子任务 |
| 二 | beta56 参考 commit、AGPL 与原作者归属 | PRD R1、风险；Design §2 | 每个 feature 的 research；任务 16 |
| 三 | 当前 UI shell、设置、icon codepoint 与缺口 | PRD 背景、R2 | 任务 0、1、2 |
| 四 | Flutter 3.47 / API 36 / AGP 9 工程基线 | PRD R2、AC1 | 任务 0 |
| 五 | edge-to-edge、back、deep link、WebView、MediaStore、SEND | PRD R7、AC7 | 任务 8、13、14、15 |
| 六 | 主题颜色、字号与 Material 行为冻结 | PRD R1、R2 | 任务 1 及各 UI 子任务 |
| 七 | 冷启动三态、Welcome/Language/Theme 顺序 | PRD R2、R3、AC2 | 任务 2；任务 16 回归 |
| 八 | 右进左出导航、root 双击 back | PRD R1、R7 | 任务 8 |
| 九 | Home BottomAppBar、tab 顺序、图标和状态保持 | PRD R2、R6 | 任务 1、5、8 |
| 十 | Login 可见布局、help 与剪贴板入口 | PRD R4 | 任务 3、14 |
| 十一 | OAuth PKCE、WebView 回调与 TLS 失败 | PRD R4、AC3 | 任务 3 |
| 十二 | Account/Credential/AccountStore、多账号与安全存储 | PRD R3、AC2 | 任务 2 |
| 十三 | per-account single-flight refresh 和一次重试 | PRD R5、AC4 | 任务 4 |
| 十四 | 统一 Pixiv Client Identity 且重新核验时效值 | PRD R5、风险 | 任务 4 research |
| 十五 | Dio/统一网络边界、系统 DNS、严格 TLS | PRD R5 | 任务 4 |
| 十六 | 受限 compatibility 网络与 WebView ProxyController | PRD R5、R7 | 任务 14 |
| 十七 | 第一条纵向链及共享 entity/store | PRD R6、R12 | 任务 5、6、7 |
| 十八 | Bookmark 短按/长按/pending/failure/同步 | PRD R6、AC5 | 任务 7 |
| 十九 | Paging 状态、host 限制与 ID 去重 | PRD R5、R6、AC4 | 任务 5 |
| 二十 | Illust Detail、badges、tags 与下载模式 | PRD R6 | 任务 6、13 |
| 二十一 | Viewer 横向分页、标题与 zoom | PRD R6 | 任务 6 |
| 二十二 | 第一条链之后的功能顺序 | PRD R8、R12 | 任务 9–15 |
| 二十三 | Search 首页、trending、tabs、ID 与 debounce | PRD R8 | 任务 9 |
| 二十四 | Comments、emoji/stamp/reply/delete 与 ID 修复 | PRD R8 | 任务 11 |
| 二十五 | Novel 当前 API、水平阅读与稳定 anchor | PRD R8 | 任务 12 |
| 二十六 | Ugoira 有界缓存、播放生命周期与 GIF | PRD R9 | 任务 13 |
| 二十七 | 流式下载队列、并发 3、MediaStore 与进度 | PRD R7、R9 | 任务 13 |
| 二十八 | History 单 DB 生命周期、紧凑 schema 与可见性计时 | PRD R9 | 任务 11 |
| 二十九 | Profile header 折叠状态与 action | PRD R8 | 任务 10 |
| 三十 | ~~Live player 行为~~ | **2026-08-29 移出范围** | 不再有 owning task |
| 三十一 | 剪贴板迁移安全封装与原版 UX | PRD R3 | 任务 14 |
| 三十二 | 原版默认设置与固定 IP 降级边界 | PRD R10 | 任务 11、13、14 |
| 三十三 | GitHub/F-Droid updater flavor 权限 | PRD R7、R10 | 任务 15 |
| 三十四 | Replica v1 不扩展原版未完成功能 | PRD R10、Out of Scope | 所有子任务；任务 16 审计 |
| 三十五 | 每阶段命令与 Startup/OAuth/Token/Paging/Bookmark/Viewer/Android 测试 | PRD R11；Implement §6 | 每个子任务；任务 16 |
| 三十六 | 不伪造 analyze/build/device-tested 状态 | PRD R11 | 每个子任务验收与汇报 |
| 三十七 | A–H 执行顺序 | PRD R12；Implement §2、§3 | 任务 0–7 |
| 三十八 | inspect、验证、小步提交、不重写历史 | PRD R12；Implement §4、§7 | 每个子任务 |
| 三十九 | 可见行为以 beta56 为准，安全/兼容以现代正确实现为准 | PRD R1、风险；Design §2 | 所有子任务 |
| 四十 | 报告完成、commit、检查、链路、blocker 与下一步 | PRD R11；Implement §4、§5 | 每个子任务 finish-work |

## Child-planning rule

本表只证明范围已纳入父任务，不代替 feature 级行为规格。开始每个已有叶子任务时，必须从 `beta56-source-map.md` 对应源码提取更精确的布局、交互、错误和生命周期证据，并写入该任务 `research/`；发现与父 PRD 实质冲突时应回到父任务重新审阅。
