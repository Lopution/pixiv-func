# 执行计划：设置页产品化与真实消费者接线

## 开工条件

- [x] 用户已确认本 child 的最终 UI 选择：相册/文件夹单一入口、DoH endpoint、ECH front host、
      恢复默认值；不暴露其它实现开关。
- [x] `09-01-ux-correctness` 已归档；已读取其 refresh/nested locator 契约。
- [x] 已确认不修改 `behavior-correctness-cleanup` 的 recovery owner，只输出 destination 模型。
- [x] 本文件与 `design.md` 已完成 review；用户已授权进入实现。

## 阶段 1：设置模型与迁移

- [x] 为预览/查看质量建立 typed enum，兼容读取旧 bool 并覆盖默认值与 schema 回写。
- [x] 增加 `NetworkMode` 持久化和 typed provider，重启/provider 重建后保持选择。
- [x] 清理 `nativeWebViewIntercept`、`insecureNoSniEnabled` 的设置 UI、controller wiring 与
      生产消费；保留必要的底层安全默认值。
- [x] 只有一个图片源时隐藏 UI，保留底层 host 校验。
- [x] 测试损坏字段独立回退、旧 schema 迁移、JSON 不泄漏凭据。
- [x] 阶段门：`flutter analyze`、`test/settings_test.dart`、相关网络/login 测试通过。

## 阶段 2：过滤谓词与发现类分页

- [x] 实现纯过滤谓词，覆盖 R18、AI、blocked tags 的交集/规范化规则。
- [x] 接入 Recommended、Ranking、Search、用户作品列表；收藏、历史、详情明确不接入。
- [x] 接入 `WidgetFeedLoader`（R18 保持基线过滤），错误与清理语义不变。
- [x] 实现有上限的续拉：cursor 去重、服务端耗尽、无过滤不续拉均有处理
      （`paged_feed_controller.dart`，`filterMaxRefillPages == 3`）。
      谓词单测在 `local_block_filter_test.dart`；一页过滤在
      `feed_generation_commit_test.dart`。续拉循环本身尚无单测。
- [x] 设置变更后失效/重新计算发现类 feed（基类 watch 过滤设置）。
- [x] 阶段门：feed/widget 单测与 widget test 通过。

## 阶段 3：下载目标与命名

- [x] 定稿 `lib/core/download/download_destination.dart` value object 和旧
      `savePath`/`saveFolder` 迁移规则（旧值无可验证语义 → 内置相册）。
- [x] 接通内置 PixivFunc 相册（`Pictures/PixivFunc`）、用户自定义相册
      （`Pictures/<name>`，MediaStore relativePath）与 SAF tree URI（系统选择器 +
      persistable permission；`SafTreeChannel.kt` + Dart 侧 `saf_tree.dart`）。
- [x] 将规范化 destination 传给 sink/channel（`DestinationAwareSinkFactory` 路由
      MediaStore/SAF；pending/finalize/abort owner 数据不变）。
- [x] 实现预设/受限模板、非法字符清理、长度裁剪、日期和多页编号（`naming_rule.dart`）；
      预览与实际落盘共用同一 owner，`DownloadRequest.displayName` 消费。
- [x] 不在本阶段改 recovery owner 算法；`DownloadDestination.identity` 为 cleanup
      留下稳定契约。
- [x] 阶段门：download/recovery 测试通过，API 29 目录选择路径待真机演示。

## 阶段 4：语言注入与设置 UI 收口

- [x] `PixivHttpClient` provider 监听 `languageTag`，请求 headers 使用当前 UI 语言。
- [x] 设置页移除实现名词和单选项图片源；普通网络页只保留三项入口（模式/诊断/高级），
      DoH/ECH 只在高级页。
- [x] 补齐多语言文案（zh/en/ja/ru）、错误态和可见设置状态。
- [x] 阶段门：`flutter analyze` 与设置 widget test 通过。

## 阶段 5：PixEz 式全出口网络快速层

- [x] 为 API、OAuth、图片、下载和 widget 共用的 policy 接入持久化/bootstrap host 地址。
- [x] Automatic 冷启动优先兼容地址并跳过 `HEAD`；失败的幂等请求保留严格路径回退，
      `directOnly` 行为不变。
- [x] 启动时预热共享 API/OAuth/image client 与图片缓存，成功请求后后台 DoH 刷新地址。
- [x] 增加 fast-route store、全出口无探测回归和重启持久化测试。
- [x] 对失效 bootstrap 地址增加 30 秒进程内冷却，并对相同 URI/token 的未取消 API GET 做单飞合并；
      不引入完成后的响应缓存，保证下拉刷新仍访问服务端。
- [ ] 阶段门：真机分别验证 API 首屏、OAuth 换 token、图片首屏、下载和 widget 刷新耗时。

## 最终验证

- [x] `flutter analyze`
- [x] `flutter test`
- [x] `git diff --check`
- [x] `flutter build apk --release --flavor github`（2026-09-07 check：88.2MB；无本地 keystore，出现 debug-signing 警告）
- [x] `flutter build apk --release --flavor fdroid`（2026-09-07 check：88.2MB）
- [ ] 用户真机验证：过滤、质量、下载三种目标、命名、多页、语言和网络模式重启保持。

## 回滚点

1. 模型/迁移；2. 过滤/续拉；3. 下载目标/命名；4. 语言/UI。每个阶段独立提交，失败只回滚
当前阶段，不覆盖其它 task 或用户改动。
