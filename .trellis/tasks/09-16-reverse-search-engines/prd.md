# 多引擎以图搜图

## Goal

SauceNAO 之外接入 TinEye / IQDB / Ascii2D。

## Requirements

- `core/reverse_image/` provider 抽象扩展；新增三个 web 端点 provider（multipart 上传/URL 提交 → 结果链接）
- 引擎选择 UI（详情页/查看器入口复用现有 SauceNAO 入口）
- 失败引擎独立报错不阻塞；结果统一跳浏览器/WebView 策略沿用 `reverse_image_external`

## Acceptance Criteria

- [ ] 四引擎可选可切；单引擎失败不影响其他
- [ ] provider 单测（构造请求/解析结果/错误映射）

## References

- Shaft：以图搜图四引擎入口（README 功能表）
- 本仓：`lib/core/reverse_image/`、`features/search/reverse_image_search_page.dart`

## Dependencies

无。
