# 设置页产品化：网络/质量/下载/过滤

## Goal

让每一个普通用户可见的设置项都真正改变产品行为。当前设置页存在三类问题：能保存但没有
消费者（C9 内容过滤、D6 文件命名）、语义不清（D4 两个 bool）、以及不该暴露给普通用户的
实现细节开关（C16 native intercept、C17 全局 insecureNoSni、C13 单选项图片源）。

覆盖审计编号：D3, D4, D5, D6, C9, C10, C13, C15, C16, C17, U5, U6, U7。

## Requirements

### R1. 内容过滤真正接线（C9）

**决策（2026-09-01，用户拍板）：做实现，不删开关。**

现状核实：`enableLocalBlockR18`、`enableLocalBlockAI`、`blockedTags` 三项设置可持久化，
但 feed 展示层没有任何消费者 —— 只在设置页自身与详情页 tag 展示处被读取。判定依据字段
已齐备：`IllustEntity.isR18`（`x_restrict == 1`）、`isAi`（`illust_ai_type == 2`）、
`tags`（`List<IllustTag>`）。

#### R1.1 过滤规则

三条规则，任一命中即隐藏：

- `enableLocalBlockR18` 开启 且 作品 `isR18`
- `enableLocalBlockAI` 开启 且 作品 `isAi`
- 作品 tags 与 `blockedTags` 有交集

#### R1.2 适用范围

**适用（发现类列表）**：Recommended、Ranking、Search 结果、用户作品列表。
这些是「系统推给用户」的内容，屏蔽诉求成立。

**适用（桌面小组件）**：小组件在锁屏与桌面会被旁人看到，泄露风险高于 app 内。
注意小组件走独立数据路径（`WidgetFeedLoader` 直接 `fetchPage(null)` 取一页后选图，
不经过 feed 状态机），因此过滤规则必须是可被两条路径共同消费的纯谓词。

**不适用**：
- 收藏列表与历史记录 —— 用户自己收藏过、看过的作品被自己的开关隐藏会造成困惑。
- 作品详情页 —— 从搜索、链接、小组件直接点进的单个作品照常展示。详情页已有
  `_RestrictedView` 处理服务端受限作品，不再叠加本地过滤层，避免「点得开但看不了」。

#### R1.3 稀疏分页处理

**决策：自动续拉直到填满一屏。**

一页 30 个作品在高屏蔽率下可能只剩几个，列表短到无法触发滚动，用户会卡在几个作品上
无法加载更多 —— 等于功能坏掉。因此过滤后可见数量低于阈值时，自动继续请求下一页，
直到达到阈值或服务端耗尽。

约束：
- 续拉必须有明确上限（连续请求次数封顶），不允许出现无上限循环请求。
- 达到上限或服务端耗尽后仍不足一屏，按正常「已加载完」状态展示，不显示错误。
- 续拉期间沿用现有的 load-more 指示，不引入新的加载态。

#### R1.4 设置变更即时生效

用户在设置页改变任一过滤开关或屏蔽标签后，返回列表时结果立即反映新设置，
不要求重启 App 或手动下拉刷新。

### R2. 网络设置简化（D3, C15, C16, C17, U5）

- 普通用户页只保留：网络模式（自动 / 仅直连）、网络诊断、高级设置。
- Automatic 是新装默认值，是 Func 的正常标准网络栈，不是「出问题后才开启的兼容模式」。
  普通用户页不出现 DoH / ECH / SNI / 证书校验 / PlatformView 等实现名词。
- **C15**：`AppSettings` 增加 `networkMode`（automatic / directOnly）并持久化。当前该
  选择不落盘，用户手动切到 Direct only 后重启或 provider 重建会退回 Automatic。
  只持久化模式本身，route memory 与探测结果不持久化。
- **C16**：删除 native login WebView intercept 的生产设置与 wiring。首次登录只走稳定
  的普通 WebView。该能力对应的目标已取消，代码不保留为「实验模式」。
- **C17**：删除普通用户可见的全局 `insecureNoSni` 开关。若图片 CDN 的 A/B 证明放松传输
  确有收益，那属于 Automatic 内部的 image-only 策略，不扩大到 API / OAuth，也不作为
  用户开关暴露。
- 高级页可为 power user 保留 DoH endpoint 与 ECH front host 编辑；网络诊断尽量只读，
  展示当前 route 与最近探测结果。

### R3. 浏览质量（D4, U6）

- 预览质量：中图 / 大图 / 原图，默认**中图**。
- 查看质量：中图 / 大图 / 原图，默认**原图**。
- 旧设置迁移：`previewQuality=true` → 大图，`false` → 中图；`scaleQuality=true` → 原图，
  `false` → 大图。
- 不为漫画 / 插画 / 放大查看分别再建一套质量选项。

### R4. 下载保存位置（D5, U7）

- 默认保存到 PixivFunc 相册（MediaStore，默认 `RELATIVE_PATH = Pictures/PixivFunc`），
  对用户显示人类可读名称。
- 用户可改为自定义相册，或改为文件夹模式。
- 文件夹模式使用 Android 系统 SAF 目录选择器并持久化系统返回的 tree URI 权限。
- 不做纯文本路径输入框，不自制文件浏览器，不自行管理 `/storage/emulated/0/...` 权限
  兼容表。
- 保存目录 ListTile 当前没有 `onTap`，属于「能看不能改」，一并修复。

> 跨 child 约束：本项引入的 destination 语义变更必须先于
> `09-01-behavior-correctness-cleanup` 的 C4/C5/C22 落地，后者的 download recovery
> owner 定义依赖最终的 destination 模型。

### R5. 文件命名（D6, U7）

- 现状：`namingRule` 可保存，但 `DownloadRequest` 仍固定生成 `<id>_p<page>.<ext>`，
  设置完全不生效。
- 普通设置页首先展示预设：作品 ID（默认）/ 作者 - 标题 - ID / 标题 - ID / 自定义。
- 自定义模板只支持受限变量：`{artist}` `{title}` `{id}` `{page}` `{ext}`，可选 `{date}`。
- 必须提供：当前模板实时预览、变量说明、非法文件字符自动清理、文件名过长自动裁剪、
  多 P 页码明确。
- 不引入 JavaScript eval、正则规则、条件语法；模板内不允许用 `/` 控制目录 —— 目录
  由 R4 单独负责。

### R6. API 语言跟随 UI 语言（C10）

- `PixivHttpClient` 当前构造参数默认 `languageTag='zh-CN'`，且 provider 未注入 settings
  的 `languageTag`。UI 设为日语 / 英语 / 俄语时，Pixiv API 仍按中文请求。
- client provider 监听 `settings.languageTag` 并传入。不新建 LocaleService。

### R7. 隐藏单选项设置（C13）

`ImageSourceMode` 当前只有 normal 与 i.pximg.net 两个值，其中一个不构成有意义的选择。
在只有一个可选项时直接隐藏该设置，未来出现第二种产品级图片源再恢复。

## Acceptance Criteria

### C9 内容过滤

- [ ] 开启 R18 屏蔽后，Recommended / Ranking / Search / 用户作品列表中不再出现 R18 作品。
- [ ] 开启 AI 屏蔽后，同上四处不再出现 AI 生成作品。
- [ ] 加入屏蔽标签后，同上四处不再出现含该标签的作品。
- [ ] 桌面小组件遵循同一套过滤规则。
- [ ] 收藏列表、历史记录、作品详情页**不受**过滤影响。
- [ ] 高屏蔽率场景下列表仍可持续滚动加载，不出现「短到滚不动」的死局；续拉次数有上限，
      不出现无上限循环请求。
- [ ] 在设置页改变过滤项后返回列表，结果立即反映新设置，无需重启或手动刷新。
- [ ] 真机截图留证到 `research/screenshots/`。

### 其余设置项

- [ ] 网络设置页只剩三项，无实现名词；`networkMode` 重启后保持用户选择。
- [ ] native login intercept 与全局 insecureNoSni 开关在代码与 UI 中均已删除。
- [ ] 预览质量默认中图、查看质量默认原图，旧设置按 R3 规则完成迁移。
- [ ] 下载默认落到 PixivFunc 相册；自定义相册与 SAF 目录均真实生效。
- [ ] 命名预设与自定义模板真实生效，多 P 不覆盖，预览与实际落盘文件名一致。
- [ ] UI 语言切到日语后，抓包确认 API 请求 `Accept-Language` 随之改变。
- [ ] 图片源设置在只有单一可选项时不可见。
- [ ] 每一项普通设置都能指出它改变了哪个真实产品行为；没有消费者的设置一律不留。

## Open Questions

- R4 的「自定义相册」与「SAF 文件夹」在 UI 上如何并列呈现，需在 design 阶段定稿。
- R2 高级页保留哪些 power user 编辑项的最终清单待定。

## Notes

- 上级需求与跨 child 约束见 `../09-01-func-1-0-hardening/prd.md`。
- 审计断言复核见 `../09-01-func-1-0-hardening/research/audit-verification.md`。
- 本 child 属复杂任务，`design.md` 与 `implement.md` 必须在 `task.py start` 前完成。
