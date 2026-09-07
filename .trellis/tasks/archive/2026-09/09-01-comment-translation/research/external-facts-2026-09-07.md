# 外部事实复核（R3，2026-09-07）

PRD R3 要求实现前复核三条会变化的外部事实。规划阶段没有 WebSearch；本次由编排会话用
WebSearch + curl 完成。结论按「是否推翻 D2 首选百度」标注。

## 1. 百度翻译开放平台：额度与实名认证

来源：`https://fanyi-api.baidu.com/access/0`（页脚 2026 Baidu，开通页实时文案）。

| 版本 | 认证 | 每月免费字符 | 超出计费 | QPS |
|---|---|---|---|---|
| 标准版 | 无需认证 | **5 万** | 49 元 / 百万字符 | **1** |
| 高级版 | 个人认证（真实姓名 + 身份证号） | **100 万** | 49 元 / 百万字符 | 10 |
| 尊享版 | 企业认证 | — | — | 100 |

- 与复核文档 G3 的数字**一致**：D2 采信的「100 万字符/月」是高级版，需要个人实名认证；
  未认证标准版 5 万字符/月 + QPS 1。R2 的引导文案（前置告知实名与额度）**维持**。
- 网上还能搜到一份旧 PDF（`fanyiapp.cdn.bcebos.com/.../通用翻译API服务升级说明.pdf`）称
  标准版「不限字符量」、高级版 200 万——那是历史文档，与当前开通页冲突，**以开通页为准**。
- `cloud.baidu.com` 的「机器翻译-通用版」是百度智能云的另一套产品（未认证无免费额度，
  个人认证领 500 万测试字符），与 app 现用的 `fanyi-api.baidu.com/api/trans/vip/translate`
  不是同一个接口，不纳入本 child。

结论：不推翻 D2。

## 2. 腾讯云机器翻译 `TextTranslate`

来源：`https://cloud.tencent.com/announce/detail/2274`（2026-04-29 公告）、
`https://cloud.tencent.cn/document/product/551/15619`（接口文档，域名
`tmt.tencentcloudapi.com`，Action `TextTranslate`，Version `2018-03-21`，默认 5 次/秒）。

- **审计「腾讯云 TextTranslate 已下线」的结论有误。** 2026-04-29 公告退市的是
  文件翻译、图片翻译、语音翻译、语种识别（2026-07-31 停售，2027-07-31 停服）；
  文本翻译 `TextTranslate` / `TextTranslateBatch` 不在退市列表，接口文档仍在维护
  （错误码含 `FailedOperation.NoFreeAmount` 「本月免费额度已用完」，说明仍有免费额度）。
- 腾讯云账号本身需要实名认证才能开通任何产品；因此「免实名可用」上腾讯并不优于百度标准版。
- 签名为 TC3-HMAC-SHA256，比百度的 MD5 sign 复杂，需要 SecretId/SecretKey 两项凭据。

结论：**腾讯重新成为候选**，但按 PRD R3「若复核推翻『首选百度』，回到用户重新决策 D2，
不自行改选」——本 child 不加腾讯 provider，把这一条交回用户决策。当前四条路径
（关闭 / Google / 百度 / 通用 LLM）不受影响。

## 3. Google gtx 端点

探测（本机经数据中心出口）：

```
curl 'https://translate.googleapis.com/translate_a/single?client=gtx&sl=ja&tl=zh-CN&dt=t&q=...'
→ HTTP 429，Google 「Sorry...」拦截页（两种 UA 相同）
```

- 端点**仍存在**（不是 404 / 下线），但对共享 / 数据中心出口返回 429。这是 gtx 的已知行为：
  住宅网络通常可用，聚合出口被限流。
- 代码 `GoogleCommentTranslationService`（`lib/core/comments/comment_translation.dart`）把
  429 映射为 `CommentTranslationFailureKind.rateLimited`（可见错误，不静默降级），与 R7 一致。
- 兼容项保留的理由（D2：给之前明确保存 Google 的用户）仍成立；不作为新用户默认。

结论：不推翻 D2；gtx 只作兼容路径。

## 对 check 的影响

- R2 文案：核对设置页百度引导是否写明「标准版 5 万字符/月、QPS 1；100 万需个人实名认证」。
- R7：百度额度错误码（54003 频率 / 54004 余额不足 / 52003 认证失败）与 Google 429 的映射需分别可见。
- D2 重决策（是否加腾讯）留给用户，不阻塞本 child 归档。
