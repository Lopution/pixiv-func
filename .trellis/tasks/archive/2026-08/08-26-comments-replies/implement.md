# 复刻评论与回复 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 提取beta56评论/回复/asset/权限行为，核验当前comment endpoints。
2. 定义CommentEntity/Store/thread indexes/repository和composer。
3. 迁入emoji/stamp资产与pubspec注册，记录hash/provenance。
4. 实现列表、replies、item actions、composer grids、translate和delete。
5. 增加ID/parent regression、分页竞态、mutation、权限、asset grid和Widget测试。
6. 运行analyze/test/build；获授权后用测试评论验证create/reply/delete。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-comments-replies
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- commentId/parentId/rootId不变量。
- root/reply paging竞态和dedupe。
- reply insertion/delete/reply count。
- 10/5 grid和asset bundle。
- permission/account/translate/error与真实可清理mutation。

## Risky Files and Rollback Points

- lib/core/entity/comment_entity.dart、CommentStore、lib/features/comments/、assets/emojis/stamps、pubspec.yaml、真实评论数据

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

