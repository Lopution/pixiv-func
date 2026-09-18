# 验收反馈修复：离线入队/搜索筛选持久化/设置分组/详情图与滚动空白

## Goal

父任务验收中发现的问题桶：断网收藏/关注未入队（传输层异常逃逸分类器）；搜索筛选状态仅存 setState 未持久化；设置页分组混乱；详情页图片 hero 落位偏小+空白（解码固有尺寸驱动布局）；快速滑动回看图片大片空白（预取被 deferred 拦截）。每 issue 一勾一 commit。

## Requirements

- TBD

## Acceptance Criteria

- [ ] TBD

## Notes

- Keep `prd.md` focused on requirements, constraints, and acceptance criteria.
- Lightweight tasks can remain PRD-only.
- For complex tasks, add `design.md` for technical design and `implement.md` for execution planning before `task.py start`.
