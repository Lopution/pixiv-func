# 基线与 Windows 侧前置条件

## HEAD 基线（task/09-11-windows-desktop @ 84d120f）

- `flutter analyze --no-pub`：No issues found
- `flutter test`（WSL）：741 通过（ui-redesign 收尾时的全量结果，本任务未改 Dart 行为）
- `git diff --check`：干净
- Flutter SDK：`/opt/flutter-3.47.2`（3.47.x）；Android SDK：`/opt/android-sdk`
- Rust：`~/.cargo`（rustup，cargo/rustc 可用）

## Windows 侧前置条件（构建必须在 Windows 执行，WSL 只能出 Dart/模板）

| 依赖 | 说明 |
| --- | --- |
| Visual Studio 2022 | "Desktop development with C++" 工作负载（MSVC + C++ CMake tools） |
| Windows 10/11 SDK | 随 VS 2022 工作负载安装；`flutter doctor` 须检出 Windows 支持 |
| Flutter 3.47.x（Windows 侧） | 与本机 pin 版本一致；`flutter config --enable-windows-desktop` |
| rustup + `x86_64-pc-windows-msvc` | rhttp 的 cargokit 走 rust-toolchain.toml；本任务在该文件追加 target |
| WebView2 Runtime | Win11 预装；缺失时登录/以图搜图页须显式提示（验收项） |
| Clash 代理 | 本机环境 `127.0.0.1:7897`；验收时确认 rhttp 是否跟随系统代理，不预设 |

## 验收命令（Windows 侧）

```powershell
flutter doctor -v          # VS C++ 工作负载 + Windows 支持检出
rustup target list --installed   # 含 x86_64-pc-windows-msvc
flutter build windows --release
.\build\windows\x64\runner\Release\pixiv_func.exe
```
