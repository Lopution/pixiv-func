# Search API verification

验证时间：2026-08-27（Asia/Shanghai）

## MuMu

- Device：`127.0.0.1:7555`
- Model：`SM_S9280`
- App：`io.github.lopution.pixivfunc`
- APK：`build/app/outputs/flutter-apk/app-debug.apk`

## Observed behavior

在已有登录账号下，使用关键词 `cat` 逐一验证三类搜索：

- `/v1/search/illust`：成功返回并渲染 Illust/Manga 双列结果。
- `/v1/search/novel`：成功返回并渲染 Novel 列表。
- `/v1/search/user`：成功返回并渲染 User 列表及关注按钮。
- `/v1/search/autocomplete`：服务端返回 `HTTP 404`；客户端显示可重试错误态，不生成或缓存伪造建议。
- `/v1/trending-tags/illust`：搜索首页成功返回 trending tags，并渲染双列入口。

此记录只保存 endpoint、状态和 UI 观察，不保存账号、token、Cookie 或搜索响应原文。
