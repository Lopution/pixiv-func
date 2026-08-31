# 实施计划：移动端 UI 动画与个人页问题

## 顺序

1. **加载前置规范**
   - [x] 执行 `trellis-before-dev`，读取 frontend 组件/状态规范。
   - [x] 检查工作树，只保留本任务目录的规划文件，避免覆盖已有用户改动。

2. **Hero 作用域与快照传递**
   - [x] 扩展 `IllustCard` 的可选 `heroScope`，点击详情时传递 `heroScope` 与 `initialEntity`。
   - [x] 给推荐、排行、新作、搜索、个人页作品流设置稳定 scope；同步更新非卡片直接入口的详情构造参数。
   - [x] 扩展 `IllustDetailPage`，首帧读取 store/initialEntity，app bar、正文和每页 Hero 使用同一 scope。
   - [x] 检查多页作品首图和普通无来源入口的 tag 行为，避免重复 tag 或无目标时的额外动画。

3. **简介默认展示**
   - [x] 移除 `_showCaption` 及展开控件，保留 `_CaptionRichText` 的渲染和链接路由。
   - [x] 增加非空/空 caption widget 测试，确保不再出现按钮和空白占位。

4. **个人资料设置入口**
   - [x] 删除 `UserPage` 内部隐式 `SettingsPage` fallback。
   - [x] 移除 profile header 的设置图标及 `onSettings` 参数链路。
   - [x] 增加设置页账户卡片进入 `MePage` 的路由/控件回归测试。

5. **验证与设备回归**
   - [x] 运行聚焦 widget tests、`flutter analyze --no-pub`、全量 `flutter test --no-pub --concurrency=1`、`git diff --check`；全量测试记录两个既有 loopback 超时。
   - [x] 构建 `flutter build apk --debug --flavor github`。
   - [ ] 在真实手机无代理环境复测：当前 `adb devices -l` 无设备，待设备连接后复测四条路径。

## 目标文件

- `lib/features/home/recommended/recommended_illust_page.dart`
- `lib/features/home/recommended/recommended_home_page.dart`
- `lib/features/ranking/ranking_page.dart`
- `lib/features/new/new_page.dart`
- `lib/features/search/search_result_page.dart`
- `lib/features/profile/user_page.dart`
- `lib/features/illust/detail/illust_detail_page.dart`
- `test/illust_detail_page_test.dart`
- `test/user_profile_test.dart`
- `test/settings_test.dart`

## 验收检查

- [x] Hero scope 在每个卡片入口和详情目标之间一致，个人页与底层信息流 scope 不相等。
- [x] 详情首帧在 API Future 未完成时已经有正文、作者信息和 Hero 目标。
- [x] 简介默认显示，空 caption 没有多余控件/间距，现有链接测试继续通过。
- [x] 设置页账户卡片打开的个人资料没有设置图标，个人资料入口不再重复推入设置页。
- [x] 聚焦测试、分析、APK 构建和 diff 检查均有真实输出记录；全量测试有两个环境性 loopback 失败。

## 不做事项

- 不改 `ReplicaPageRoute` 曲线，不引入全局 Hero controller 或缓存预热系统。
- 不等待网络请求后再导航，不把图片是否命中缓存伪装成确定成功。
- 不新增与这四个缺陷无关的防御性 null 分支、重试机制或状态副本。
