# Risks & Decision Points — W9 onboarding-auth-content-layout

基线 `main@8067b2d`（分支 docs/09-22-ui-leaf-planning，工作树产品代码与 main 一致）。

## 高风险

1. **W1 未合入是硬前置。** W1（09-22-interaction-outcome-correctness）现在仍是 `planning`，连 PRD 都是 TBD；WebView "reopen/reload/return" 的动作语义契约尚不存在。W9 的 WebView 布局统一（draft ③）若先于 W1 动手，只能统一结构、不能定动作；且两包同碰 `login_webview*_page.dart`，**必须等 W1 合入 main 后再做该 stage**（父任务 §3 同文件串行规则）。建议 stage 顺序把 WebView 放最后。
2. **`SelectableText.rich` 内链接手势竞争。** Flutter issue #168864：span 的 TapGestureRecognizer 与长按选择/contextMenuBuilder 在部分版本竞争。后果若发生：点链接变成开始选择。缓解：widget test 只能验证 tap 路径；长按共存必须真机/桌面验证，验证不到就标"未验证"。备用方案：保持正文可选择但链接行降级为段落下方独立链接行（改动 parser 输出，范围变大，不建议首选）。
3. **引导/登录页矮屏溢出是真实回归风险而非理论。** language/theme/login 均为固定 `Column`+`Spacer` 不可滚动；login 还有 `height*.4` 硬高度（L280）。改 shell 后必须用 `tester.view.physicalSize` 跑 320×568 / 横屏 640×320 断言无 overflow——否则只是把溢出从一类设备挪到另一类。

## 中风险

4. **共享 shell 放哪层。** 放 `app/widgets/` 需满足 ≥3 消费者规则（welcome/language/theme/login = 4，达标）但要写清"扩展+接入同 stage"；放 `features/onboarding/widgets/` 则 login 不能复用 → 两份近似实现会撞 W10 "无平行组件"验收。**推荐 app 层一份**。命名避开 layering_test 的私有 widget 命名模式。
5. **`FittedBox(scaleDown)` 去留。** welcome 的双行标题有明确注释（L39-41）说是 locale 稳定锚点；父任务验收矩阵又要求 1.3x 大字号不丢可达性。缩字 vs 换行是观感/可达性权衡，**需叶子 PRD 阶段明确**：建议 welcome 保留（品牌 lockup），language/theme 标题改可换行。
6. **外部浏览器登录不是现成能力。** `/login/callback` + intent_router 通道存在，但 PKCE verifier 只在 `OAuthService._session` 内存里，WebView dispose/detach 即 `discardSession`。PixEz 式"浏览器打开授权页"若要加入需要会话持久化/存续改动——**超出布局包范围**；登录页次级帮助区最多放既有能力（剪贴板导入、帮助文本），不要顺手加"浏览器打开登录"。
7. **desktop WebView 无 widget test 先例。** quality spec 有 InAppWebView fake 模式（L233-248），照做即可，但 desktop 错误卡统一后要补 smoke test；WebView2 缺失态只能在 Windows 验证，标"未验证"即可。

## 低风险 / 决策点（需叶子 PRD 拍板）

8. **Spotlight 正文限宽数值**：建议 680–720dp（≈45-55 中文字/行），最终值需真实文章目测——写成常量并注释依据；不要塞进 AppBreakpoints（内容宽≠断点）。
9. **`SharePayload` 要不要加 `spotlight` factory**：裸构造可用（author='pixivision'）；加 factory 更语义化但是 core 改动。建议裸构造 + 注释；若后续别处分享 pixivision 再加。
10. **新增 l10n key**：`share`（通用）、`openInBrowser`（打开原文）——四语言 arb + `lookup.dart`；`cardActionShare`/`linkCopied` 已存在可复用。
11. **代理提示 dialog 是否做手机 sheet**：默认保持 `showAppDialog` 全断点（语义已一致、零成本）；要 sheet 则页面级分支即可，单消费者不扩共享 owner。
12. **'later' 文案槽**：language/theme 底部的 "稍后可在设置中变更" 是纯文本不是按钮——保持纯文本，勿加假 affordance。
13. **login `getMoreHelp` 死链接**：染色无 recognizer（L332-339）——删除或改成真动作，属本包必须处理的"可见但不可点"。
14. **mobile signup 模式无进度条**（delegate 无 onProgress，L63-70）vs desktop 有——统一为都显示。
15. **错误文案带不带 URL**：desktop 带、mobile 不带；建议统一为带 host、不带完整 query（卡片宽度有限）。
16. **`_StartupError` 顺手修**：startup_gate 属本包，retry 从 TextButton 升 FilledButton + 限宽是低成本一致性收益；可选。

## 不能做的事（边界复述）

- 不改 parser/OAuth/intent 语义；不加 SelectionArea；不自造断点常量；不新建 SnackBar/overlay 平行入口；不动 W3/W5/W6/W8 文件；不写 git（分支/提交由实现阶段按 AGENTS.md 流程走）。
