# Novel typed markup 与长文预算 — Implementation Plan

## Start Gate

- 在 `08-26-novel-reader` 的现有交互契约完成复核并通过本叶子 planning review 后启动；不修改归档目录。

## Steps

1. 阅读 `novel_entity.dart`、`novel_repository.dart`、`novel_layout.dart`、`novel_reader.dart` 及 `test/novel_reader_test.dart`，记录现有 parser/layout 假设和 beta56 页行为。
2. 添加 token fixtures，先让 newpage/chapter/jump/pixivimage/uploadedimage/unknown 的失败现状可复现。
3. 实现 typed scanner/mapper，保留原始 unknown；把 jump/image 验证和共享 network request 分离。
4. 给 parser/layout 增加 bounded chunk、取消、progress 和 generation commit gate，保持水平 reader UI。
5. 补齐 token、Unicode、页边界、超限、取消和章节切换测试；按真实 API/设备情况记录证据。
6. 更新 integration typed-markup evidence，不扩张 Novel 功能。

## Validation

```bash
/opt/flutter-3.47.0/bin/flutter test test/novel_reader_test.dart
/opt/flutter-3.47.0/bin/flutter analyze
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-novel-markup-hardening
git diff --check
```

若触及图片/Android 渲染，再增加 debug build 和 API 36 设备验证；解析通过不能代替真实 Novel API 验收。

## Completion Gate

- [ ] typed token 和 unknown preservation 覆盖所有审查标记。
- [ ] 长文在预算内分块、可取消，旧章节结果不能提交当前 reader。
- [ ] jump/image 输入安全失败，归档无 diff，integration evidence 已回填。
