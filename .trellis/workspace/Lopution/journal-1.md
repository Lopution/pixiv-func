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


## Session 15: 09-01 settings-productization: check, close gaps, archive
<!-- trellis-session: v=2 fp=8d02f3a7e59898d7 -->

**Date**: 2026-09-08
**Task**: 09-01 settings-productization: check, close gaps, archive
**Branch**: `task/09-01-settings-productization`

### Summary

Full-scope Grok check of the settings child (already on main via 6d720f2) passed with notes; closed the code-side notes, tightened three weak tests, recorded device-only items and the R3 quality-group deviation, archived.

### Main Changes

- Removed dead LoginWebViewInterceptController and networkWebViewIntercept*/networkInsecureNoSni* keys in all four locales; locale-parity test pins them absent
- Dropped unused imageSourceProvider (imageSource field kept hidden per R7)
- Widget feed loader refills up to 3 extra pages (same cap as feeds) before transient failure; R-18 baseline kept
- Tests: networkModeCode round-trip, legacy quality migration both halves, Accept-Language header + provider rebuild on languageTag, bounded refill loop semantics (refill ApiError is swallowed, first-page error surfaces)
- Spec: on-disk SQLite shared by parallel test files (database is locked) documented; sweep goes to spec-test-lint-hardening
## Session 16: child B release-size-per-abi：per-ABI 拆分、sqlite 排除、rhttp 裁剪、schema 2 updater、体积门禁
<!-- trellis-session: v=2 fp=56b602b3d727adff -->

**Date**: 2026-09-08
**Task**: child B release-size-per-abi：per-ABI 拆分、sqlite 排除、rhttp 裁剪、schema 2 updater、体积门禁
**Branch**: `task/09-07-release-size-per-abi`

### Summary

09-02 child B 完成：release 产物改为 arm64-v8a / armeabi-v7a 两个 split APK（obfuscate），arm64 27,435,321 B（fat 88.6 MB → −69%）；libsqlite3.so 排除；cargokit 陈旧 ABI 清理；rhttp 去掉 multipart/form/socks/cookies/charset 并收窄 tokio（query 保留，B5e 已回退）；updater manifest schema 2 多资产按 supportedAbis 选择；CI android-size 无 secrets 门禁 + B7 阈值；opt-level s 已测量未采用（待真机墙钟）。

### Main Changes

- B0 干净 ABI 基线（三 ABI fat 88.6 MB 中 85.9 MB 为 native）；B1 packaging.jniLibs.excludes 排除 libsqlite3.so（每 ABI −1.7 MB，historyDatabaseFactory 抽出可测）；B2 cargokit 构建前删陈旧 ABI 输出
- B3 ci.yml/release.yml 改 --split-per-abi + --obfuscate，双 APK 验签且证书一致，tool/apk_size_report.py（32 MB arm64 硬顶），新增无 secrets 的 android-size job（干净 runner 12m36s 通过）；B4 updater schema 2：顶层 packageName/cert，assets[] abi/url/size/sha256/versionCode，Kotlin supportedAbis，abi_unsupported 不得报已最新，versionCode % 1000
- B5 rhttp features：multipart/form/socks/cookies/charset 去掉、tokio 收窄，librhttp.so 5,412,048→5,042,000 / 3,636,460→3,340,216；B5e（query）回退——compat 适配层 io_request.dart 对每个请求传 query，app 测试在 Rust 边界之上 mock 未能发现；UPSTREAM.md 记「query 保留」
- B6 opt-level s 测得 −1,539,000 / −884,944，未采用未提交（采用门需用户真机墙钟；对照 APK 在 build/b6-compare/）；B7 阈值 28,435,321 / 24,226,795（vars 可覆盖）；B8 §6 收束；check 后补 android-release analyze-size 归档、release 重跑总是 upload --clobber、6 个 manifest 解析拒绝测试；spec backend/release-artifacts.md
## Session 17: 09-01 comment-translation：D2 事实复核、实名文案归位、封套负向测试、归档
<!-- trellis-session: v=2 fp=b0bb9e455c02bf8d -->

**Date**: 2026-09-08
**Task**: 09-01 comment-translation：D2 事实复核、实名文案归位、封套负向测试、归档
**Branch**: `task/09-01-comment-translation`

### Summary

评论翻译 child 收尾：复核 D2 三条外部事实（百度标准版/高级版额度、腾讯 TMT 在售、Google gtx 大陆阻断），实名要求只挂高级版（hint + picker 四语言），未使用的 translationProvider 回退改为 disabled，迁移封套逐层 key 集合与文本否定断言，百度错误码表补齐 90107/54000/54005，PRD 验收逐条附证据；大陆网络真实翻译等留给用户真机。

### Main Changes

- 研究：research/external-facts-2026-09-07.md 三条事实复核；D2 路径集合不变（关闭 / Google / 百度 / 通用 LLM），腾讯 provider 作为可选项留给用户决定
- i18n：translateBaiduHint 与 translateBaidu picker 四语言中实名认证只出现在高级版句子；BaiduCommentTranslationService 额度注释改为两档事实
- settings_controller.translationProvider 回退 google → disabled，与 translationSelectionProvider 一致
- 测试：迁移封套 version/payloadType/payload/checksum、accountId/userId/credential、accessToken/refreshToken 精确 key 集合 + 解码文本不含 baidu/llm/apikey；错误码表 8 条
- spec：backend/quality-guidelines.md 新增「Secrets stay out of every serialized surface」

### Git Commits

| Hash | Message |
|------|---------|
| `82e6a19` | chore(task): record settings-productization full-scope check |
| `ba6e637` | settings(network): remove dead login-intercept controller and leftover i18n keys |
| `e6ec619` | settings: drop unused imageSourceProvider |
| `dcf0f1b` | filter(widget): refill up to the shared page cap before giving up |
| `9c1b020` | test(settings): cover networkMode persistence, legacy quality migration, Accept-Language |
| `e5f017a` | test(feed): cover the bounded filter refill loop |
| `50f8e95` | test(settings): make migration, R6 provider watch and widget null-cursor cases discriminating |
| `ef15bb7` | chore(task): point refill coverage at the new tests |
| `cfbcfe0` | chore(task): tick settings-productization code-side acceptance criteria |

### Testing

- [OK] flutter analyze clean; full suite 641 tests, 4 loopback timeouts pass alone; fdroid/github release APKs 88.2 MB each (before child B)
| `fa9220c` | docs(size): record B0 clean ABI baseline |
| `4365c8a` | size(android): exclude unused sqlite3 native asset |
| `00f1df8` | build(rhttp): clean stale ABI outputs before cargo build |
| `8184471` | release: ship per-ABI APKs |
| `1967416` | updater: multi-asset manifest (schema 2) |
| `42bcc4b` | size(rhttp): drop reqwest multipart feature |
| `f78c278` | size(rhttp): drop reqwest form feature |
| `da669b0` | size(rhttp): drop reqwest socks feature |
| `6da1cec` | size(rhttp): drop reqwest cookies feature |
| `c463431` | size(rhttp): drop reqwest query feature |
| `280122d` | size(rhttp): drop reqwest charset feature |
| `c117953` | size(rhttp): narrow tokio features |
| `719b008` | Revert "size(rhttp): drop reqwest query feature" |
| `c737cb9` | ci: enforce per-ABI APK size budget |
| `e53d039` | docs(size): record per-ABI trim results and cache config |
| `29a4f44` | ci: archive analyze-size in android-release, always upload release assets |
| `7e740f6` | test(updater): cover manifest key, ABI, size and hex rejections |
| `61936d4` | spec(backend): record release artifact, size gate, updater and rhttp feature contracts |

### Testing

- [OK] flutter analyze 0；flutter test 635（两条 loopback 超时单独重跑通过）+ 新增 6 → 641；插件 32；cargo fmt/test --locked 2+1 ignored；Kotlin github/fdroid 单测通过；apk_size_report/update_release self-test 通过；fdroid split 27,435,321 / 23,226,795，github（debug 签）27,459,921 / 23,251,395，均在 B7 阈值内、每包仅一个 lib/<abi>、无 libsqlite3.so
| `ec92d20` | docs(translation): re-verify the three external facts behind D2 |
| `078458a` | i18n(translation): attach the real-name requirement to Baidu premium, not standard |
| `2860ba8` | fix(translation): keep real-name on the Baidu premium tier only |
| `08da6f0` | test(translation): pin transfer envelope keys and all Baidu error codes |
| `4135b11` | chore(task): tick comment-translation acceptance criteria with evidence |

### Testing

- [OK] flutter analyze 0；flutter test 全量 626（download_manager 一条环境超时单独重跑通过）；受影响四个测试文件 46 passed；git diff --check 干净

### Status

[OK] **Completed**

### Next Steps

- User device items: screenshots to research/screenshots, networkMode after restart, custom album + SAF, multi-P naming, ja Accept-Language capture, stage-5 timings
- User decision: keep three quality groups (preview/detail/view) or collapse detail+view per R3 wording
- 用户真机：API 29 与高版本各一次按 ABI 选资产的自更新（需 secrets + draft release）；--obfuscate 后 widgetBackgroundMain；B5 后登录/图片列表/大图下载与 ECH/TLS；B1 历史读写；B6 opt-3 vs opt-s 墙钟（build/b6-compare）
- android-release 持续红直到 PIXIV_RELEASE_KEYSTORE_B64 等 secrets 配置；可考虑把 android-size 加入 required checks
- F 迁 material_ui 后重测并重设 B7 阈值默认值
- 用户真机：大陆网络百度真实翻译、LLM 自定义 endpoint、Google 回归、关闭时零请求
- 用户决定是否新增腾讯 TMT provider（同样需实名，TC3 签名工作量更大）


## Session 18: 09-01 release-blockers: final check, contract tests, archive
<!-- trellis-session: v=2 fp=19ae8d18785f2e13 -->

**Date**: 2026-09-08
**Task**: 09-01 release-blockers: final check, contract tests, archive
**Branch**: `task/09-01-release-blockers`

### Summary

Verified the release signing / manifest signing / redirect allowlist implementation on main after child B; pinned the verifier contract in tests; archived. Real self-update on API 29 and modern devices plus the official certificate fingerprint stay with the user (need the real keystore + CI secrets).

### Main Changes

- Fail-closed github signing re-verified: no material -> verifyGithubReleaseSigning names the four properties, no APK; debug opt-in prints the warning and apksigner shows CN=Android Debug
- Both flavor split builds ok (fdroid 27,435,321 / 23,226,795; github debug-signed 27,459,921 / 23,251,395); update_release.py dry run produces schema 2 with exactly the parser's key sets
- Tests pin SHA256withECDSA + the five verifier codes and the measured release-assets.githubusercontent.com hop; root .gitignore covers jks/keystore/p12/pem; spec release-artifacts.md gains a Signing section

### Git Commits

| Hash | Message |
|------|---------|
| `98f3333` | chore(task): start release-blockers on its branch |
| `9d7260e` | docs(release): record GitHub release asset redirect chain |
| `c224d5e` | docs(release): point R5 at the sibling that implements it |
| `378913c` | test(updater): pin verifier algorithm, error codes and the measured CDN hop |
| `4939b23` | docs(release): tick verified items; leave device and secrets items to the user |
| `3a13304` | spec(backend): record the release signing and verifier contract |

### Testing

- [OK] flutter analyze clean; flutter test 658 passed (one oauth loopback timeout flake, rerun 16/16)
- [OK] Secret scan over branch log/diff, ls-files and tree: clean; release.yml leak-scan step present

### Status

[OK] **Completed**

### Next Steps

- User: configure keystore + manifest key secrets, dispatch release.yml, verify official fingerprint, run API 29 and modern-device self-update (positive + tampered)
- Next 09-01 children: reverse-image-saucenao and network-perf-ab checks, behavior-correctness-cleanup, then parent AC


## Session 19: 09-01 network-perf-ab: archive gate, doc sync, probe environment header
<!-- trellis-session: v=2 fp=851a3694497366e6 -->

**Date**: 2026-09-08
**Task**: 09-01 network-perf-ab: archive gate, doc sync, probe environment header
**Branch**: `task/09-01-network-perf-ab`

### Summary

Archive gate for the network route work already on main: analyze clean, network suites green, N-FIXED-1..9 re-verified. Synced comments, task docs and the frontend state-management spec to the shipped ladder (ECH first, insecureNoSni last, forced on, undelivered-only POST advance) and added an environment header to the copyable probe report. Two PRD criteria are marked superseded by the user's 2026-09-03 decision rather than ticked.

### Main Changes

- NetworkProbeEnvironment header (env/app-version/os/network-mode/doh/ech-front) in toCopyableText, assembled by the probe page
- state-management.md route contract rewritten to the final tier order with file references; design/implement notes mark the 09-03 PixEz-first draft as revised by the device probe

### Git Commits

| Hash | Message |
|------|---------|
| `ea6e884` | chore(task): start network-perf-ab on its branch |
| `5892a11` | feat(network): prepend measurement environment to copyable probe reports |
| `ad6eb7f` | docs(network): describe the shipped route ladder, not the 09-03 draft |
| `51098cf` | docs(network-perf): record acceptance verdicts, marking decision-superseded items |

### Testing

- [OK] flutter analyze clean; network_probe 18 + restricted_compat_network 27 + network_fast_route 4 + i18n_network_keys 4 passed

### Status

[OK] **Completed**

### Next Steps

- Parent 09-01 acceptance check once behavior-correctness-cleanup and reverse-image-saucenao are archived

## Session 20: 09-01 reverse-image-saucenao: anonymous policy re-verified, result-page and link fixes, archive
<!-- trellis-session: v=2 fp=b8265748acec6cbe -->

**Date**: 2026-09-08
**Task**: 09-01 reverse-image-saucenao: anonymous policy re-verified, result-page and link fixes, archive
**Branch**: `task/09-01-reverse-image-saucenao`

### Summary

Re-verified SauceNAO's anonymous HTML route with one synthetic probe (JSON API refuses anonymous callers), which exposed that every real result page was classified as a Cloudflare challenge; fixed that, routed SauceNAO's Pixiv link forms (member_illust.php / member.php / /en/ prefix) into the app, surfaced rate-limit wait time, daily limit and challenge as distinct outcomes, made no-match an empty success, kept every test offline and sanitized the fixture. Archived; device checklist stays with the user.

### Main Changes

- _classifyHtml matches only challenge markers; sanitized real result page fixture; challenge/dailyLimit codes; retryAfter rendered; no-match -> empty success
- IntentRouter accepts member_illust.php?mode=&illust_id=, member.php?id= and two-letter language prefixes; deny table pinned
- Page tests inject offline providers (previous test could POST to saucenao.com); success-path exactly-once cleanup test

### Git Commits

| Hash | Message |
|------|---------|
| `204f086` | fix(reverse-image): stop treating real SauceNAO result pages as challenges |
| `d4bcd3d` | research(reverse-image): re-verify SauceNAO anonymous policy and tick stage 0 |
| `88e0eb7` | fix(reverse-image): route SauceNAO's Pixiv links into the app |
| `59acfb4` | fix(reverse-image): show wait time, daily limit and challenge as distinct outcomes |
| `5ea1e51` | test(reverse-image): keep page tests offline and pin success-path cleanup |
| `f7bd03a` | test(reverse-image): finish sanitizing the SauceNAO result fixture |
| `4e5b6e4` | docs(reverse-image): tick verified acceptance items; device checks stay with the user |
| `7a2b716` | test(reverse-image): cover daily-limit copy and more deny shapes; sync design notes |
| `a61981f` | docs(reverse-image): record final full-suite result |

### Testing

- [OK] flutter analyze clean; full flutter test 672 passed; targeted reverse-image/intent tests 49

### Status

[OK] **Completed**

### Next Steps

- User device: Pixiv screenshot -> in-app detail, non-Pixiv -> browser, rate-limit wait copy, challenge copy, privacy notice
- Parent 09-01 acceptance once behavior-correctness-cleanup is archived

## Session 21: 09-01 behavior-correctness-cleanup: guard tests, spec truth, archive
<!-- trellis-session: v=2 fp=4c3004d94feb63e7 -->

**Date**: 2026-09-08
**Task**: 09-01 behavior-correctness-cleanup: guard tests, spec truth, archive
**Branch**: `task/09-01-behavior-correctness-cleanup`

### Summary

Closed the behaviour-correctness child on its own branch: 14 guard tests for the C2/C3/C4/C6/C7/C8/C21 fixes that had shipped without tests, state-management.md rewritten to the code on main, the guard-deletion rule added to both quality-guidelines, implement/PRD ticked; device matrix stays with the user.

### Main Changes

- New tests: opt-in auth replay is one refresh + one replay (second 401 surfaces); detail merge overwrites empty caption/tags, visible=false and smaller pageCount while feed merge never regresses them; closed widget gate starts nothing and a false->true recheck starts it; cold start builds only the current tab and visited tabs keep State; UserRoute pushes the user page, UnknownRoute only snackbars; removeAccount clears the remote outbox but not local history; legacy Pictures/PixivFunc destination migrates to builtin
- spec/frontend/state-management.md: EntityMergeSource, single opt-in replay, accountId + DownloadDestination.identity owner, fetchPageForContext, widget gate, lazy tabs, outbox clearing, exact-owner recovery
- Guard-deletion rule (delete the tests that froze the wrong behaviour in the same commit, no skip:) in frontend + backend quality-guidelines; widget_feed_loader comments name why credentialRevision stays
- Kept: feed merge ANDs visible (false sticks once seen) - deliberate, documented; not a C3 violation

### Git Commits

| Hash | Message |
|------|---------|
| `4d4424f` | chore(task): start behavior-correctness-cleanup on its branch |
| `511ad3c` | test(cleanup): guard the C2/C3/C4/C6/C7/C8/C21 behaviours |
| `82c8783` | spec(frontend): describe merge sources, replay opt-in, owner identity and lazy tabs as shipped |
| `6aeddbe` | docs(cleanup): attempt-first wording and current counts |
| `01b28c4` | docs(cleanup): tick verified items; device matrix stays with the user |

### Testing

- [OK] flutter analyze clean; flutter test 674 passed (660 before + 14 new); git diff --check clean
- [OK] Check agent (f4d319a3) found no blockers on HEAD; rg skip: in test only skip: false

### Status

[OK] **Completed**

### Next Steps

- User device: API 29 + modern, with/without widget, kill/restart recovery, bookmark/follow/comment token refresh, user deep link, multi-tab scroll
- 09-01 parent func-1-0-hardening: AC check and archive via PR once network-perf-ab (PR #7) and saucenao (PR #8) are merged


## Session 22: 09-01 parent func-1-0-hardening: acceptance on main, settings-consumer spec, archive
<!-- trellis-session: v=2 fp=0072e97ece0f7148 -->

**Date**: 2026-09-08
**Task**: 09-01 parent func-1-0-hardening: acceptance on main, settings-consumer spec, archive
**Branch**: `task/09-01-func-1-0-hardening`

### Summary

Closed the 09-01 parent: all seven children merged (PR #3-#9), parent-level acceptance verified against main HEAD 1186ff2 (children archived, R5 final owner, R6 greps zero, N-FIXED-1..9 intact with network suites 112/112), the one missing spec convention (settings must have a real consumer) written, and the journal union-merge hazard recorded in AGENTS.md. Device matrix stays with the user.

### Main Changes

- spec/frontend/quality-guidelines.md: 'A setting without a real consumer' forbidden pattern with the settings R7 gated-field exception
- research/parent-acceptance-2026-09-08.md: AC1-AC5 evidence; AC2 judged on final state (D5 and identity owner landed together in 6d720f2)
- AGENTS.md: after a rebase that touched .trellis/workspace, rebuild journal-1.md as main's copy + own block, re-point hashes, check session order (union merge interleaved sessions 19/20 today)

### Git Commits

| Hash | Message |
|------|---------|
| `90ab551` | spec(frontend): a setting without a real consumer is removed |
| `c4b1f60` | docs(09-01): parent acceptance evidence on main HEAD 1186ff2 |
| `a2ef1ad` | docs(agents): rebuild the journal after a rebase; union merge interleaves sessions |

### Testing

- [OK] Check agent (a692c601) network suites 112/112, flutter analyze clean; git diff --check clean

### Status

[OK] **Completed**

### Next Steps

- User: parent 最终真机验收 (API 29 + modern, mainland networks with/without proxy, OAuth, full feature pass, kill/restart recovery, updater install x2 with secrets) plus each child's device list in research/parent-acceptance-2026-09-08.md
- User decisions still open: settings R3 three vs two quality groups; Tencent translation provider
- 09-02: child D stages 3-4, child C plan from the recount, then F and E


## Session 23: Dart 架构收敛 C 阶段完成
<!-- trellis-session: v=2 fp=5a5ca1ae0631340a -->

**Date**: 2026-09-09
**Task**: Dart 架构收敛 C 阶段完成
**Branch**: `task/09-07-dart-architecture-convergence`

### Summary

完成 C0-C9：分层清零、组件与大文件拆分、core owner 收敛、ChangeNotifier 迁移、下载恢复上限、查询计划测试、颜色 token 化与局部声明私有化；同步 frontend spec。

### Main Changes

- 完成 C0-C9 全部实现与 implement.md 勾选，保留现有交互语义。
- 将跨 feature 的 FollowSwitchButton 归属 app/widgets，并清零 layering allowlist。

### Git Commits

| Hash | Message |
|------|---------|
| `9f8fb0a9a988193154ac59d325345aa4659125cd` | docs(frontend): record architecture and component contracts |
| `010de7a32c5d5ed632f6cf83f645712679aeb7eb` | refactor(core): move repositories and controllers under core |
| `dc3dce207919abd790f90591c8242d9831e3aef7` | refactor: consolidate colors and privatize local declarations |
| `37d8ad81cdc7712612201acd03492a4eb755d36c` | refactor: split large methods |
| `3c0f7513afa416c99cbf57ec9b0d8a6ce197dfaa` | refactor(network): split policy factory and image cache |
| `b66099220699a7d66d89d4851f7ec8226f8c8da0` | refactor(profile): split profile feed widgets |
| `cc4406c9737c122ae4c9d7251e547a5a30a0bcfb` | refactor(illust): split detail page widgets |
| `a310cc205f6c14b2fe614b23b94987b408fdf026` | refactor(settings): split settings pages into focused files |
| `772d2ea34229c8385e23ab1157e39da30c25b533` | test(history): verify indexed read query plans |
| `5bcd89541d59db93e44475f88743dfd59d07b1b7` | perf(download): cap DownloadRecoveryStore growth |
| `a36620d85c3e7d83a17d8ad2a583e1204fd3fecf` | refactor(novel): NovelReaderController becomes a plain view-model |
| `37e917df034fde3aa33cde3fbd938ae28affe143` | refactor(history): HistoryFeedController over the paged-feed base |
| `606ccc5fc042cf9dc3f78cdcb2a69d1254c824f7` | refactor(state): ReverseImageSearchController becomes a Riverpod Notifier |
| `ac8423084607403366b744aae8b99e302399e62b` | refactor(profile): ProfileEditController becomes a Riverpod Notifier |
| `6a9b75364891cfcc777b12761648e7c43a11f854` | docs(trellis): tick C6 in implement.md |
| `cc1714074e525efd570992ce54a98ec3297d9264` | refactor(i18): delete the ReplicaStrings table |
| `c451dcf05c2cc0efa90cc3d2b8f7752e94a57b1a` | refactor(i18n): migrate all call sites to context.l10n (gen-l10n) |
| `ad0ce3e25aae29a4adf4ee2cd71acc6ca702cbcb` | feat(i18n): generate AppLocalizations from ARB (gen-l10n) |
| `dfa0c10670978c22a8541fc7251ae49224b8c86d` | docs(trellis): tick C5e in implement.md |
| `b2e80133627172e63e7fa2260bb59bbab437cef2` | refactor(net): provider-owned HTTP clients for third-party traffic |
| `215cd5d88d7aac56fbd6bb2f5dea767101ee0397` | docs(trellis): tick C5d in implement.md |
| `0d5d0e2dc92f8322212bd6a3ff8a2c2a65c8b60d` | refactor(ui): single owner for SnackBar and debug logging |
| `1f20c311c01cdf75908034ce178058261f73c720` | docs(trellis): tick C5c in implement.md |
| `9ac0e6e38e041321631426be2709e3ff32cf5ebc` | refactor(entity): shared JSON field readers in core/entity/json_read.dart |
| `11ab06fd3876ce6b264967e5c343c550b1651239` | refactor(settings): single owner for SharedPreferences and keys |
| `b6f7314142237bafcab57723413a77c0888530c3` | refactor(net): derive hosts from PixivClientIdentity |
| `a9fa23fd2c313703e80beb34779c1b9e84872924` | docs(trellis): tick C3b-2/C3c-1/C4a/C4b in implement.md |
| `3effa47e94bc946a6b11284673ac949fd9ee1e2b` | refactor(app): move startup_gate into lib/app behind the routes facade |
| `d57a05fa360e5f70cc485ead136e79a9f71c2c20` | refactor(core): relocate remaining data-layer files into lib/core |
| `c68cd7f8725893ca60ffb6112426b19200f21b5d` | perf(image): size-aware decode policy for PixivImage variants |
| `b883a18b4a99c6e1bba4b98049f920d9bbdfbc6a` | docs(task): tick child C0-C3 in implement.md |
| `96fee882e26adaef6cc436b3df6634a84656fc65` | refactor(ui): MotionTokens single source; ReplicaPageRoute under app/motion |
| `22a63246b493c13ddf1b89d9b0f22667c5cc4f0a` | chore: ignore flutter_test golden failure artifacts (test/failures/) |
| `b109d28ac4e5fa266da86c5fad440c12ff774013` | refactor(ui): converge remaining feed states on FeedTail/FeedEmpty/FeedError |
| `c4c8dbfbd1d4a5b720eca136c2d5a6e09006ff71` | refactor(ui): shared FeedTail/FeedEmpty/FeedError in feed_states.dart |
| `1b466578ea95c159349c936dbddc5b697286d035` | refactor(ui): IllustCard, BookmarkSwitchButton, feed grid and hero motion under app |
| `1fe644586726706c25f1054d8625850a4bab93a6` | refactor(nav): routes.dart facade and shell metrics provider |
| `949e5dff23c5b587093c8e1c2cfa0c5204e9115f` | refactor(nav): merge navigation into lib/app/navigation, drop replicaRoute() |
| `d4664bbbaedc8fdd62b28f4e5a39fb93f1ce3655` | refactor: drop unused declarations and icon glyphs; enable unreachable_from_main |
| `b3636f50ba3d8dd9419d0a570cf1b2c42176aa77` | refactor: remove zero-reference barrels and tag search adapter |
| `3cd23f27568c5baa851fe4239b8d77e77ba968ac` | test(arch): layering rules with current violations allow-listed |
| `1830d1727627e5069715e545a4aae1d057a53526` | docs(task): child C planning — recount, design, implement derived from parent |

### Testing

- [OK] flutter analyze --no-pub：No issues found。
- [OK] flutter test -j 4 --no-pub：00:34 +690，All tests passed。
- [OK] flutter test test/architecture/layering_test.dart --no-pub：0 edges、0 data files、0 widget files。

### Status

[OK] **Completed**

### Next Steps

- 推送 task/09-07-dart-architecture-convergence，创建 PR，等待 CI 通过后以 merge commit 合并。
- 合并后由后续任务处理 D/E/F 与父任务。


## Session 24: Child D 原生层与 Rust 插件整理完成
<!-- trellis-session: v=2 fp=59ea654e55da0ed7 -->

**Date**: 2026-09-09
**Task**: Child D 原生层与 Rust 插件整理完成
**Branch**: `task/09-07-native-rust-hygiene`

### Summary

完成 Android channel 契约与线程整理、updater 去重、MediaStore/SAF 写块聚合、rhttp fork 文档与 backend spec；全量验证通过，API 29 真机四项回归留待用户设备。

### Main Changes

- 统一 Android channel 参数校验与错误码，移出四类同步 IO，并清理 API 29 下的死分支与模板残留。
- 抽取 updater 平台信息 helper，补充下载写块聚合测试与 rhttp 七步同步记录。
- 归档 D 任务并更新 Android channel、Rust plugin 与任务验证规范。

### Git Commits

| Hash | Message |
|------|---------|
| `3afecf5` | chore(task): start native-rust-hygiene with D0 recount and plan |
| `181147a` | chore(task): curate context and start native-rust-hygiene on its branch |
| `d1817b0` | docs(spec): record the Android channel contracts as shipped |
| `9d3dd8e` | android: uniform channel argument and error handling |
| `5eebef7` | docs(task): tick D1 and D2 |
| `7bc2f98` | android: move channel IO off the main thread |
| `e5230b9` | docs(task): tick D3 |
| `bb95212` | android: remove pre-API29 branches and template residue |
| `f881440` | android(updater): share platform info helpers |
| `47bb70c` | docs(task): tick D4 and D5 |
| `72e1f4f` | download: coalesce channel writes to 256 KiB |
| `86f2237` | docs(rhttp): complete fork record and seven-step sync |
| `3a3ff67` | docs(rhttp): add rust-plugin spec |
| `9459776` | docs(spec): channel contracts after D2-D5 |
| `334f464` | docs(task): record D final verification |
| `bca740b` | fix(rebase): preserve SAF picker interface |
| `3af1680` | docs(task): refresh D verification count |

### Testing

- [OK] flutter analyze --no-pub：No issues found。
- [OK] flutter test -j 4 --no-pub：692 passed。
- [OK] Kotlin 双 flavor、插件 Flutter、FRB、cargo fmt、cargo test --locked 与双 flavor split release 均通过。
- [OK] APK size/ABI 检查通过；github 本地构建仅使用显式 debug signing，apksigner v2 验证通过。

### Status

[OK] **Completed**

### Next Steps

- 推送 task/09-07-native-rust-hygiene，创建 PR，等待 CI 后以 merge commit 合并；用户设备补做 API 29 四项真机回归。


## Session 25: 完成 F8 体积校准与视觉现代化收尾
<!-- trellis-session: v=2 fp=d238d4f949f69eca -->

**Date**: 2026-09-10
**Task**: 完成 F8 体积校准与视觉现代化收尾
**Branch**: `task/09-07-interaction-visual-modernization`

### Summary

完成最终 fdroid split APK 复测并将 arm64-v8a/armeabi-v7a 门槛更新为 28,981,038/24,805,284 B；补全 README 与 frontend component guidelines，归档 F。flutter analyze 通过，全量测试 713 passed，size report 与 git diff --check 通过。排查出 WSL Meta TUN fake-IP 到 pub.dev 的 TLS 转发不回包，显式使用 Windows Clash 127.0.0.1:7897 后 pub get 与构建成功。

### Git Commits

| Hash | Message |
|------|---------|
| `6d178ae` | ci: recalibrate per-ABI APK budgets after UI modernization |
| `7c1fd8b` | docs: retire replica visual freeze |

### Status

[OK] **Completed**


## Session 26: 完成 E5 strict/async lint 并收尾 child E
<!-- trellis-session: v=2 fp=8091be2e9fa8e61b -->

**Date**: 2026-09-10
**Task**: 完成 E5 strict/async lint 并收尾 child E
**Branch**: `task/09-07-spec-test-lint-hardening`

### Summary

启用 analyzer strict-casts/strict-raw-types/strict-inference 与 unawaited_futures/avoid_dynamic_calls,按报告把 dynamic 响应参数改为 http.Response 等类型化写法并显式 unawaited;flutter analyze 0 问题,全量 716 test 通过(高于 E0 基线 713),dart format 与 git diff --check 干净。spec 的 quality/type-safety 已随 E1/E4 提交覆盖两批 lint 契约。前置:PR #13(F interaction-visual-modernization)合并后本分支 rebase 至 8cdb13a,重放 9 个提交内容不变。

### Git Commits

| Hash | Message |
|------|---------|
| `8374a88` | docs(test): recount spec and test hardening baseline |
| `bd27035` | docs(spec): replace templates with shipped contracts |
| `52ae7d0` | docs(core): document domain ownership |
| `3cabe5b` | test: share account and preferences fixtures |
| `8388807` | test: replace brittle type checks and remove diagnostic case |
| `2568f8f` | test(ui): cover shared component semantics |
| `5cf021d` | style: format Dart sources and enforce CI check |
| `3dd4a01` | lint: enable dependency order, quotes and final locals |
| `db696e4` | lint: enable strict types and async checks |

### Status

[OK] **Completed**


## Session 27: UI redesign: tokens, IA, shared components, responsive/a11y matrix
<!-- trellis-session: v=2 fp=2e22433137ea9904 -->

**Date**: 2026-09-11
**Task**: UI redesign: tokens, IA, shared components, responsive/a11y matrix
**Branch**: `task/09-11-ui-redesign`

### Summary

Semantic ThemeExtension layer, /me branch + root /settings, shared FeedLoading/TagChips/AuthorSummary, SliverLayoutBuilder grid columns, AppBreakpoints, golden matrix (10 files), reduced-motion page transitions. 741 tests green.

### Main Changes

- lib/app/theme/func_semantic_tokens.dart; lib/app/widgets/{tag_chips,author_summary,settings_action_button}.dart; lib/app/layout/app_breakpoints.dart; routes.dart /me branch + root settings; home_page NavigationBar/NavigationRail

### Git Commits

(No commits - planning session)

### Testing

- [OK] flutter analyze --no-pub: 0 issues; flutter test: 741 pass

### Status

[OK] **Completed**

### Next Steps

- PR review/merge; windows-desktop task deferred


## Session 28: windows-desktop: runner + 平台能力层 + 桌面登录/外链/CI
<!-- trellis-session: v=2 fp=dce4014246b2f6c0 -->

**Date**: 2026-09-11
**Task**: windows-desktop: runner + 平台能力层 + 桌面登录/外链/CI
**Branch**: `task/09-11-windows-desktop`

### Summary

Windows runner + capability 层 + InAppWebView 登录/SauceNAO + file_selector/url_launcher 降级 + CI windows job；762 测试通过；Windows 侧运行验收待真实设备

### Git Commits

| Hash | Message |
|------|---------|
| `84d120f` | docs(task): draft windows desktop task plan |
| `a783921` | docs(windows): record platform baseline and toolchain prerequisites |
| `71d582e` | feat(windows): add runner and desktop capability layer |
| `95860ef` | feat(windows): oauth login via in-app webview |
| `fe3b466` | feat(windows): saucenao result webview via inappwebview |
| `6f9cbc9` | fix(windows): track flutter metadata and drop unused test imports |
| `89d12ec` | fix(windows): gate double-back-to-exit to android |
| `5d4d0b4` | style(test): format windows capability test |
| `3fc1841` | fix(windows): route caps checks through injectable provider |
| `c4394f6` | feat(windows): url_launcher external links and webview2 missing prompt |
| `35bbac5` | ci(windows): release zip artifact on windows runner |
| `c276694` | test(windows): desktop verification evidence and remaining limits |
| `f9486ff` | refactor(windows): reuse outbound url capability for external results |

### Testing

- [OK] flutter analyze 0 issues; flutter test 762 pass; dart format lib test clean; git diff --check clean

### Status

[OK] **Completed**

### Next Steps

- 真实 Windows 机器跑 build/登录/下载验收清单后 archive


## Session 29: Android static performance audit
<!-- trellis-session: v=2 fp=1cd952e91d74bd53 -->

**Date**: 2026-09-13
**Task**: Android static performance audit
**Branch**: `task/09-13-android-static-performance-audit`

### Summary

完成 Android 滑动与 Hero 掉帧的静态审计：确认刷新率不是主因，定位 uncapped DPR decode width、800px masonry cache、Hero 动态裁剪与路由叠加为主要候选；对照 Pix-EzViewer、PixMix、Pansy、Pixview、Piko；analyze 和 13 个聚焦测试通过，全量测试保留 1 个既有 fade 契约失败，未做真机验证。

### Git Commits

| Hash | Message |
|------|---------|
| `ed91bee` | docs(perf): audit Android frame-drop paths |

### Status

[OK] **Completed**


## Session 30: UX 修复批次：详情页比例/个人页头部/后台缓存存活/底栏动效
<!-- trellis-session: v=2 fp=088a36d14937a6c2 -->

**Date**: 2026-09-15
**Task**: UX 修复批次：详情页比例/个人页头部/后台缓存存活/底栏动效
**Branch**: `task/09-15-startup-transition-i18n-fixes`

### Summary

修复四个视觉问题：详情页多图按固有比例自适应（meta_pages 无每页尺寸，改 PixEz 式固有尺寸+估算占位盒）；个人页 banner 缩矮至 320/350dp 并 Shaft 式整体暗化+白字；后台切回淡入重播——自定义 WidgetsFlutterBinding 跳过 imageCache.clear()（Android UI_HIDDEN 触发全清）；底栏切页落点重放真实点按的两层粉色 ink（InkHighlight+主题 splash，overlayColor.resolve(pressed) 取色）。另含早前累积的渐变参数对齐、caption 链接 TextSpan 化、搜索聚焦延迟、全局 Bouncing 物理、字号 token 收敛。

### Main Changes

- 详情页多图：去固定 AspectRatio，解码后按固有比例布局（PixEz 同款）
- 个人页头部：320/350dp 矮 banner + 黑色 scrim 降对比度 + 白色身份信息
- main.dart 自定义 binding：handleMemoryPressure 不再 imageCache.clear()，后台切回不重播淡入
- 底栏切页：目标 item 重放粉色 InkHighlight+InkSparkle（与顶栏逐项同源）
- 字号收敛 FuncSemanticTokens/textTheme；caption 链接内联化；搜索聚焦延迟至转场完成；全局 Bouncing 物理

### Git Commits

| Hash | Message |
|------|---------|
| `582a29e` | fix(illust): hero/ugoira transition guard, instant tier hand-off, PixEz fade timings |
| `b775628` | fix(illust): size detail pages by each image's intrinsic aspect ratio |
| `36938cb` | fix(illust): render caption links as inline text spans |
| `475fe2c` | refactor(ui): converge hard-coded font sizes onto semantic text tokens |
| `578b94c` | fix(search): focus the query field after the route transition completes |
| `caae01e` | feat(scroll): unified bounce physics and no platform overscroll glow |
| `b4c4198` | fix(nav): replay landing ink on the destination bar after a branch swap |
| `e166e2e` | feat(profile): shorter dimmed banner with white identity text |
| `8d76c5c` | fix(app): keep the decoded image cache across background trips |
| `f6e93c7` | docs(spec): image hand-off, intrinsic sizing and cache-survival contracts |

### Testing

- [OK] flutter analyze 0 issues；全量 783 测试通过；git diff --check 干净
- [OK] arm64 github release APK 构建通过（31.2MB）

### Status

[OK] **Completed**

### Next Steps

- 开 PR 合入 main（analyze-and-test CI）


## Session 31: 搜索筛选会员适配与刷新图片交接修复
<!-- trellis-session: v=2 fp=581b2555f86f28e8 -->

**Date**: 2026-09-16
**Task**: 搜索筛选会员适配与刷新图片交接修复
**Branch**: `task/09-16-search-filter-and-image-swap`

### Summary

非会员 popular_desc 改路由 popular-preview 端点，duration 预设客户端换算为日期区间并与自定义日期互斥；PixivImage 转 stateful，element 级 URL 交接置零淡化（Glide 语义），feed 卡片按 scope+作品 id 加 ValueKey

### Main Changes

- is_premium 从 OAuth user 解析，三登录流/refresh/transfer/JSON 全链路透传
- SearchFilters.toQuery 不再发 duration，预设换算 start_date/end_date；duration 与自定义日期互斥 + start>end 校验
- 非会员 popular_desc → /v1/search/popular-preview/* 端点（去 sort），Premium 保持原端点
- PixivImage stateful 化 + _lastShownUrl 坑位交接检测 → fade=0；卡片 ValueKey 跟随作品

### Git Commits

| Hash | Message |
|------|---------|
| `920185f` | feat(auth): persist user.is_premium through login and refresh |
| `1441c37` | fix(search): route free popular sort to popular-preview, resolve durations to dates |
| `d6dcef9` | fix(search): make duration and dates mutually exclusive in the filter sheet |
| `512fde4` | fix(image): replace cross-work dissolve with instant slot hand-off |
| `cdd1ada` | docs(spec): slot hand-off and premium search contracts |

### Testing

- [OK] flutter analyze 0 issue；全量 795 测试通过；CI 7/7 绿（PR #20）

### Status

[OK] **Completed**

### Next Steps

- 观察真机上 preview 端点翻页游标行为（next_url 参数回声未验证）


## Session 32: 小说域修复与功能闭环
<!-- trellis-session: v=2 fp=069d502a0d3108b1 -->

**Date**: 2026-09-16
**Task**: 小说域修复与功能闭环
**Branch**: `task/09-16-novel-domain-repair`

### Summary

小说正文改走 /webview/v2/novel 内嵌 bootstrap JSON（根因：detail 端点无正文字段）；补齐小说收藏 add/delete、评论 v3 端点泛化、九模式排行与 trending-tags/novel；父任务规划文档随本 PR 入库

### Main Changes

- fetchDetail = /v2/novel/detail 元数据 + /webview/v2/novel 内嵌 JSON 正文合并；extractNovelWebPayload 提取正文/上传图/pixivimage 缩略图/系列前后篇；缺脚本或截断对象显式报错
- BookmarkRepository 扩展 addNovel/deleteNovel（novel_id + restrict）；NovelStore 双向同步 BookmarkStore；详情页 BookmarkSwitchButton(isNovel)
- 评论域泛化 (workKind,workId)：/v3/novel/comments、/v2/novel/comment/replies、/v1/novel/comment/add；嵌套路由 + 详情页入口
- /v1/novel/ranking 九模式 feed + NovelRankingPage + 排行页入口；NovelRow 提取共享；trending-tags 按 illust/novel 分端点 + 搜索首页 SegmentedButton

### Git Commits

| Hash | Message |
|------|---------|
| `fea4b01` | docs(09-16): novel repair 根因证据与设计固化 |
| `d30c912` | fix(novel): 正文改走 /webview/v2/novel 内嵌 JSON |
| `17d1ca7` | feat(novel): 收藏 add/delete 与系列 prev/next 兜底 |
| `9549d69` | feat(novel): 评论支持 novel 维度 |
| `1cf35d7` | feat(novel): 小说排行与热词 |

### Testing

- [OK] flutter analyze 0 issue；dart format 干净；flutter test 全量 810 通过（含 novel_ranking_feed_test 5 条、webview 提取器 6 条、评论/收藏泛化回归）；git diff --check 干净

### Status

[OK] **Completed**

### Next Steps

- 开 PR 合入 main；内嵌图 token 的阅读器渲染属读者改版延期项（当前有效 token 折叠、无效显示原文，有测试覆盖）


## Session 33: motion-layer：动效语法统一 + reduce-motion + 四层接入
<!-- trellis-session: v=2 fp=78d1a430ee3a5459 -->

**Date**: 2026-09-16
**Task**: motion-layer：动效语法统一 + reduce-motion + 四层接入
**Branch**: `task/09-16-motion-layer`

### Summary

MotionTokens 单一数据源+MotionScope 双源降级；路由转场三原语；StaggeredEntrance+PressScale 挂全量卡片；sheet/dialog 单入口收口字面量

### Main Changes

- lib/app/motion/*, lib/app/navigation/routes.dart, feed/novel 卡片与列表, 设置页

### Git Commits

| Hash | Message |
|------|---------|
| `6fe3a60` | docs(09-16): motion-layer 设计固化 |
| `20972e4` | feat(motion): MotionTokens 扩展 + 减少动态效果设置 |
| `97a2709` | feat(motion): 路由转场语义化 |
| `30ff34b` | feat(motion): 列表进场 stagger + 卡片按压反馈 |
| `820d922` | feat(motion): 弹层时长统一 + 动效字面量收口 |
| `a2eda8a` | test(motion): 降级矩阵与进场/按压断言 |

### Testing

- [OK] flutter analyze 0 issue; flutter test 821 全绿; motion_test 11 例

### Status

[OK] **Completed**

### Next Steps

- rebase main → PR → 进入 Wave 2 card-quick-actions


## Session 34: card-quick-actions：卡片长按快捷菜单与稍后再看
<!-- trellis-session: v=2 fp=397f5c084a15472b -->

**Date**: 2026-09-16
**Task**: card-quick-actions：卡片长按快捷菜单与稍后再看
**Branch**: `task/09-16-card-quick-actions`

### Summary

IllustCard 长按弹动作 sheet（收藏/下载/稍后再看/分享，注册表可扩展），watchlater.db 账号隔离落库 + 列表页/路由/设置入口

### Main Changes

- lib/app/widgets/card_actions/*（CardAction 抽象 + 有序注册表 + showAppBottomSheet 弹层）；lib/core/watchlater/*（watchlater.db、repository 幂等 add/rowid 次序、AsyncNotifier store）；lib/features/watchlater/watchlater_page.dart；IllustEntity.toJson 往返；routes.dart 'watchlater' common route + openWatchLater；settings_page tile；四语言 cardAction*/watchLater* 键

### Git Commits

| Hash | Message |
|------|---------|
| `927583e` | docs(09-16): card-quick-actions 设计固化 |
| `d26a3a8` | feat(watchlater): core 域落库与 store |
| `407dc46` | feat(card): 动作注册表与长按菜单 |
| `3c4e2cb` | feat(watchlater): 列表页与入口 |
| `7ba69fc` | test(card): 手势/分发/暂存覆盖 |

### Testing

- [OK] flutter analyze 0 issue；flutter test 833 全绿（新增 card_action_test 6 项 + watch_later_store_test 6 项）；dart format 干净；git diff --check 干净

### Status

[OK] **Completed**

### Next Steps

- Wave 2 继续 mute-system（屏蔽动作注册位已留在 illustCardActionsProvider 注释）


## Session 35: mute-system：三态屏蔽与服务端同步
<!-- trellis-session: v=2 fp=dadb0830c51abe56 -->

**Date**: 2026-09-16
**Task**: mute-system：三态屏蔽与服务端同步
**Branch**: `task/09-16-mute-system`

### Summary

tag/用户屏蔽走官方 /v1/mute 同步，单作品屏蔽本地按账号持久化；卡片模糊+点按揭示，设置可切直接隐藏；/settings/muted 三段管理页；长按菜单新增屏蔽此作品/作者

### Main Changes

- lib/core/mute/*(MuteStore 账号边界+hydrate+legacy blocked_tags 迁移推送+pending 回滚；MuteRepository /v1/mute/list+edit)；mute_predicate；IllustCard+MutedCover+revealedMuteIdsProvider；PagedFeedController/WidgetFeedLoader hide 模式过滤；MutedItemsPage+/settings/muted 路由+设置 tile；_MuteWorkAction/_MuteUserAction 注册进 illustCardActionsProvider；info_block tag 屏蔽改走 MuteStore；blocked_tags/BlockedTagsPage 删除；四语言 mute* 键

### Git Commits

| Hash | Message |
|------|---------|
| `b139c3c` | docs(09-16): mute-system 设计固化 |
| `fcd884f` | chore(09-16): mute-system 上下文清单与激活 |
| `e22fb92` | feat(mute): core 域与服务端同步 |
| `df7e8ba` | feat(mute): 卡片模糊变体与 feed 过滤 |
| `60d5ae8` | feat(mute): 长按动作与管理页 |
| `59976bc` | test(mute): 端到端覆盖 |

### Testing

- [OK] flutter analyze 0 issue；相关测试全绿（card_action 10 项、mute_store 9 项、paged_feed hide/blur 2 项）；全套件 1 项失败已修（ranking fixture 未接 /v1/mute/list）

### Status

[OK] **Completed**

### Next Steps

- Wave 2 末项 search-filter-v2


## Session 36: search-filter-v2：Shaft v3 级过滤器面
<!-- trellis-session: v=2 fp=d0bd7ad809adb1db -->

**Date**: 2026-09-16
**Task**: search-filter-v2：Shaft v3 级过滤器面
**Branch**: `task/09-16-search-filter-v2`

### Summary

SearchFilters 补齐 Shaft v3 面:男/女向人气 sort(会员)、AI 三态(仅 AI 客户端谓词)、bookmark_num_min/max(会员 wire+客户端兜底)、ratio_pattern、content_type、width/height_min/max;sheet 按 SearchResultType 分组;novel sort 归一

### Main Changes

- search_models 新枚举+字段+cacheKey/toQuery(illust-only 范围);next_page_parser 白名单;search_repository popular 三档非会员 preview;_SearchFeedController.filterPageIds 客户端谓词;search_filter_sheet type 参数+五组新控件;四语言 23 键

### Git Commits

| Hash | Message |
|------|---------|
| `2d2bfbf` | docs(09-16): search-filter-v2 设计固化 |
| `c7e6243` | feat(search): 过滤模型与 wire 契约 |
| `8769b9e` | feat(search): 客户端兜底谓词与会员路由 |
| `460d617` | feat(search): 过滤 sheet 分组 |

### Testing

- [OK] flutter analyze 0 issue;flutter test 857 全绿(新增 wire 序列化/cursor 白名单/谓词/sheet 渲染断言);dart format 干净

### Status

[OK] **Completed**

### Next Steps

- Wave 3:bookmark-tags / content-expansion / network-account-settings / reverse-search-engines


## Session 37: bookmark-tags: 收藏标签体系
<!-- trellis-session: v=2 fp=91c642ce71441d5d -->

**Date**: 2026-09-16
**Task**: bookmark-tags: 收藏标签体系
**Branch**: `task/09-16-bookmark-tags`

### Summary

收藏标签维度闭环:add 携带 tags[](空格拼接,PixEz 形)、bookmark detail 预填编辑 sheet、用户标签集合分页管理页、tag 过滤收藏 feed

### Main Changes

- core/bookmark: BookmarkTagFacet/UserBookmarkTag/BookmarkDetail/UserBookmarkTagPage 模型;op/entry 携带 tags;addWithRestrict 透传;repository 增 fetchDetail/fetchUserTags;NextPageParser 白名单 bookmark-tags 两端点 + bookmarks tag 参数;收藏 sheet 重构为 _BookmarkEditSheet(已收藏长按开编辑,detail 预填,已选 InputChip+建议 FilterChip+新标签输入);userBookmarkTagSuggestionsProvider/bookmarkDetailProvider;UserBookmarkTagsController 轻量分页;BookmarkTagsPage + BookmarkTagFeedPage(ProfileFeedKey.bookmarkTag);路由 bookmarks/tags + bookmarks/tag;profile 收藏 tab 工具栏标签入口;四语言 l10n

### Git Commits

| Hash | Message |
|------|---------|
| `41d1b4e` | docs(09-16): bookmark-tags 设计固化 |
| `708962a` | feat(bookmark): 标签模型与 wire 契约 |
| `a80e421` | feat(bookmark): op/entry 携带标签 |
| `73bacdf` | feat(bookmark): 收藏 sheet 标签编辑 |
| `44d182e` | feat(bookmark): 标签页与过滤收藏 |

### Testing

- [OK] flutter analyze 0 issue;872 测试全绿(含 layering_test 修复跨 feature import:BookmarkTagFeedPage 移入 features/profile);git diff --check 干净

### Status

[OK] **Completed**

### Next Steps

- Wave 3 继续:content-expansion(series+pixivision)


## Session 38: content-expansion: 插画系列与 pixivision
<!-- trellis-session: v=2 fp=cd96fc7fc56f008b -->

**Date**: 2026-09-17
**Task**: content-expansion: 插画系列与 pixivision
**Branch**: `task/09-16-content-expansion`

### Summary

插画系列(浏览/上下文导航/详情入口/用户系列 tab)与 pixivision spotlight(列表/应用内富文本文章页/搜索入口)落地;cursor 语义校验与 commit-gate 复用既有分页契约

### Main Changes

- core/series: IllustSeriesEntity/Context + store + repository(3 端点,cursor 钉住 seriesId/userId,/v1/illust-series/illust 404=非系列) + 两个 feed/context controller;features: IllustSeriesPage + UserSeriesFeed + 详情页 IllustSeriesSection(prev/next 导航) + profile 第4 chip(ProfileWorkSection 解耦 wire UserWorkType);路由 series/:id
- core/spotlight: SpotlightArticle + store + repository(文章 HTML 走 thirdPartyHttpClientProvider,桌面 UA+pixivision Referer+Accept-Language)+ category family feed;article_parser(am__body/_feature 变体→段落含链接/标题/图/.illust 卡);SpotlightArticlePage(blocks 渲染,pximg→PixivImage,其余→CachedNetworkImage,/artworks//users/ 站内跳转,其余 url_launcher);SpotlightFeedPage(SegmentedButton 三类)+ search 页入口行;路由 spotlight + spotlight/article/:id?url=;四语言 l10n 10+ 键
- 质量:修复 gen_l10n_lookup.py 过期 import(replica_strings→replica_language);layering R5 改名 _SeriesCardView/_UserSeriesCardView;spec 补两条测试坑(autoDispose .future 需持订阅;http.Response 默认 latin-1)

### Git Commits

| Hash | Message |
|------|---------|
| `729676c` | docs(09-16): content-expansion 设计固化 |
| `bfbced3` | feat(series): 系列模型与 wire 契约 |
| `66ae8ed` | feat(series): 系列 feed 与系列页 |
| `f54d098` | feat(series): 详情页与用户页入口 |
| `160fca5` | feat(spotlight): 文章模型与列表 feed |
| `bc4347d` | feat(spotlight): 文章解析与应用内详情 |
| `6c1691a` | feat(spotlight): 列表页与搜索页入口 |

### Testing

- [OK] flutter analyze 0 issue;flutter test 912 全绿(series repository/store/feed/page、详情系列区块、profile chip、spotlight repository/feed/parser/page、搜索入口);dart format 干净;git diff --check 干净

### Status

[OK] **Completed**

### Next Steps

- 父任务 09-16-client-parity-upgrade 剩余子任务:feed-resilience / watchlist-local-library / tablet-desktop-layout 等


## Session 39: 09-16-network-account-settings: 镜像 + 服务端显示设置 + 备份
<!-- trellis-session: v=2 fp=b1a612c77ccc64f2 -->

**Date**: 2026-09-17
**Task**: 09-16-network-account-settings: 镜像 + 服务端显示设置 + 备份
**Branch**: `task/09-16-network-account-settings`

### Summary

镜像预设/自定义、服务端显示设置同步、JSON 备份导出导入

### Main Changes

## 摘要

完成「网络与账号设置增强」全部 6 个实施勾：图片镜像（官方 + pixiv.cat/re/nl 预设 + 自定义 HTTPS 反代，经 `PixivDestinationRegistry` 白名单与 `PixivPolicyHttpClient` URL 重写贯穿图片缓存与下载通道）、服务端显示设置（`/v1/user/ai-show-settings` 与 `/v1/user/restricted-mode-settings` 读写，乐观 toggle + 失败回滚 + 可见错误）、JSON 备份（`pixivfunc.backup.v1` envelope，settings/mutes/history，merge/overwrite 策略，SAF 导出 + file_selector 导入，凭据经 `SecretSettingRef` 不出仓）。

## 关键决策

- 镜像 host 走 destination registry 的 `extraImageHosts` 白名单，而非放开 host 校验；第三方镜像只走 `dohRealSni` 路由（ECH/noSni/insecureNoSni 是 Pixiv 自有设施专用）。
- 自定义镜像持久化值必须是 `https://` 规范形式，UI 侧 `normalizeCustomSource` 升级裸 host；拒绝非 443 端口与 IP 字面量，与 registry 约束对齐。
- 导入 merge/overwrite 的明确语义：settings 覆盖、服务端 mute tags/users 永远只增（与官方客户端共享状态，不删）、本地 workIds union 或替换、history 按账号+类型+ID 取新 `lastViewedAt`。
- 导出走既有 SAF 抽象（Android method channel / 桌面文件系统），写失败删除半成品文档。

## 测试

- 全量 `flutter test`：962 通过、0 失败（含 `test/architecture/layering_test.dart`）。
- `flutter analyze` 0 issue；`dart format --set-exit-if-changed lib test` 0 改动；`git diff --check` 干净。
- 新增覆盖：`image_mirror_test`、`restricted_compat_network_test`（镜像集成）、`server_display_settings_test`（8 例）、`backup_test`（20 例）、`settings_test`（镜像 UI）。

## 坑位（已写入 spec）

- `material_ui` 的 `SwitchListTile`/`SnackBar`/`ScaffoldMessenger` 遮蔽 Flutter 同名类：测试必须用 `material_ui` 的 `MaterialApp`，否则 snackbar 被静默丢弃。
- Riverpod 3 对失败的 `AsyncNotifierProvider.build` 无限退避重试：`provider.future` 永不到终态，错误断言要走 `AsyncError` 状态监听。


### Git Commits

| Hash | Message |
|------|---------|
| `f283526` | docs(09-16): network-account-settings 计划与上下文 |
| `c52d571` | feat(mirror): 镜像源模型与 URL 重写 |
| `961e03d` | feat(mirror): 图片通道接入与探针 |
| `8ed7920` | feat(mirror): 镜像选择 UI |
| `2dfcf1d` | feat(account): 服务端显示设置读写 |
| `5d42bc5` | feat(backup): 导出导入域与 JSON schema |
| `735d467` | feat(backup): 导出导入 UI |

### Status

[OK] **Completed**


## Session 40: 09-16-reverse-search-engines: 多引擎以图搜图（IQDB/Ascii2D/TinEye + WebView 自上传）
<!-- trellis-session: v=2 fp=05f930f09095bc82 -->

**Date**: 2026-09-17
**Task**: 09-16-reverse-search-engines: 多引擎以图搜图（IQDB/Ascii2D/TinEye + WebView 自上传）
**Branch**: `task/09-16-reverse-search-engines`

### Summary

Session summary was not supplied.

### Main Changes

为 SauceNAO 反向搜图接入 IQDB / Ascii2D / TinEye，四引擎可选可切。

**传输决策**：Ascii2D/TinEye 在 Cloudflare 后面，headless multipart 会被 cf-mitigated 质询拦（Shaft #733 实证）。走 WebView 自上传——真实页面手势 + 真浏览器 Cookie/JS 上下文过质询，不注入 input.files、不自动 click、不退回 headless。Android 侧 `armReverseUpload` channel 校验 `cacheDir/reverse_image_inputs/` 归属后 FileProvider 出 content Uri，vendored InAppWebViewChromeClient 的 armed 槽一次性消费；桌面无选择器拦截，提示用户在页面选择器里重选同图。IQDB 留在 headless（无 CF），引擎级约束 jpeg/png/gif ≤8MiB ≤7500px 在发请求前拒绝。

**结构**：`ReverseImageEngineSpec`（displayName/transport/端点/host 白名单/约束）+ `ReverseImageSearchWebUpload` outcome；导航策略按引擎 host 参数化；`selectEngine` 无输入也可切换（持久化选择），失败保留输入换引擎重试；`AppSettings.reverseImageEngine` 持久化。

**测试教训**：`InAppWebView` 在 widget 测试需 `InAppWebViewPlatform.instance` fake（断言 null platform）；平台类型经 `flutter_inappwebview` 主包 re-export，直引 platform_interface 触发 depend_on_referenced_packages。已写入 frontend/quality-guidelines。


### Git Commits

| Hash | Message |
|------|---------|
| `50d6deb` | docs(09-16): reverse-search-engines 计划与上下文 |
| `9974116` | feat(reverse-image): 引擎模型与 IQDB provider |
| `9c0ad4d` | feat(reverse-image): WebView 上传通道 |
| `d95b0d6` | feat(reverse-image): 引擎切换与导航策略泛化 |
| `6483d65` | feat(reverse-image): 引擎选择与上传 UI |

### Testing

- [OK] flutter analyze 0 issue；flutter test 全量 990 通过；:app:testGithubDebugUnitTest BUILD SUCCESSFUL；dart format 0 改动；git diff --check 干净

### Status

[OK] **Completed**
