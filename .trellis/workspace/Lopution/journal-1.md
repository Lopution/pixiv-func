# Journal - Lopution (Part 1)

> AI development session journal
> Started: 2026-08-26

---



## Session 1: Illust detail viewer closeout: waterfall feed parity fix + MuMu verification

**Date**: 2026-08-27
**Task**: Illust detail viewer closeout: waterfall feed parity fix + MuMu verification
**Branch**: `main`

### Summary

Fixed recommended feed layout to beta56 two-column waterfall (SliverMasonryGrid.count, full-aspect fitWidth previews) after real-device screenshots showed fixed 0.72 tile cropping tall portraits; restored feed Hero tags; accepted real next_url viewed[n] params; split flaky loopback tests when running full suite; verified detail/viewer/back/scroll-restore on MuMu (note: MuMu display overridden to landscape coords, ADB input needs 1920x1080 space). Task 08-26-illust-detail-viewer archived.

### Git Commits

| Hash | Message |
|------|---------|
| `ca0d2da` | (see git log) |

### Status

[OK] **Completed**


## Session 2: bookmark-state-sync：BookmarkStore + beta56 收藏按钮真机验收

**Date**: 2026-08-27
**Task**: bookmark-state-sync：BookmarkStore + beta56 收藏按钮真机验收
**Branch**: `main`

### Summary

Session summary was not supplied.

### Main Changes

# bookmark-state-sync 实现与真机验收

## Main Changes
- 新建 `lib/core/bookmark/`：BookmarkStore（账号+实体类型+实体 ID 键控）承载 confirmed restrict / pending operation / previous state / error，begin 非乐观且同 key 去重，commit/fail 校验 operation revision 防晚到响应覆盖，observeRemote 以 snapshotRevision 门控旧快照
- `IllustStore` 通过 `onConfirmed` 闭包单向同步实体 `isBookmarked`（避免 Riverpod 循环依赖）；三处 fetch 点（recommended / illust detail / tag search）请求前捕获 `bookmarkRevisionNow()` 传入 mergeAll 作快照门控
- `BookmarkSwitchButton` 复刻 beta56：feed 卡片标题行右侧（10px 缩进）+ 详情 app bar actions；短按 toggle（未收藏=public add / 已收藏=delete），长按仅未收藏且非 pending 时弹约 35% 屏高 restrict sheet（公开/私密 DropdownButton + 取消/确定胶囊）；pending 显示 24px CupertinoActivityIndicator
- 网络：`pixiv_http_client.dart` 把 400 且 body 含 `invalid_grant` 也作为 token 过期触发 single-flight refresh（真机上 /v1/illust/recommended 对过期 token 回 400 而非 401）；非 2xx 附带截断脱敏的响应体摘要进 `ApiHttpError.detail`
- spec 更新：state-management.md 增加 BookmarkStore mutation protocol、isBookmarked 权威合并方向、invalid_grant 网络契约

## Testing
- `flutter analyze` 干净；全量 145+ 测试分片通过（主套件 126 + download_manager 30 + oauth_service 13），新增 bookmark store 9 / flow 7 / widget 6 及 invalid_grant 2 例
- MuMu 真机（127.0.0.1:16384）验收：public add（短按空心→实心）、详情↔feed 跨页双向同步、详情短按 delete、长按 restrict sheet 私密 add、已收藏长按不开 sheet（Flutter tap 语义下 900ms 静止按压触发 toggle，与 beta56 onLongPress:null 行为一致）；服务端无残留
- 真机诊断过程：ApiHttpError(http 400) 经 body 摘要定位为 OAuth invalid_grant → 修复后 token 自动 refresh 恢复

## Next Steps
- 归档 bookmark-state-sync，继续父任务下一叶子 ranking-feed


### Git Commits

| Hash | Message |
|------|---------|
| `5f5f66f` | (see git log) |
| `8bae089` | (see git log) |

### Status

[OK] **Completed**


## Session 3: Comments and replies implementation

**Date**: 2026-08-27
**Task**: Comments and replies implementation
**Branch**: `main`

### Summary

完成评论与回复子任务：接入评论/回复分页、ID与thread store、非乐观发送/删除、emoji/stamp、翻译和详情入口；通过全量测试与 MuMu 只读评论 API 验证。

### Main Changes

- 新增 CommentEntity、CommentStore、CommentRepository、CommentActions 和分页控制器
- 迁入 38 个 emoji 与 40 个 stamp 资源并注册 10/5 列输入面板
- 新增评论页面、回复页、头像跳转、翻译、本人删除和安全失败态

### Git Commits

| Hash | Message |
|------|---------|
| `92c9e96` | (see git log) |

### Testing

- [OK] flutter analyze；flutter test 216/216；flutter build apk --debug；Trellis validate；MuMu 127.0.0.1:7555 读取评论成功

### Status

[OK] **Completed**

### Next Steps

- 启动 08-26-history-persistence，继续按执行序号推进


## Session 4: Complete browsing history persistence

**Date**: 2026-08-27
**Task**: Complete browsing history persistence
**Branch**: `main`

### Summary

完成 08-26-history-persistence：新增单连接 SQLite 历史库、紧凑记录、账号隔离 outbox、前台可见 Stopwatch 计时、历史页面与设置/作品/小说接入；补充迁移、事务、重试、损坏行和计时测试。flutter analyze、全量 flutter test（224）、debug APK、MuMu 页面与后台恢复检查均通过。

### Git Commits

| Hash | Message |
|------|---------|
| `7d04ee0` | (see git log) |

### Status

[OK] **Completed**
