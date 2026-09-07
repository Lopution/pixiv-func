# 技术设计：SauceNAO 反向搜图 transport

设计基线：HEAD `409df51`。输入校验、临时文件所有权、状态机、结果映射和外链安全已经存在，
本 child 只补真实 transport 与可诊断失败路径。

## 1. 前置事实与边界

- 必须先取得 SauceNAO 当前匿名调用、额度和限流规则的可核验资料；若匿名调用已关闭，停止
  实现并回到用户重新决定 D1，不把“让用户填写 key”当作等价替代。
- provider 只接收 `OwnedReverseImageInput` 指向的临时文件；不得把完整图片重新读成长期内存
  缓冲，也不得发送 Pixiv token、账号 ID、设备标识或反向搜图历史。
- 不新增 provider 注册表、插件发现机制或备用 mirror。沿用现有
  `ReverseImageProvider` 接口和 `ReverseImageResultMapper.fromSauceNaoJson`。

## 2. WebView transport 合同

新增受控 SauceNAO WebView flow，HTTP client、端点、时钟和 multipart 工厂均可注入测试。生产
端点、字段名和匿名参数只能在 research 复核后写入，不在代码中硬编码未验证的额度承诺。

请求流程：

1. 检查 `CancelToken` 和输入文件状态；
2. 以 multipart/file streaming 方式提交文件，使用输入的 MIME，不复制全图；
3. 在受控 WebView 中加载结果 HTML，限制导航和资源边界；
4. 拦截 Pixiv 作品/作者链接交给现有原生路由，其它链接维持页面或外部浏览器；
5. WebView/HTTP 错误、challenge、限流、超时和取消返回可见失败，不返回空成功。

错误映射：

| 情况 | outcome | 约束 |
|---|---|---|
| 2xx HTML 结果页 | webview ready | 页面由 SauceNAO 渲染，不在首版自建 HTML parser |
| HTML challenge/错误页 | `malformedResponse` / `providerUnavailable` | 不得当作无匹配 |
| 429/明确限流字段 | `rateLimited` | 解析非负 retry-after，无法解析则仍可见且可重试 |
| timeout/socket/取消 | `network` / `cancelled` | 取消沿用现有 token，网络错误标记 retryable |
| provider 明确拒绝匿名 | `providerUnavailable` | 记录事实，不偷偷切 provider |
| mapper `unsafeResultUrl` | 原 code | 不放宽外链安全 |

## 3. 数据流与 UI

`ReverseImageSearchController` 继续拥有 generation、状态和 exactly-once 输入释放；WebView flow
不删除临时文件。UI 保留“无匹配”与“请求失败”两种终态，Pixiv 作品/作者链接走站内路由，其它
链接交给 `ReverseImageExternalLauncher`。限流错误展示等待时间和重试动作，不自动循环重试。

## 4. 文件责任

| 责任 | 主要文件 |
|---|---|
| WebView/HTTP | `lib/core/reverse_image/reverse_image_provider.dart` 或同目录新文件 |
| 流程所有权 | `lib/core/reverse_image/reverse_image_controller.dart`（仅必要适配） |
| UI/路由 | `lib/features/search/reverse_image_search_page.dart` |
| 平台输入 | `image_input.dart`、`reverse_image_platform.dart`（不改变权限边界） |
| 测试 | `test/reverse_image_search_test.dart`、`reverse_image_search_page_test.dart`，新增 transport fixture |
| 研究 | `research/anonymous-policy.md`、真实响应/限流记录（脱敏，不保存图片、token、key） |

## 5. 风险与回滚

- SauceNAO 返回 challenge/错误 HTML 时必须失败可见；首版不自建脆弱 HTML parser。
- multipart 库若只能整包缓冲，不能以方便为由突破内存边界；更换为支持文件流的现有 HTTP
  能力，或停止实现并上报工具限制。
- 只提交 provider 与测试；若真机限流策略发生变化，回滚 transport 阶段，不改产品决策。
