# 恢复原版 iconFont 字体资产

## Goal

恢复 beta56 的原始字体图标资产及注册，使当前 Home shell 和后续页面使用与原版一致的 glyph，而不是 Material fallback。

## Confirmed Facts

- 当前 lib/app/icons/app_icons.dart 已保存 0xe900–0xe90d 映射，但 pubspec.yaml 尚未注册字体，仓库也没有 assets/icon.ttf。
- beta56 commit 的 assets/icon.ttf、lib/app/icon/icon.dart 与 pubspec.yaml 提供二进制来源、codepoint 和 family 名 iconFont。
- 当前 Home 前四项使用 AppIcons，第五项原版就是 Icons.settings。

## Dependencies

- 08-26-flutter-android-scaffold 完成并形成可提交基线。

## Requirements

- R1: 从固定 beta56 commit 导入原始 assets/icon.ttf，记录来源 commit 与 SHA-256；不得使用相似字体或重绘资产替代。
- R2: 在 pubspec.yaml 注册 family iconFont，并保持全部既有 AppIcons codepoint 与 matchTextDirection 语义。
- R3: Home 的 Recommended、Ranking、New、Search 使用字体图标，Settings 保持 Icons.settings；不得引入 Material fallback。
- R4: 增加能发现资产缺失、family 拼写错误和 Home 图标回退的聚焦测试或确定性资源检查。
- R5: 除字体资源和必要注册/测试外，不调整布局、颜色、尺寸或导航行为。

## Acceptance Criteria

- [ ] assets/icon.ttf 的 SHA-256 与 beta56 固定 commit 中资产一致，并在任务 research 中记录。
- [ ] flutter pub get 后 AssetManifest 可解析该字体，Home 五个图标来源符合要求。
- [ ] flutter analyze、flutter test 和 flutter build apk --debug 全部通过。
- [ ] APK 中包含字体资产；至少完成一次可观察的渲染检查，未出现缺字方框或 Material 替代图标。

## Out of Scope

- 重新设计图标或 Home 导航。
- 迁移 emoji/stamp 等其他资产。
- 修改字体文件本身。

## Risks and Deferred Items

- 二进制资产无法通过普通 diff 审查，必须依赖固定 commit、SHA-256 和 APK/渲染验证。

## Source Anchors

- beta56 assets/icon.ttf、lib/app/icon/icon.dart、pubspec.yaml
- 当前 lib/app/icons/app_icons.dart、lib/features/home/home_page.dart、pubspec.yaml

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
