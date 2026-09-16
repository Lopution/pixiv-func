# 卡片长按快捷操作与稍后再看

## Goal

信息流卡片从"只能点开"升级为长按呼出快捷操作菜单；新增纯本地 watch-later 暂存。

## Requirements

- `IllustCard` 增加长按手势槽；菜单渲染归 app 层，动作经「动作注册表」分发到 core 域（父 design §2）
- 菜单项（首版）：收藏/取消收藏、下载、稍后再看、屏蔽此作品、屏蔽作者、分享（屏蔽项在 mute-system 落地后接入，先留注册位）
- `core/watchlater/`：本地暂存（账号隔离、可增删、时间序）；`features/watchlater/` 列表页
- 不打断现有 tap→Hero→详情/查看器手势与 `onTapDown` 预加载

## Acceptance Criteria

- [ ] 长按菜单在所有 IllustCard 场景一致（推荐/排行/搜索/相关/用户作品）
- [ ] watch-later 增删、列表浏览、从列表进入详情可用；纯本地不产生网络请求
- [ ] widget 测试覆盖手势与动作分发；语义标签完整（无障碍）
- [ ] 菜单弹出动效走 motion token（与 motion-layer 的先后以实际落地顺序在 design 中记录）

## References

- Shaft：`ui/watchlater/`（WatchLaterFeedFragment/Tabs）；卡片长按菜单交互
- PixEz：`illust_row_page`/`picture_list_page` 长按行为

## Dependencies

被 `mute-system` 依赖（屏蔽入口）。
