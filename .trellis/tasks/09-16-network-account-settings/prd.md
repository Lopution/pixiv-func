# 网络与账号设置增强

## Goal

镜像易用性 + 服务端显示设置同步 + 数据可携带。

## Requirements

- 图片镜像一键预设：pixiv.cat / pixiv.re / pixiv.nl / 自定义反代，落到现有 host override 底层（`network_policy`），探针页可测镜像
- 服务端同步：`/v1/user/ai-show-settings`(+edit)、`/v1/user/restricted-mode-settings` 读写，与本地开关对齐策略
- 数据导出导入：设置 + 屏蔽 + 历史（JSON 文件，版本号 schema）

## Acceptance Criteria

- [ ] 镜像切换即时生效且图片加载走所选 host；自定义反代可校验连通性
- [ ] 服务端设置读写在账号边界内一致；失败可见
- [ ] 导出文件可重新导入；导入冲突策略明示（合并/覆盖）

## References

- Shaft：镜像切换（README 网络自检节）、`db/mirror/`
- PixEz：`page/network/network_select.dart`、`data_export_page.dart`

## Dependencies

无。
