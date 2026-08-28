# 开源项目筛选记录

## 快照与方法

本次筛选快照日期为 2026-08-27。候选仓库先通过 GitHub 仓库元数据确认默认分支、许可证、近期推送和规模，再克隆到临时目录并固定到下表 commit；结论来自该 commit 的实际源码与测试，不以 README、star 数或搜索摘要代替源码证据。

GitHub 的 `pushed_at` 只是仓库级活跃度信号，不代表每一项功能都在近期维护。所谓“优秀”在本审查中表示项目具有较强覆盖面、活跃度或某一成熟子系统，不表示其所有安全和架构决策都适合 Replica v1。

| 项目 | 固定版本 | 许可证 | 2026-08-27 观测值 | 选择理由 | 主要用途 |
|---|---|---|---|---|---|
| [pixez-flutter](https://github.com/Notsfsssf/pixez-flutter/tree/837f95836ee674ecfa996c2603e20bf9197ff9ee) | `837f95836ee674ecfa996c2603e20bf9197ff9ee` | GPL-3.0 | 12,687 stars / 473 forks；2026-08-26 推送 | 同为 Flutter，覆盖多账号、ECH/兼容网络、Ugoira、Widgets、Updater 等大量功能 | 验证 Flutter/native 与网络模式边界；采用 ECH capability gate，拒绝其关证书/SNI、固定 IP、硬编码秘密、吞错、页面私有状态和整文件内存路径 |
| [Pixiv-Shaft](https://github.com/CeuiLiSA/Pixiv-Shaft/tree/7e3e1a8cf3322503f727f665d8a67762387ea69f) | `7e3e1a8cf3322503f727f665d8a67762387ea69f` | GPL-2.0 | 7,695 stars / 243 forks；2026-08-27 推送 | Android 客户端中对 failure classification、无代理网络诊断、feed、写操作队列、single-flight、Widgets 和反向搜图的失败现场记录最完整 | 主要正向参考 failure taxonomy/诊断与状态机；拒绝 trust-all、SNI hack、固定 IP/反代，只借鉴设计与测试，不复制 GPL 源码 |
| [Pix-EzViewer](https://github.com/ultranity/Pix-EzViewer/tree/96a4f4b42df82c679d43c5da9283537ce7531590) | `96a4f4b42df82c679d43c5da9283537ce7531590` | MIT | 1,246 stars / 40 forks；2026-07-04 推送 | 较新的 Kotlin/Jetpack 客户端，Novel 标记解析与 DNS/SNI 失败边界具有独立研究价值 | 采用纯解析器、分块和 host-scoped DoH/TTL 思想；拒绝 HTML fallback、证书关闭、SNI 替换、固定 IP、全量 Bitmap 和静态 PKCE 等路径 |
| [Pixeval](https://github.com/Pixeval/Pixeval/tree/d1b9cafe48ab6b9817e92feb8aa9f1238d698050) | `d1b9cafe48ab6b9817e92feb8aa9f1238d698050` | GPL-3.0 | 3,121 stars / 209 forks；2026-08-25 推送 | 虽为桌面客户端，但下载 task-group、临时文件提交、历史恢复和 typed search form 较成熟 | 补充多页下载聚合状态、文件所有权、恢复和参数验证模式 |
| [PixivBiu](https://github.com/txperl/PixivBiu/tree/1628d9868d70a5e5934d7536f82331f9178d170f) | `1628d9868d70a5e5934d7536f82331f9178d170f` | MIT | 1,441 stars / 79 forks；2026-08-18 推送 | 下载状态持久化、Ugoira 输入约束和签名更新源最完整 | 主要用于媒体流水线、进程恢复与 updater 供应链加固 |

## 采用边界

- GPL-2.0/GPL-3.0 项目仅作为架构、失败模式和测试设计参考；本次没有复制其代码、资源或文本实现。
- MIT 项目即使允许复用，本次也只做规划研究；未来若确需移植具体代码，必须单独记录版权头、来源 commit 和修改说明。
- 任何仓库中的 OAuth client secret、固定 IP、证书绕过、BODY 日志、密码存储或历史网络常量均视为不可信实现事实，不能成为 Replica v1 的常量来源。
- Pixiv API、OAuth、Live、Android 平台与第三方反向搜图服务仍需在对应任务开始时重新核验；固定 commit 只能证明审查时的开源实现。

## 未选为主样本的原因

本次没有用“仓库数量”代替覆盖质量。五个样本已经互补覆盖 Flutter、Android、桌面成熟下载器和 Go 后端型媒体/Updater；继续加入只复制旧 Pixiv API 封装、长期无维护或没有可定位源码测试的仓库，边际价值很低，也会放大过时常量被误采纳的风险。
