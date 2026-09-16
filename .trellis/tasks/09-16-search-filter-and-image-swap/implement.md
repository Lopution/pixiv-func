# Implement Plan

## 1. is_premium 解析与持久化

- `lib/core/auth/oauth_service.dart`：`OAuthUserProfile` 加 `isPremium`（`user['is_premium'] == true`），`_parseTokenResponse` 填充；login 与 `refreshSession` 共用此解析，天然两处同步。
- `lib/core/auth/account.dart`：`Account.isPremium`（默认 false）、`copyWith`/`toJson`/`fromJson`/`==`/`hashCode` 全链路透传。
- 调用点补传：`login_page.dart`、`login_webview_page.dart`、`login_webview_desktop_page.dart` 的 `Account(...)`；`pixiv_http_client.dart` refresh 路径 `copyWith(isPremium: result.profile.isPremium)`。
- `account_transfer_service.dart` 导入路径保持默认 false（导入包无 premium 信息）。

## 2. 搜索请求语义修正（search_models.dart）

- `SearchFilters.toQuery`：
  - `duration != null` → 换算 `start_date`/`end_date`（`DateTime.now()` today−{1,7,30} ~ today），不发 `duration`。
  - `search_target` 一律显式发送（服务端默认即 `partial_match_for_tags`，省略无收益；与 PixEz/pxview 一致）。
  - duration 与 start/end 同时存在时 duration 优先并覆盖日期（防御；UI 已互斥）。
  - `cacheKey` 不变（duration wireValue 仍在 key 里，身份不变）。
- `IllustSearchQuery`/`NovelSearchQuery` 增加 `toPreviewQuery()`：同 toQuery 去掉 `sort`（preview 端点自带人气序）。

## 3. popular-preview 路由（search_repository.dart + next_page_parser.dart）

- `_kNextPageEndpoints` 增加：
  - `/v1/search/popular-preview/illust`: word, search_target, duration?, start_date, end_date, filter, offset
  - `/v1/search/popular-preview/novel`: 同上
- `searchRepositoryProvider` watch `accountStoreProvider.select((s) => s.value?.current?.isPremium ?? false)` → `_PixivSearchRepository` 持有 `isPremium`。
- `_spec(query)`：sort==popularDesc && !isPremium → preview path + `toPreviewQuery`；否则原路径 + `toQuery`。`requiredQuery` 对应替换（next_url 校验继续工作）。

## 4. 筛选 sheet 互斥（search_filter_sheet.dart）

- 选 duration chip → `copyWith(duration: v, startDate: null, endDate: null)`。
- `_pickDate` 选中日期 → `copyWith(startDate/endDate: v, duration: null)`。
- Apply 前校验 `startDate > endDate` → snackbar/禁用（用现有 `showAppSnackBar` 或 disabled 按钮；选简单方案：onPressed 置 null + subtitle 红字提示？定稿：按钮禁用 + 在 end tile subtitle 显示错误文案——用现有 l10n 加新 key `searchInvalidDateRange`）。

## 5. PixivImage 坑位交接（pixiv_image.dart）

- `PixivImage` 改为 `ConsumerStatefulWidget`；State 记 `_lastShownUrl`。
- `slotHandoff = _lastShownUrl != null && _lastShownUrl != widget.imageUrl` → `crossfade = fade && previousTransition == null && !slotHandoff`。
- build 末尾更新 `_lastShownUrl = imageUrl`。
- 其余逻辑（transitionKey 历史、transitionPlaceholder、`_recordWhenDecoded`、preload/decodeWidthFor）不动。

## 6. feed 卡片 key

- `IllustCard({Key? key, required this.entity, ...}) : super(key: key ?? ValueKey('illust-${entity.id}'))`。
- `NovelCard` 同样处理（查 entity id 字段名）。

## 7. 测试

- `search_catalog_test.dart`：duration→日期断言更新；新增 popular-preview 路由（premium/非 premium 两路）、search_target 省略断言。
- 新增/更新 pixiv_image 测试：同 element 换 url → fade 0；新 element + transitionKey 历史 → transitionPlaceholder；全新 → crossfade。
- feed 卡片 key 断言（可选）。

## 8. 验证

`dart format` → `flutter analyze` → 全量 `flutter test` → `git diff --check` → arm64 github APK 打包。

## Rollback

单 commit revert 即可；无数据迁移（isPremium 缺失字段默认 false）。
