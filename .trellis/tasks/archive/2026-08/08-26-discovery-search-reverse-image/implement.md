# 复刻发现、搜索与反向搜图 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start；中间父任务本身不作为实现目标。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 依次审阅并批准 ranking-feed、new-content-feeds、search-catalog 规划。
2. 完成前三个叶子任务后审阅 reverse-image-search 的实时服务 research 并批准。
3. 核对四个子任务共享 store、generation-scoped paging commit、route、错误和账号隔离契约。
4. 运行 refresh/append竞态、跨 Home tabs、详情/收藏、返回/状态保持、SEND image/provider capability 的集成回归。
5. 记录中间父任务 acceptance evidence 并归档。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-discovery-search-reverse-image
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 四叶子 task validate/archive 状态。
- Home tab/scroll/query state integration。
- 跨入口同 ID entity/bookmark 同步。
- refresh-wins、old-generation entity/cursor rejection、filter/account generation isolation。
- 账号切换和 error/offline isolation。
- Reverse Image app picker 与 Android SEND 到详情链路。

## Risky Files and Rollback Points

- 四个 child task contracts、Home tabs、shared stores/routes；本任务不直接编辑业务实现

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。
