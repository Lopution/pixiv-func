# 执行计划：小说域修复与功能闭环

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `docs(09-16): novel repair 根因证据与设计固化`——提交本任务 prd/design/implement + 父任务与全部子任务规划文档（随本 PR 入库，沿用 09-02 先例）
- [x] `test(novel): webview 正文提取器与 detail 合并的失败测试`——fixture + 提取器 + fetchDetail 双源合并测试，先红（与下一勾合并提交：测试文件 import 新实现文件，拆开会使中间提交不可编译）
- [x] `fix(novel): 正文改走 /webview/v2/novel 内嵌 JSON`——fetchWebText + 提取器 + fetchDetail 合并；阅读器可渲染真实正文；内嵌图 URL 已入实体（渲染走后续读者改版）
- [ ] `feat(novel): 收藏 add/delete 与系列 prev/next 兜底`——BookmarkRepository/Store 扩展 + 详情页入口
- [ ] `feat(novel): 评论支持 novel 维度`——CommentStore (workKind,workId) 泛化 + repository 端点 + 详情页评论入口
- [ ] `feat(novel): 小说排行与热词`——/v1/novel/ranking feed + ranking 页入口 + trending-tags/novel
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed .`
- `flutter test`（全绿，含 layering_test 与新增 novel 测试）
- `git diff --check`

## 回滚点

每个 commit 独立可 revert；webview 提取器失败只影响小说详情（其余面不受影响）。
若 comment store 泛化风险大，在 commit 5 前可拆出为单独 PR。
