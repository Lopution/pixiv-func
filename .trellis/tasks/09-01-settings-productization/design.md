# 技术设计：设置页产品化与真实消费者接线

设计基线：HEAD `409df51`；实现前必须先读取已归档的 `09-01-ux-correctness` 最终契约。

## 1. 边界与不变契约

本 child 负责设置模型、设置 UI、发现类 feed / widget 的本地过滤、图片质量、下载目标与
命名、网络模式持久化和 API 语言注入。它不重新设计网络协议、不实现翻译或反向搜图 provider，
也不提前修改 download recovery 的 owner；后者由 `behavior-correctness-cleanup` 消费本 child
定稿后的 destination 模型。

必须保持：API 29、`AppSettings` 旧 JSON 可迁移、账号凭据不进入普通设置或迁移剪贴板、详情/收藏/
历史不受本地过滤影响、图片 host 仍由 `PixivDestinationRegistry` 管理。

## 2. 模型与迁移

- 将 `previewQuality` / `scaleQuality` 收敛为已有质量语义的 typed enum（预览默认中图、查看默认
  原图），保留旧 bool 读取兼容；迁移只在读取旧 key 时发生，写回当前 schema。
- 增加持久化 `NetworkMode`（`automatic` / `directOnly`），只保存用户选择，不保存 route memory
  或探测结果。网络 provider 只消费这个字段。
- 将下载目标抽象为明确的模式和值：内置 PixivFunc 相册、用户选择的 MediaStore 相册、SAF tree
  URI。`savePath` / `saveFolder` 的旧值按可验证规则转换；无法安全转换时回到内置相册并给出可见
  的设置状态，不把旧字符串当作任意文件路径执行。
- 命名规则由预设 ID 或受限模板组成。模板只允许 `{artist}`、`{title}`、`{id}`、`{page}`、
  `{ext}`、可选 `{date}`；在单一 owner 中完成非法字符清理、长度裁剪和多页编号。
- 屏蔽规则实现为纯谓词 `isLocallyBlocked(IllustEntity, Set<String>)`，不依赖 UI、网络或
  `settingsProvider` 的副作用；`blockedTagsProvider` 与 R18/AI 开关共同形成输入。

## 3. UI 与数据流

### 3.1 过滤

发现类 feed 在把服务端页提交到 `PagedFeedController` 前使用同一纯谓词；被过滤页为空或低于
阈值时，沿用现有 cursor/load-more 机制有限续拉。为避免空页面触发无限请求，设置最大连续续拉
次数，并在服务端耗尽时正常结束。`WidgetFeedLoader` 直接复用该谓词和相同上限；收藏、历史、
详情路径不调用它。

设置变更通过 Riverpod provider 使发现类列表失效或重新计算；返回页面即可看到新结果，不依赖
重启或用户手动刷新。UX child 已完成的 nested refresh 行为保持不变。

### 3.2 下载设置

最终采用“保存位置”单一入口，先选择“相册 / 文件夹”模式。相册模式默认
`Pictures/PixivFunc`，自定义相册名只接受规范化名称；文件夹模式只通过系统 SAF 目录选择器
取得 tree URI 和持久权限。Dart request 只接收规范化后的目标描述，sink/platform channel
负责写入；不提供文本路径输入和自制文件浏览器。

### 3.3 网络与质量

普通网络页仅显示网络模式、网络诊断、高级设置；DoH/ECH 等实现名词只留在高级页。删除
`nativeWebViewIntercept` 与全局 `insecureNoSniEnabled` 的用户设置和 wiring。Automatic 的
Pixiv 原生网络出口统一复用 PixEz 式兼容快速层：API、OAuth、图片、下载和 widget 后台先使用
持久化或 bootstrap host 地址、共享连接池，并跳过冷启动 `HEAD`；成功后异步用 DoH 更新地址。
失败的幂等请求仍可进入严格梯子，失效 fast 地址在进程内冷却 30 秒，`directOnly` 不启用兼容
层。API client provider 监听 `languageTag` 并把它传给 headers/client。相同 URI 与 bearer token
的未取消 GET 共享一个 in-flight 请求，但不保留完成后的业务响应，因此下拉刷新不会读到旧缓存。

图片源只有一个产品值时隐藏整个选择项，不删除底层兼容枚举。

## 4. 文件责任

| 责任 | 主要文件 |
|---|---|
| schema/迁移/provider | `lib/core/settings/app_settings.dart`、`settings_controller.dart`、`settings_repository.dart` |
| 过滤 | 新增或扩展 `lib/core/settings/blocked_tags.dart` 的纯谓词；各发现 feed 与 `widget_feed_loader.dart` |
| 下载目标/命名 | `lib/core/download/download_request.dart`、`download_sink.dart`、`download_manager.dart`、`media_store_channel.dart` 及 SAF 平台接口 |
| 设置 UI | `lib/features/settings/settings_page.dart`、`network_settings_page.dart` |
| 语言/网络注入 | `lib/core/network/pixiv_headers.dart`、`pixiv_http_client.dart`、对应 provider |
| PixEz 兼容快速层 | `lib/core/network/compat/network_fast_route_store.dart`、`network_policy.dart`、`network_providers.dart` |
| 回归 | `test/settings_test.dart`、feed/widget/download/http 测试；必要时新增纯谓词测试 |

## 5. 风险与回滚

- destination 模型和 recovery 记录是跨 child 契约：先提交模型与设置侧测试，再通知 cleanup 消费；
  不在本 child 修改 recovery owner 判定。
- 过滤会改变发现类列表的页数，必须验证 cursor、去重、续拉上限和 widget 独立路径；出现无限
  请求、空成功或详情被误过滤立即回滚该阶段。
- 删除设置字段前先保留旧 JSON 的读取迁移测试；删除点与 UI/wiring 同一阶段提交，失败只回滚
  当前阶段。
- 每一阶段都运行 `flutter analyze`、相关测试、`git diff --check`；API 29 目录选择、MediaStore
  pending/finalize/abort 和两个 flavor 由最终集成阶段验证。
- 兼容快速层是 Automatic 的内部传输实现，不扩大为设置项；其 host 地址只接受公开 Pixiv
  目的地、成功请求后的 DoH 刷新和有限 bootstrap，无法刷新时仍保留可观察的严格梯子回退。

## 6. 已确认的产品选择

用户于 2026-09-02 确认：自定义相册与 SAF 文件夹按 3.2 的单一入口呈现；网络高级页只保留
DoH endpoint、ECH front host 和恢复默认值。完整决策记录见 parent 的
`research/decision-record-2026-09-02.md`。
