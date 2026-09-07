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


## Session 5: Restricted Pixiv compatibility network implementation

**Date**: 2026-08-28
**Task**: Restricted Pixiv compatibility network implementation
**Branch**: `main`

### Summary

Implemented exact-host direct-first policy, strict secure-DNS candidate path, shared API/OAuth/image/download integration, WebView fail-closed gate, and API35 MuMu evidence; API36 remains blocker.

### Git Commits

| Hash | Message |
|------|---------|
| `97ee84e` | (see git log) |

### Status

[OK] **Completed**


## Session 6: Android platform boundary hardening

**Date**: 2026-08-28
**Task**: Android platform boundary hardening
**Branch**: `main`

### Summary

Implemented strict Android intent and shared-image validation, versioned WebView capability and route sessions with exact-host/revision/owner fencing, PKCE state and login lifecycle handling, and lifecycle-aware double-back behavior. Focused and all 37 split serial test files passed; flutter analyze and debug APK build passed. MuMu emulator API 35 validated with real logged-in account, WebView lifecycle and root back; API 36 and physical/carrier coverage remain explicit blockers. Archived 08-27-android-platform-boundary-hardening.

### Git Commits

| Hash | Message |
|------|---------|
| `ff4dd2e` | (see git log) |

### Status

[OK] **Completed**


## Session 7: Profile edit boundary and API35 MuMu verification

**Date**: 2026-08-28
**Task**: Profile edit boundary and API35 MuMu verification
**Branch**: `main`

### Summary

Implemented typed ProfileEditController, bounded image/text validation, owner fencing, confirmed-only AccountStore/UserStore commit, reachable profile edit UI and explicit unavailable submit route. Verified API35 MuMu 127.0.0.1:16384 with proxy null, non-VPN validated Wi-Fi, real signed-in read-only profile flow and disabled Save. Focused profile/settings tests passed; full suite 321 passed with existing icon_font MissingPluginException. Archived profile-edit; API36 and live mutation remain explicit blockers.

### Git Commits

| Hash | Message |
|------|---------|
| `cd3435e` | (see git log) |

### Status

[OK] **Completed**


## Session 8: Live player feasibility gate closed as real blocker

**Date**: 2026-08-28
**Task**: Live player feasibility gate closed as real blocker
**Branch**: `main`

### Summary

Revalidated live endpoints on 2026-08-28 via in-app probe through the real PixivHttpClient/policy stack on MuMu API35 (127.0.0.1:16384, proxy null, Wi-Fi VALIDATED NOT_VPN): list/list?for_android/list?for_ios all HTTP 200 with lives=0, /v1/live/detail?live_id=0 HTTP 404, sketch.pixiv.net refused by PixivDestinationRegistry before I/O, web /lives redirects to closed 410. Gate failed (no live object), so no player dependency, no live UI, no fixture path; sanitized evidence + restored-APK screenshot stored in archived research. Fixed pre-existing icon_font golden failure (HomePage EventChannel subscription) via stub AndroidIntentSource; full suite 348 passed, analyze clean, debug build ok.

### Git Commits

| Hash | Message |
|------|---------|
| `e47d680` | (see git log) |

### Status

[OK] **Completed**


## Session 9: 08-29-replica-v1-completion 实施：DoH 接线、SNI spike 重大发现、R7 六项修复

**Date**: 2026-08-29
**Task**: 08-29-replica-v1-completion 实施：DoH 接线、SNI spike 重大发现、R7 六项修复
**Branch**: `main`

### Summary

完成 08-29-replica-v1-completion 主体实施：SNI spike 三项平台行为钉死（connectionFactory 明文泄漏重大发现→secureDns 阶梯补 SecureSocket.secure 包装）；tlsHandshake 可回退；DNS wire codec + DohResolver 默认接线 + 路由记忆 + 双出口共用 ladder；分层探测页 + 网络设置 + 四语言 i18n；minSdk=29 + 剪贴板 capability <33 警告；U1-U6 全部修复含同型排查（copyWith 唯一漏字段 createDate；feed 页无快照场景不为同型）。新增测试 87+ 全过，analyze clean，debug APK 已装 RMX5200（transport 22，minSdk 29 生效）。全量测试仅 environment-flaky 用例偶发（WSL loopback）。

### Git Commits

(No commits - planning session)

### Status

[OK] **Completed**

### Next Steps

- 用户真机验证 U1-U6 + 大陆分层探测报告回填；A1-A5 残余阻塞待用户


## Session 10: 修复 Pixiv 默认 ECH 网络路径

**Date**: 2026-08-31
**Task**: 修复 Pixiv 默认 ECH 网络路径
**Branch**: `main`

### Summary

完成 rhttp ECH 传输、Dart 路由阶梯、DoH HTTPS RR 解析、下载与 WebView 登录拦截的质量修复；移除依赖本机 /tmp/ech65.out 的不可复现测试入口，补充仓库 fixture 约束。flutter analyze、串行全量 flutter test、Rust cargo check/test、Android github debug APK 构建、任务校验和 git diff --check 均通过；尚未进行物理真机或真实境内网络验收。

### Git Commits

| Hash | Message |
|------|---------|
| `9c8a44a` | (see git log) |

### Status

[OK] **Completed**


## Session 11: 修复移动端 UI 动画与个人页问题

**Date**: 2026-08-31
**Task**: 修复移动端 UI 动画与个人页问题
**Branch**: `main`

### Summary

按作用域隔离作品 Hero 并传入列表快照，使详情首帧同步渲染；简介默认显示；彻底移除个人页设置入口；补充回归测试与 frontend spec。相关分析、聚焦测试和 github debug APK 通过；全量测试有两个既有 loopback 超时，当前无连接设备未做设备复测。

### Git Commits

| Hash | Message |
|------|---------|
| `39181c2` | (see git log) |

### Status

[OK] **Completed**


## Session 12: Pixiv 网络与作品转场修复

**Date**: 2026-09-01
**Task**: Pixiv 网络与作品转场修复
**Branch**: `main`

### Summary

完成无代理网络路径、ECH 缓存与探测结论优化；修复首图/头像预热、Hero 作用域与视口裁剪、反向下拉刷新；analyze、聚焦测试 41 项、全量测试 522 项和 github debug APK 构建均通过。

### Git Commits

| Hash | Message |
|------|---------|
| `2eaf5aa` | (see git log) |

### Status

[OK] **Completed**


## Session 13: 修复刷新与用户页 UX 正确性

**Date**: 2026-09-02
**Task**: 修复刷新与用户页 UX 正确性
**Branch**: `main`

### Summary

按 PixEz 参考方案将共享下拉刷新收敛到 easy_refresh；NestedScrollView tab 使用 locator Header；个人主页头像、身份区与 pinned toolbar 分层并修复滚动冲突；同步 spec 与回归测试。flutter analyze 无问题，flutter test +541 全部通过；保留用户真机观感验证项。

### Git Commits

| Hash | Message |
|------|---------|
| `10ea29d` | (see git log) |

### Status

[OK] **Completed**


## Session 14: Child A：依赖与工具链健康（A0–A10 全部落地）
<!-- trellis-session: v=2 fp=c04418dc80504a67 -->

**Date**: 2026-09-07
**Task**: Child A：依赖与工具链健康（A0–A10 全部落地）
**Branch**: `task/09-07-dependency-toolchain-health`

### Summary

09-02 child A 在 task/09-07-dependency-toolchain-health 分支上按 implement.md 逐 checkbox 提交完成：删 cupertino_icons、Flutter 3.47.2、pub/Cargo lock 刷新（121 crate）、work-runtime、CI actions v6/v5、archive 4/image 4.9、flutter_secure_storage 11 + AGP 9.1.1 + compileSdk 37 + androidx.core 1.19、FRB 三元组精确 pin + tool/frb_check.sh、仓库根 rust-toolchain.toml 1.98.0 + --enforce-lockfile/--locked、cargo fmt 门禁（生成文件 rustfmt::skip）、CI 新增 plugin/android-unit/deps-report。全量检查 PASS WITH NOTES；真机验证留给用户。

### Main Changes

- cupertino_icons 移除；Flutter 3.47.0→3.47.2（本地 /opt/flutter-3.47.2 + ci.yml/release.yml）；pubspec.lock/Cargo.lock 刷新并记录新 crate 许可证
- androidx.work ktx→work-runtime 2.11.2；archive ^4.2.0 带动 image 4.9.2（Inflate 命名参数、ZipEncoder.encode 非空两处源码改动）
- flutter_secure_storage 11 + AGP 9.1.1 + compileSdk 37 字面量 + androidx.core 1.19.0；platforms;android-37.0 命名导致 cargokit plugin.gradle 解析修补（UPSTREAM.md D-5）
- flutter_rust_bridge 三元组收紧到精确 2.12.0（两处 lock 由 2.13.0 回到 2.12.0，插件 lock 改为跟踪），tool/frb_check.sh 进 CI
- 仓库根 rust-toolchain.toml（cargokit 只读 app 根目录，rustup 向上查找同一文件；.rustc_info.json 已证实 APK 构建用 1.98.0）；CI --enforce-lockfile；plugin/android-unit/deps-report 三个 job
- spec：frontend/quality-guidelines.md 新增 Dependency and toolchain upgrade policy（跟随 Flutter 验证矩阵、Kotlin 停在 2.4.0 的理由、lock 即契约、生成代码免 fmt 门禁）

### Git Commits

| Hash | Message |
|------|---------|
| `5212efb` | chore(task): start dependency-toolchain-health on its branch |
| `289faa5` | research(09-02): re-verify dependency and APK baselines at child A start |
| `68bcae3` | deps: drop unused cupertino_icons |
| `fe0643e` | build: flutter 3.47.2 |
| `9b071fb` | deps: refresh pubspec.lock and Cargo.lock |
| `67644db` | ci: bump actions |
| `2d09f8c` | build(android): use work-runtime instead of the empty ktx artifact |
| `20811c0` | deps: archive 4 / image 4.9 |
| `8071f49` | deps: flutter_secure_storage 11 |
| `ff65707` | build(rhttp): pin flutter_rust_bridge triplet and add check |
| `170b49f` | build: pin rust toolchain and enforce lockfiles |
| `abccfac` | style(rhttp): apply cargo fmt to the ECH additions |
| `718f932` | ci: run kotlin, plugin and rust tests |
| `da55454` | docs(spec): record the dependency and toolchain upgrade policy |

### Testing

- [OK] flutter analyze 0；flutter test 626；插件 flutter test 32；cargo fmt --check 干净、cargo test --locked 2 passed/1 ignored；Kotlin 13+13；fdroid release 88,181,654 B、github（debug 签名覆盖）88,206,254 B，compileSdk 37/minSdk 29/targetSdk 36，cupertino_icons 0 项
- [OK] flutter pub outdated 直接依赖仅剩 go_router 17.5.0 / cached_network_image 3.4.1（留给 child F）；cargo update --dry-run 0 packages；frb_check.sh 从根目录与 /tmp 均 OK

### Status

[OK] **Completed**

### Next Steps

- 用户真机验证：Ugoira 播放 + GIF 导出（archive/image 升级后）；登录凭据与翻译凭据读写（secure_storage 11 后）
- 用户配置 GitHub secrets PIXIV_RELEASE_KEYSTORE_B64 等四项，否则 android-release job 持续红（非 ruleset 必需项，不阻塞合并）
- 首次 CI 绿后考虑把 plugin / android-unit 加入 protect-main ruleset 的 required checks
- child B（release-size-per-abi）以本分支合并后的 main 为基线重跑 B0
