# Windows 桌面验收证据与遗留限制

## 已完成（Linux/WSL 侧可验证）

| 项 | 结果 | 证据 |
| --- | --- | --- |
| `dart format --set-exit-if-changed lib test` | 通过 | 阶段 4 质量门 |
| `flutter analyze --no-pub` | No issues found | 阶段 4 质量门 |
| `flutter test` 全量 | 762 通过（743 + oauth/tls 单独 19） | oauth/tls 在 loopback 环境下需单独跑，属已知环境性慢，非回归 |
| `git diff --check` | 干净 | 阶段 4 质量门 |
| 平台能力矩阵 | 16 测试通过 | `test/windows_capability_test.dart` |
| 登录拦截判定纯函数 | 5 测试通过 | `test/login_navigation_decision_test.dart` |
| Android 语义零回归 | 全量套件通过 | 上述 |

## Windows 侧人工验收清单（必须在真实 Windows 执行）

- [ ] `flutter doctor -v`：VS2022 C++ 工作负载 + Windows SDK 检出
- [ ] `flutter build windows --release` 成功（CI `windows-build` job 可代证编译链接；运行验收仍需本机）
- [ ] 启动 `build\windows\x64\runner\Release\pixiv_func.exe`：标题 `Pixiv-func`、初始 1280×720、最小尺寸生效
- [ ] 登录：`/login/web` 走 `LoginWebViewDesktopPage`（WebView2），`pixiv://account?code=` 拦截 → exchange → 回到 Home
- [ ] 注册入口（`create=true`）同样完成
- [ ] 代理下真实数据：推荐/排行/详情在 Clash `127.0.0.1:7897` 环境加载
- [ ] 样板链路：详情 → 大图 → 收藏 → 下载
- [ ] 下载落盘：默认目录 `<Downloads>/PixivFunc`、`.part` 暂存、完成物化、取消清理
- [ ] 账号迁移：导出/导入二维码 + 剪贴板兜底在 Windows 端工作
- [ ] 窗口缩放、横宽比、NavigationRail 切换正常
- [ ] 卸载/缺失 WebView2 Runtime 时登录页出现安装指引（`getAvailableVersion()` 探测路径）
- [ ] 以图搜图：file_selector 选图 → SauceNAO 结果页（InAppWebView）→ 外链进系统浏览器

## 已知限制 / 不声明完成项

- **未在真实 Windows 构建/运行**：当前环境为 WSL2/Linux，`flutter build windows` 与 WebView2 运行时行为未经验证；CI `windows-build` job 提供编译级证据，运行验收见上表。
- **rhttp 系统代理**：Clash 环境下 rhttp 是否跟随系统代理未实测（design §6 要求先实测再决定是否加设置项）。
- **`flutter_inappwebview_windows` 回跳兜底**：`shouldOverrideUrlLoading` 在 WebView2 上可能漏过 302 链，已加 `onUpdateVisitedHistory` 二次拦截兜底；真实 Pixiv 回跳可靠性仍待 Windows 实测（design §9 停止条件适用）。
- **WebView2 缺失探测**：`getAvailableVersion()` 在 Windows 侧返回 `null` 的分支只到 UI 层；真实缺失环境的表现未验证。
- **后台 widget / 推送 / SAF**：桌面端均为显式降级（inert intent source、MissingPlugin 吞掉、desktop file sink），未做桌面等价功能。
