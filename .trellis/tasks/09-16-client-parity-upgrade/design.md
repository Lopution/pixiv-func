# 设计：跨子任务共享契约

> 本文档只写跨 child 的共享面与归属裁决；各 child 的详细设计在各自 `design.md`。

## 1. 分层归属（强制，layering_test 执行）

| 新域 | 归属 | 说明 |
|---|---|---|
| watch-later | `core/watchlater/` + `features/watchlater/` | 本地暂存记录 + 列表页 |
| mute | `core/mute/`（store + repository + 服务端同步适配） | 屏蔽是唯一 owner；`settings/local_block_filter` 收敛进新域或保留为 R18/AI 谓词，由 child 3 定 |
| bookmark tags | `core/bookmark/` 扩展 | tag 列表走 BookmarkStore 协议扩展，不新建全局 store |
| illust series / pixivision | `core/illust/`、`core/spotlight/` 新域 | series 与 pixivision 是两个端点域 |
| 多引擎识图 | `core/reverse_image/` 扩展 provider 抽象 | 现有 `reverse_image_provider.dart` 抽象外新增 TinEye/IQDB/Ascii2D provider |
| feed 快照 | `core/paging/` + `core/history/` 复用 SQLite | 快照是 feed 层能力，不是新 store |
| 离线动作队列 | `core/mutation/` 扩展（持久化 envelope + replay） | 不复用 download recovery，队列语义独立 |
| 本地小说库 / TXT 导出 | `core/novel/` 扩展 | 本地文件导入与线上实体共用排版管线 |
| 动效 | `app/motion/` 扩展 | 唯一天然跨层动效 owner，见 §4 |
| 双栏布局 | `app/layout/` 扩展 | 断点与 two-pane 容器，feature 页面只声明内容 |

## 2. 卡片交互面（child 2/3/12 共用，必须一次定型）

- `IllustCard`(`app/widgets/feed/illust_card.dart`）当前只有 `onTap`+`onTapDown` 预加载；新增**长按**手势槽，菜单渲染归属 app 层，动作分发到 core 域 provider（bookmark/download/watchlater/mute/share)。
- 长按菜单的项由「动作注册表」生成：每个域注册 `label/icon/enabled/onSelected`，而不是在 card 里堆 if——后续 mute/watchlater 各自把自己的项注册进去，不互相 import。
- 屏蔽后卡片呈现「模糊揭示」态（blur + 揭示按钮），属于 `IllustCard` 的一个渲染变体，由 `MuteStore` 的订阅位驱动；本地过滤谓词从"隐藏"改为"标记模糊"，隐藏开关保留为设置项。

## 3. 数据层共享决策

- **Mute 服务端同步**：`/v1/mute/list|edit`（pixes 已验证端点）。MuteStore = 服务端为权威 + 本地覆盖（单作品屏蔽官方不支持，仅存本地）。revision 协议沿用 BookmarkStore 模式。
- **feed 快照**：序列化 `{feedKey, accountId, ids, entities, cursor, timestamp}` 进 SQLite（复用 `history_database` 的打开/迁移路径或新建 `snapshot_database`——child 10 设计时定，倾向独立 db 文件）。恢复时机：feed provider 构造时同步读快照作为 initial IDs，再走正常刷新；过期阈值与容量上限必须显式定义。
- **离线动作队列**：mutation envelope 持久化表 `(id, accountId, kind, payload, revision, createdAt, attempt, nextRetryAt)`；在线路径不变，离线/5xx 入队；重放走同一 repository 调用，不旁路。
- **下载 Range 续传**：`DownloadSink` 增加 `resumeFrom` 字节偏移；transport 发 `Range: bytes=<offset>-`；部分文件保留策略与命名后缀（`.part`）在 child 9 的 design 定，禁止破坏现有 MediaStore/SAF 语义。

## 4. 动效层（child 12 的边界）

- 现状：`app/motion/` 已有 `hero_transition`/`hero_rect_clip`/`drag_to_dismiss`/`motion_tokens`（时长/曲线单一来源）。
- 补齐目标：页面转场构建器（go_router `pageBuilder` 统一入口）、列表项进场 stagger、卡片按压缩放/反馈、sheet 与长按菜单弹出曲线、查看器页间过渡。全部时长/曲线/距离进 `motion_tokens`，页面/组件不写字面量。
- 降级：设置项"减少动态效果"开关 + 系统 `MediaQuery.disableAnimations` 合并判定，单 owner。
- 不动：hero 图片转场与 viewer 手势的已验证行为；不动效装饰加载失败态。

## 5. 端点与契约备注

- 新端点统一进 `pixiv_http_client` 的 host/path 白名单校验路径，cursor `next_url` 走既有 allowlist。
- `/v1/search/popular-preview/*` 已存在；`popular_desc` 已有。性别向/收藏数排序若需 Premium 参数，子任务 4 先验证非会员行为再定 UI 呈现。
- pixivision 文章：`/v1/spotlight/articles` + 文章详情为 web 富文本——复用 PixEz `soup_page` 思路：API 拿元数据 + 结构化渲染，避免整页 webview。

## 6. 测试与验证基线

- 每个新 store/repository：合并矩阵、revision 丢弃、账号边界、cursor 校验测试（对照现有 `illust_store`/`comment` 测试模板）。
- 卡片长按与模糊揭示：widget 测试覆盖手势与动作分发；动效只断言 token 使用与开关降级，不做逐帧断言。
- 快照/离线队列：用 fake clock + fake transport 断言重放、过期、容量淘汰。
- 小说修复必须先写**可复现的失败证据**（报错/空态路径），修复后加回归测试；不允许绕过根因。

## 7. 回滚

每个 child 独立 PR；整体回滚 = `git revert -m 1 <merge-sha>` 逐 child 回退。数据迁移（SQLite 新表）必须可向后兼容：旧版本读到新表存在不得崩溃；删除表走显式迁移而非静默 drop。
