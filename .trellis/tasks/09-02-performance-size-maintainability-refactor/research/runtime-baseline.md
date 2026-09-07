# 运行时性能基线（协议与记录）

用途：09-02 R9 的前提。C 开工前在真机采集一次，C 完成后同机型同脚本复测一次；所有"更省/更快"的结论
必须引用本文件的数字。模拟器没有可用的 Pixiv 网络与真实账号，agent 不能代测（见 09-01 parent 2026-09-01 决定）。

## 协议

### 构建

```bash
export PATH="/opt/flutter-3.47.0/bin:$PATH"     # A 完成后为 3.47.2
flutter build apk --profile --flavor fdroid --target-platform android-arm64   # B3 之后改为 --split-per-abi
adb install -r build/app/outputs/flutter-apk/app-fdroid-profile.apk
```

同一账号登录，关闭其它后台应用，Wi-Fi 固定为同一网络；记录机型、Android 版本、网络路线（`NetworkAccessPolicy`
诊断页显示的当前 route）。

### 指标

| # | 指标 | 采集方法 | 取值 |
|---|---|---|---|
| 1 | 冷启动 TotalTime | `adb shell am force-stop io.github.lopution.pixivfunc` → `adb shell am start -W -n io.github.lopution.pixivfunc/.MainActivity`，读 `TotalTime` | 5 次中位数（ms） |
| 2 | 首帧 | `flutter run --profile --flavor fdroid --trace-startup`，读 `build/start_up_info.json` 的 `timeToFirstFrameMicros` | 5 次中位数（ms） |
| 3 | 滚动 jank | DevTools Performance 录制推荐页匀速下滑 60 秒（手指匀速或 `adb shell input swipe` 循环），统计 build 或 raster > 16 ms 的帧占比 | 百分比 |
| 4 | 内存 | `adb shell dumpsys meminfo io.github.lopution.pixivfunc` 的 `TOTAL PSS` 与 `Graphics`，分别在冷启动后 10 秒、推荐页浏览 50 张后、200 张后 | MB ×3 |
| 5 | 详情打开延迟 | 点击卡片到详情首图完成显示（DevTools timeline 中 `PixivImage` 完成事件，或 60fps 屏幕录制计帧） | 10 次中位数（ms） |
| 6 | 重复请求 | 开启网络诊断日志，执行脚本会话：推荐 → 打开第 1 张详情 → 作者页 → 返回 → 返回，重复 5 次；统计相同 URL 出现 ≥2 次的请求数与总请求数 | 重复数 / 总数 |
| 7 | 图片缓存 | `adb shell run-as io.github.lopution.pixivfunc du -sk cache/libCachedImageData`（或 `flutter_cache_manager` 实际目录）与对象数 | MB / 个 |

## 基线（C 前）

| # | 指标 | 值 | 机型 / Android / 路线 | 日期 |
|---|---|---|---|---|
| 1 | 冷启动 TotalTime | 待测 | | |
| 2 | 首帧 | 待测 | | |
| 3 | 滚动 jank | 待测 | | |
| 4 | PSS 冷启动 / 50 张 / 200 张 | 待测 | | |
| 5 | 详情打开延迟 | 待测 | | |
| 6 | 重复请求 | 待测 | | |
| 7 | 图片缓存 | 待测 | | |

## 复测（C 后）

| # | 指标 | 值 | 相对基线 | 结论（阈值：首帧 +10%、PSS +10%、jank +2 pp 为回归） |
|---|---|---|---|---|
| 1 | 冷启动 TotalTime | 待测 | | |
| 2 | 首帧 | 待测 | | |
| 3 | 滚动 jank | 待测 | | |
| 4 | PSS 冷启动 / 50 张 / 200 张 | 待测 | | |
| 5 | 详情打开延迟 | 待测 | | |
| 6 | 重复请求 | 待测 | | |
| 7 | 图片缓存 | 待测 | | |

## 已知的代码事实（2026-09-07，供解释数据）

- `lib/app/pixiv_image.dart` 没有 `memCacheWidth/Height`/`ResizeImage`；全仓库只有 `reverse_image_search_page.dart:227`
  用了 `cacheWidth: 1024`。瀑布流按原始像素 decode → 指标 4 的主要可动因子。
- `illustStoreProvider` 是普通 `Provider<IllustStore>`，不会因实体更新触发卡片重建；`bookmark_switch_button.dart:181`
  已按键 `select`。重建问题不能预设，靠指标 3 与 DevTools rebuild 统计判断。
- 首帧前只等 settings 与账号读取（`app.dart:61`、`startup_gate.dart:26`）；`Rhttp.init`、网络 warmUp、
  widget coordinator 均 `unawaited` → 指标 1/2 的可动因子只有这两次读取。
- `IllustDetailController` 每次打开都请求 `/v1/illust/detail` 并 `mergeAll`（`illust_detail_controller.dart:63`）——
  这是刻意取全量字段还是重复，由指标 6 判断。
- 下载恢复记录 `setStringList` 整表重写（`download_recovery.dart:400–422`）；history 有 3 个索引
  （`history_database.dart:76,80,96`）。
