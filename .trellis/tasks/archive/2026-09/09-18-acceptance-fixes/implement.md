# 执行计划：验收反馈修复桶

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。
> 用户在真机验收 09-16 父任务时发现的问题，逐项修复；新发现的问题追加到本桶。

- [x] `fix(actionqueue): 断网时收藏/关注/追更真正入队`——`isConnectivityError` 改用 `TransportFailureClassifier`：policy 层抛的 `NetworkFailureException`/`SecureResolutionException` 逃逸了 `on ApiError`/`isConnectivityError` 双保险 → 落 `on Object` → `store.fail` 弹错且不入队。连通类 kind（dns/connect/timeout/reset/tlsHandshake/rateLimit/SecureResolution/HTTP≥500）入队，certificateMismatch/auth/4xx 维持可见失败 + 回归测试
- [x] `feat(search): 搜索筛选条件持久化`——`SearchFilters` 序列化进 settings 存储（SharedPreferences 路径），启动回填、sheet 改动即写；premium 排序等字段与会员状态的交互维持现状
- [x] `refactor(settings): 设置页分组重构`——13 个平铺 tile 按 Shaft hub 模型分组（账号/外观/网络与浏览/内容/下载/数据/关于），分组标题替代裸 Divider，「稍后再看」移出设置页（它是功能入口不是设置项）
- [x] `fix(detail): 详情图 hero 落位偏小+空白/返回回弹`——`PixivImage.detail` 在 loose Stack 里取解码像素固有尺寸：hero 阶段卡片宽度解码 → 控件只有 ~300px 缩在左上、右下空白，detail 档落地后「突然缩放占回」；返回飞行的源矩形同理偏小 → 落位回弹。修法：detail 变体隐式 `width: double.infinity`，盒子始终全宽+按已解码比例出高，与解码档位解耦（PixEz 同款：显式屏宽）
- [x] `fix(feed): 快滑回看图片大片空白`——预取被 `recommendDeferredLoadingForContext` 拦截且 watermark 只前进：fling 中整窗跳过预取、回滚方向无预取 → 逐出/未解码的槽位裸露占位色。修法：deferred 时仍发小批量批次（解码在 IO 管线不占帧），窗口覆盖回滚方向
- [x] `chore(09-18): acceptance-fixes journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test test/action_queue_test.dart test/action_replay_test.dart` + search/settings/detail/feed 相关测试
- `flutter test`（全量）
- `git diff --check`

## 回滚点

每勾独立可 revert。详情图改动只调一个变体的默认宽度参数；预取改动只放开 deferred 分支的批量上限。

## 风险

- `isConnectivityError` 扩分类后，原本「可见失败」的证书类错误仍必须可见——certificateMismatch 显式排除，有测试锁定。
- `width: double.infinity` 的 detail 图：真实比例与 API 估计值不一致的多图页会在解码完成时跳一次盒子（现状是「小图→跳大」，改后是「估计盒→比例修正」），多图页 meta 缺宽高时接受。
- 预取放开 deferred 分支：批次上限 4 且每帧最多一批，fling 中解码占用的是 IO 线程不是帧预算。
