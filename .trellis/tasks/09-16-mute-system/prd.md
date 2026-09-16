# 屏蔽体系升级

## Goal

从"仅本地 tag+R18+AI 隐藏"升级为 tag/user/单作品三态屏蔽 + 模糊揭示 + 管理页 + 服务端同步。

## Requirements

- `core/mute/`：MuteStore（账号隔离、kind=tag|user|work、confirmed/pending/error、revision 协议沿用 BookmarkStore 模式）
- 服务端同步：`/v1/mute/list`、`/v1/mute/edit`（add/remove tags+users；官方不支持单作品 → 单作品仅本地）
- 卡片呈现：被屏蔽条目不消失，渲染模糊+揭示按钮变体（父 design §2）；设置项可切回"直接隐藏"
- 屏蔽管理页：按 kind 分组的列表、移除、跳转对象
- 与 `local_block_filter`（R18/AI/tag）归并关系在本 child design 中裁决并写回 spec
- 长按菜单注册"屏蔽此作品/屏蔽作者"项（依赖 card-quick-actions 的注册表）

## Acceptance Criteria

- [ ] 三态屏蔽对全部 feed 生效；模糊揭示可临时查看且不影响已屏蔽状态
- [ ] 服务端同步双向一致；失败有可见错误与本地兜底；离线进入离线队列（若 feed-resilience 已落地）
- [ ] 管理页增删查可用；账号切换边界正确
- [ ] store 合并/revision/边界测试齐全

## References

- Shaft：`ui/muted/`（MutedTagsFeedFragment/MutedUserFeedFragment/MutedObjectsFeedFragment + MuteTagSheet）
- pixes：`network.dart` getMuteList/editMute（服务端端点用法）

## Dependencies

依赖 `card-quick-actions`（菜单入口）。
