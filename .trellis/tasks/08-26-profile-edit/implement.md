# 实现本人资料编辑 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 审查beta56完整Profile edit流程，实时核验当前API/Web capabilities并记录结论。
2. 定义ProfileDraft、validation、repository/capability和account revision contract。
3. 实现表单/离开确认、图片选择裁剪、cancel/error状态。
4. 实现安全submit adapter和UserStore/AccountStore原子更新。
5. 增加字段/图片/竞态/账号切换/Web安全与Widget测试。
6. 运行analyze/test/build，在受控测试账号验证修改、失败和恢复。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-profile-edit
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- draft dirty/validation/leave guard。
- image MIME/size/pixel/crop/cancel/cleanup。
- server field errors与retry。
- account switch/base revision race。
- success store sync、failure rollback、Web bridge secret audit。

## Risky Files and Rollback Points

- lib/features/profile/edit/、UserStore/AccountStore update、image/WebView plugins、真实账号资料

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

