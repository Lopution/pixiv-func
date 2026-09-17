# 设计：网络与账号设置增强

## 证据基线（已确认）

- `AppSettings.imageSource` 现为单 host 字符串，`ImageSourceMode` 仅 `normal('i.pximg.net')`；`selectImageSource` 用 `fromHost` 校验。browse 页注释明确「单一选项无意义，隐藏至第二个产品级源出现」——本任务即触发条件。
- 图片流量路径：`PixivImage.provider(url)` → `CachedNetworkImageProvider` → `PixivImageCache`(`HttpFileService`) → `policy.client(PixivDestinationPurpose.image)`。
- `PixivDestinationRegistry._allows(image)` 只允许 `PixivClientIdentity.downloadHosts = {i.pximg.net, s.pximg.net}`；host 精确匹配（小写、无端口、无 userinfo、仅 https）。镜像 host 必须进白名单，否则被 `PixivDestinationException` 拒。
- `networkAccessPolicyProvider` 已 watch DoH/ECH/mode 并重建 policy——`imageSource` 变更走同一路径即可「即时生效」（镜像 host 无 fast-route 记录，自然走 standard 路由，等价 Shaft `requiresStandardClient`）。
- 参考语义（`/root/pixiv-audit`）：Shaft `ImageHostManager`——Mode {PIXIV, PIXIV_CAT, PIXIV_RE, PIXIV_NL, CUSTOM}；preset 同时映射 `i.`/`s.` 子域（`i.pixiv.cat`/`s.pixiv.cat`）；CUSTOM = 全 URL 前缀 `https://host[/path]` 整体替换、保留原 path/query；PixEz `PixivImageSource.resolveUri` 支持自定义源带 path 前缀（`_joinPaths`）。
- 服务端设置 wire（PixEz `api_client.dart` 实证）：
  - `GET /v1/user/ai-show-settings` → `{show_ai: bool}`
  - `POST /v1/user/ai-show-settings/edit` form `show_ai=true|false` → 回 `{show_ai}`
  - `GET /v1/user/restricted-mode-settings` → `{is_restricted_mode_enabled: bool}`
  - `POST /v1/user/restricted-mode-settings` form `is_restricted_mode_enabled=true|false`
- 本地 `enableLocalBlockAI`/`enableLocalBlockR18` 是客户端过滤（`local_block_filter.dart`），与服务端开关语义不同层，保持独立。
- 导出基础设施：`safTreePicker`(Android SAF / desktop `getDirectoryPath`) + `safDocumentSinkFactory.create(treeUri, displayName, mimeType)` 写文件；导入用 `file_selector` `openFile(XTypeGroup)`。
- `account_transfer.dart` 已有 versioned envelope + 校验导入的范式。
- Mute：`MuteState{tags,users,workIds}`；tags/users 服务端同步（`/v1/mute/list|edit`），workIds 纯本地按账号存 prefs；`MuteRepository.edit` 为服务端写口。
- History：`HistoryRepository.page(accountId)` 分页、`upsert`、`clear(accountId)`；`HistoryRecord.toColumns/fromRow` 已是可序列化形态。
- Settings：`AppSettings.toJson/fromJson(fallback)` + `schemaVersion`，`PreferencesSettingsRepository` 写 `replica.settings.v2`。

## 1. 图片镜像

**配置模型**：`AppSettings.imageSource` 语义升级为「图片源前缀」，持久化仍为单字符串：`i.pximg.net`（默认）| `i.pixiv.cat` | `i.pixiv.re` | `i.pixiv.nl` | 自定义完整前缀 `https://host[:port][/path]`。`ImageSourceMode` 枚举扩展为 {normal, pixivCat, pixivRe, pixivNl, custom}；`fromHost` 无法识别时若字符串是合法 https host 前缀则归 custom（新增 `customImageSource` 字符串字段保存原始前缀——参考 Shaft mode+customHost 双字段）。

**URL 重写**：新 `ImageMirrorResolver`（`core/network/` 或 `core/settings/` 旁）：
- 仅当 `uri.host ∈ {i.pximg.net, s.pximg.net}` 时重写；其余原样返回。
- preset：`i.pximg.net→i.pixiv.cat`、`s.pximg.net→s.pixiv.cat`（re/nl 同理）。
- custom：前缀整体替换 scheme://authority[/path-prefix]，拼接原 path+query（对齐 Shaft/PixEz 语义）。
- 非法 URL/空前缀一律返回原 URL（fail-open，不引入静默失败通道之外的复杂度）。

**接入点**：`PixivImageCache` 的 `HttpFileService` 前包一层 rewrite `http.Client`（或自定义 FileService）：先 `resolver.resolve(request.url)` 再交给 policy client。这样 policy/registry/route memory 全部按镜像 host 键控，复用既有冷却/池化语义；下载通道（`policy_download_transport`）同样走 image purpose，重写放同一 resolver 口。

**白名单**：`NetworkAccessPolicy`/`PixivDestinationRegistry` 接受 `extraImageHosts` 参数；`networkAccessPolicyProvider` watch `imageSource` 派生活动 host 集合传入。custom 前缀的 host 也进白名单；scheme 非 https 的自定义前缀在校验期拒绝。

**探针**：`network_probe.dart` 增一条 image-mirror 步骤（对当前生效 host 发 GET 已知小图 `/img-inf/` 或 HEAD 任一 pximg 路径，校验 2xx + 非空）；probe 页列出所选镜像的连通性结果。自定义输入框旁提供「测试」按钮直接跑该探针。

**UI**：browse 页移除 `sources.length > 1` 门控，渲染 preset 单选 + 自定义行（输入 + 测试 + 校验错误可见）。

## 2. 服务端显示设置

新 `ServerDisplaySettingsRepository`（`core/settings/` 或 `core/profile/` 旁，按 mute_repository 的 `PixivHttpClient` 范式）：

```dart
Future<bool> fetchAiShow();            // GET ai-show-settings → show_ai
Future<bool> editAiShow(bool v);       // POST .../edit form show_ai → 回声
Future<bool> fetchRestrictedMode();    // GET → is_restricted_mode_enabled
Future<bool> editRestrictedMode(bool); // POST form is_restricted_mode_enabled
```

均为 authed 调用 → 天然账号边界；切账号后重新拉取（provider 按 accountId 键控或在 controller 层跟随 currentAccount）。

**对齐策略**：服务端开关 = 服务器返回内容的权威；本地 block 过滤 = 对已返回内容的客户端兜底。两者并列展示、各自独立，不做隐式联动（用户可见两个不同区域：「账号显示设置（服务端）」与「本地过滤」）。写失败回滚 optimistic 状态 + SnackBar。

**UI**：account_settings_page 增「账号显示设置」section，进入时拉取两项；toggle 乐观更新、失败回滚并提示。

## 3. 数据导出导入

新 `core/backup/`（命名遵循 core 域惯例）：

```jsonc
// pixiv-func-backup-YYYYMMDD-HHmm.json
{
  "schema": "pixivfunc.backup.v1",
  "exportedAt": "ISO-8601",
  "accountId": "…",           // 导出时的账号 id，导入仅提示不强制
  "settings": { …AppSettings.toJson()… },
  "mutes": { "tags": [], "users": [{"userId":1,"name":""}], "workIds": [] },
  "history": [ {…HistoryRecord 行形态…} ]
}
```

- **导出**：settings.toJson + 当前 MuteState + `HistoryRepository.page` 全量翻页 → `safTreePicker.pickTree` 选目录 → `safDocumentSinkFactory.create(mimeType: application/json)` 写流。
- **导入**：`openFile` 选 JSON → 校验 `schema` 与版本 → 弹「合并 / 覆盖」选择：
  - 合并：settings 覆盖写（单一对象无可合并粒度）；mute = tags/users 走 `muteRepository.edit` 逐个 add（服务端语义无删除）、workIds 本地并集；history = upsert（按 identity 保留较新 `last_viewed_at`）。
  - 覆盖：settings 覆盖写；workIds 替换为导入集（服务端 tags/users 仍走 add——覆盖语义不主动删服务器数据，UI 明示）；history = `clear(accountId)` 后整体插入。
- 校验失败（坏 JSON、未知 schema/version、字段类型错）抛带公开消息的 `BackupImportException`，UI 可见。
- **安全**：settings 内含 translation credential 的 `SecretSettingRef`——导出引用字段（ref id）而非凭据本体；凭据不进备份。

## 风险

| 风险 | 缓解 |
|---|---|
| 镜像站内容/证书不可靠 | 仅 https、校验连通性后再保存；fast-route 不会缓存非 pximg host |
| 服务端设置 GET 字段名漂移 | mapper 集中在 repository，单测钉 `{show_ai}`/`{is_restricted_mode_enabled}` |
| mute 导入对服务端列表只做 add | UI 文案明示「导入只增不删（与官方客户端共享服务端列表）」 |
| SAF create 需要 treeUri | 复用 download_destination 已验证的 pickTree→create 链路 |
