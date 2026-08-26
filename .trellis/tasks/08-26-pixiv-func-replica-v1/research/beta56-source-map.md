# beta56 固定源码映射

## 固定参考

- 仓库：`svenfuss/pixiv_func_mobile`
- Commit：`c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`
- 核验日期：2026-08-26
- 用途：仅作为 Replica v1 的可观察布局、交互和生命周期基准；安全、平台和网络内部实现以当前规范与实时核验为准。

历史源码中的 OAuth 凭据、固定密钥、Cookie、固定 IP 和其他敏感值不在本仓库复制。以下路径均相对于固定 commit。

## 任务到源码映射

| Trellis 任务 | beta56 主要源码锚点 | 规划时确认的行为 |
|---|---|---|
| `restore-icon-font` | `assets/icon.ttf`、`lib/app/icon/icon.dart`、`pubspec.yaml` | `iconFont` 与 `0xe900`–`0xe90d` codepoint |
| `secure-account-store` | `lib/models/account.dart`、`lib/app/services/account_service.dart` | 多账号可见行为；旧普通偏好存储不复用 |
| `oauth-pkce-webview-login` | `lib/pages/login/`、`lib/app/api/auth_client.dart` | Login/WebView/回调流程；旧 TLS 放行和 Cookie 输出不复用 |
| `pixiv-network-token-refresh` | `lib/app/api/`、`lib/app/http.dart` | API/auth 客户端边界；当前身份值与 endpoint 必须实时核验 |
| `recommended-feed-paging` | `lib/pages/recommended/`、`lib/components/illust_previewer/` | 推荐内容、卡片和分页入口 |
| `illust-detail-viewer` | `lib/pages/illust/illust.dart`、`controller.dart`、`scale/scale.dart` | 详情、多页查看、缩放和下载模式入口 |
| `bookmark-state-sync` | `lib/components/bookmark_switch_button/` | 短按、长按和收藏状态反馈 |
| `android-platform-parity` | `android/app/src/main/AndroidManifest.xml`、`MainActivity.kt`、`lib/app/platform/`、`lib/app/url_scheme/` | intent、WebView、平台桥与 URL scheme 的原版入口 |
| `ranking-feed` | `lib/pages/ranking/` | 11 类榜单 tab 与独立分页 |
| `new-content-feeds` | `lib/pages/new/` | Following、Everyone、My Pixiv 与类型切换 |
| `search-catalog` | `lib/pages/search_guide/`、`lib/pages/search/`、`lib/pages/search/result/{illust,novel,user}/` | 搜索首页、趋势标签、三类结果和过滤 |
| `reverse-image-search` | `lib/pages/search/result/image/` | 图片选择、外部检索和 Pixiv 结果路由；外部服务必须实时核验 |
| `user-profile-follow` | `lib/pages/user/`、`lib/components/follow_switch_button/` | 用户页、作品/收藏/关系 tab、关注状态 |
| `profile-edit` | `lib/pages/user/me_settings/` | 个人资料、工作区和 Web 编辑入口 |
| `comments-replies` | `lib/pages/illust/comment/`、`lib/components/comment_input/`、`comment_item/` | 评论、回复、emoji/stamp 与删除 |
| `history-persistence` | `lib/pages/history/`、`lib/app/db/history_db.dart` | 历史入口和可见性语义；旧逐操作开库/整对象 JSON 不复用 |
| `settings-parity` | `lib/pages/settings/`、`lib/app/services/settings_service.dart`、`lib/models/settings.dart` | 原版设置项、默认值与导航 |
| `novel-reader` | `lib/pages/novel/`、`lib/components/novel_viewer/` | 小说详情和水平阅读 |
| `download-manager-mediastore` | `lib/pages/downloader/`、`lib/app/downloader/downloader.dart`、`lib/models/download_task.dart` | 队列、并发和状态；旧整块加载实现不复用 |
| `ugoira-player-export` | `lib/pages/illust/ugoira_viewer/`、`lib/components/frame_gif/` | Ugoira 播放和 GIF 导出；改为有界解码 |
| `restricted-compat-network` | `lib/app/api/web_api_client.dart`、`lib/app/platform/webview/` | 兼容路径的用户意图；固定 IP、全局证书放行和 SSL proceed 禁止复用 |
| `secure-clipboard-account-migration` | `lib/pages/login/login.dart`、`lib/app/encrypt/encrypt.dart` | 剪贴板导入/导出入口；旧静态加密方案不复用 |
| `live-player` | `lib/pages/live/`、`lib/pages/recommended/live/`、`lib/components/live_previewer/` | 16:9、控制层、清晰度、全屏和作者入口；接口当日核验 |
| `android-home-widgets` | `android/app/src/main/kotlin/moe/xiaocao/pixiv/appwidget/`、对应 `res/layout` 与 `res/xml` | 推荐/刷新小组件；秘密不得进入普通偏好 |
| `updater-flavors` | `lib/app/updater/updater.dart`、`android/app/src/main/AndroidManifest.xml` | GitHub 更新入口；权限改由 flavor 隔离 |

## 使用规则

1. 每个叶子任务开始时再次打开其源码锚点，并把更精确的行为证据写入该任务 `research/`。
2. OAuth、Pixiv API、反向搜图、Live、Android/WebView API 等时效性结论必须在实现开始当天核验；无法核验时保持明确 blocker。
3. 固定源码决定 Replica 行为，不授权恢复 TLS 绕过、明文秘密、无界内存、固定 IP 或全局高风险权限。
