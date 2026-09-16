# 小说域修复与功能闭环

## Goal

小说页/阅读器当前运行期不可用（用户 2026-09-16 报告）。先定位根因并修复，再补齐小说功能闭环。

## Requirements

- 根因定位：复现小说页打开/阅读器进入的失败路径，记录错误与调用链（先证据后修复，禁止绕过式修补）
- 修复小说详情页与阅读器：详情可打开、正文可取、翻页/排版可用、系列内章节可切换
- 小说收藏：`/v2/novel/bookmark/add` + `/v1/novel/bookmark/delete`（BookmarkStore 协议扩展）
- 小说评论：`/v3/novel/comments`、`/v2/novel/comment/replies`、`/v1/novel/comment/add`（CommentStore 域扩展）
- 小说排行：`/v1/novel/ranking` feed 页；`/v1/trending-tags/novel`
- 系列页与详情页入口互通打磨

## Out of scope

本地 txt 库 / TXT 导出 / watchlist → `watchlist-local-library`。

## Acceptance Criteria

- [ ] 根因写入任务记录（什么坏、为什么坏、如何防回归）+ 回归测试
- [ ] 小说详情/阅读器/系列/排行/评论在真机或测试中走通
- [ ] 收藏变更遵循 mutation revision 协议，pending/fail 可观测
- [ ] 新 feed 复用 PagedFeedController + FeedCommitGate

## References

- 本仓：`lib/core/novel/`、`lib/features/novel/`、`lib/core/comments/`
- PixEz：`/root/pixiv-audit/pixez-flutter/lib/page/novel/`（rank/series/comment/viewer 全套）
- skana_pix：`/root/pixiv-audit/skana_pix/lib/utils/text_composition/`（排版引擎参考）

## Dependencies

无（第一优先级）。blocks `watchlist-local-library`。
