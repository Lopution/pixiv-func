# 技术设计：移动端 UI 动画与个人页问题

## 边界

本任务只调整作品卡片/详情路由的 Hero 作用域与首帧输入、详情简介默认状态，以及当前账号资料页的设置入口。现有 `ReplicaPageRoute` 的右侧滑入/返回曲线、Riverpod 共享实体 store、Pixiv 图片缓存和网络 API 均保持不变。

## Hero 数据流

1. 每个可打开作品详情的列表调用方提供稳定的 `heroScope`（推荐、排行、新作、搜索、个人页分别拥有自己的 scope）。
2. `IllustCard` 用 `heroScope + entity.id` 生成唯一 tag，并在点击时将同一个 `heroScope` 与 `entity` 快照传给 `IllustDetailPage`。
3. `IllustDetailPage` 的首帧优先使用 `illustStoreProvider` 中的实体，缺失时使用卡片传入的 `initialEntity`；详情图片使用同一 scope 生成目标 Hero。
4. 个人页列表使用包含用户和 feed 筛选器的 `profile:<userId>:<kind>:<workType>:<restrict>` scope，因此个人页到详情仍可飞行，但个人页和底层推荐/排行/搜索列表不会因为相同作品 ID 匹配 Hero。
5. 非卡片入口（历史记录、深链、反向搜图等）继续使用默认 scope/无来源快照；没有匹配源 Hero 时由 Flutter 正常执行普通滑入，不添加全局禁用或伪造源节点。

## 首帧加载

- `IllustDetailPage` 增加可选 `initialEntity` 和 `heroScope` 参数。
- `async.when(loading:)` 与 app bar 的实体读取统一采用“store snapshot 优先、initialEntity 兜底”。这样 API Future 尚未完成时，详情结构、作者信息和 Hero 目标已经存在。
- 不在点击回调中等待网络请求，避免把导航响应时间绑定到 API；图片由已有 `PixivImage`/`CachedNetworkImage` 在首帧开始加载，命中缓存时立即显示，未命中时保留现有占位行为。

## 详情简介

- 删除 `_showCaption` 状态、切换回调和“简介”展开控件。
- 非空 caption 直接渲染现有 `_CaptionRichText`；空 caption 不插入简介区块。
- 保持已有 HTML typed-span 解析和站内/站外链接行为，不复制解析逻辑。

## 设置入口

- `UserPage._buildProfile` 不再为 `onSettings == null` 创建 `SettingsPage` fallback。
- 头部 delegate 和 `UserPage` 不再接收或渲染 `onSettings`；设置页账户卡片进入的 `MePage` 与其它个人资料入口均不显示设置图标。

## 兼容性与风险

- Hero tag 只由代码定义的 scope 与整数作品 ID组成，不引入用户输入到路由标识。
- 详情 API 仍会刷新并通过 `IllustStore` 合并；首帧快照只是渲染输入，不改变数据所有权。
- 不新增缓存、预取队列、route observer 或跨页面状态；这样可以直接用 widget test 覆盖可观察的首帧与 tag 关系。
- 真实图片是否已在设备缓存中无法由离线测试保证，设备验收需记录首次未缓存作品和返回路径截图。

## 回滚

如 Hero 作用域回归导致已有入口异常，可单独回退 `IllustCard`/`IllustDetailPage` 的 scope 参数，保留简介与设置入口修复；不涉及网络或持久化迁移。
