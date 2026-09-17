# 网络与账号设置增强

## Goal

镜像易用性 + 服务端显示设置同步 + 数据可携带。

## Requirements

- 图片镜像一键预设：`i.pixiv.cat` / `i.pixiv.re` / `i.pixiv.nl` / 自定义反代前缀（`https://host[/path]`），落到现有 host override 底层（`network_policy` 白名单 + image client URL 重写），探针页可测镜像；preset 同时映射 `i.`/`s.` pximg 子域
- 服务端同步：`GET /v1/user/ai-show-settings`(`{show_ai}`) + `POST /v1/user/ai-show-settings/edit`(form `show_ai`)、`GET|POST /v1/user/restricted-mode-settings`(`is_restricted_mode_enabled`)；服务端开关与本地 `enableLocalBlockAI/R18` 过滤并列展示、各自独立（不同语义层）
- 数据导出导入：设置 + 屏蔽（tags/users/workIds）+ 历史（JSON 文件，`pixivfunc.backup.v1` schema，SAF/file_selector 读写）

## Acceptance Criteria

- [ ] 镜像切换即时生效（policy 按 imageSource watch 重建）且图片加载走所选 host；自定义反代可校验连通性
- [ ] 服务端设置读写在账号边界内一致；失败可见（乐观更新 + 回滚 + SnackBar）
- [ ] 导出文件可重新导入；导入冲突策略明示（合并/覆盖）；屏蔽导入只增不删（服务端列表与官方客户端共享）

## References

- Shaft：`ceui/lisa/http/ImageHostManager.kt`（Mode 枚举 + i./s. 双映射 + CUSTOM 全前缀 + requiresStandardClient）
- PixEz：`er/pixiv_image_source.dart`（resolveUri 前缀拼接）、`network/api_client.dart`（ai-show/restricted-mode wire）、`page/hello/setting/data_export_page.dart`

## Dependencies

无。
