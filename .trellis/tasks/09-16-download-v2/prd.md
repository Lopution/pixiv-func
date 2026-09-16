# 下载体系增强

## Goal

对齐 Shaft 下载体验：字节级断点续传、批量下载、caption 导出、命名模板扩展。

## Requirements

- HTTP `Range` 续传：`DownloadSink` 增加 `resumeFrom`；部分文件 `.part` 语义与恢复（父 design §3）；禁止破坏 MediaStore/SAF 语义
- 批量下载：整页套图、整作者作品入队；批量任务可暂停/取消/观察
- caption 导出（同名 .txt 随图保存，可选开关）
- 命名模板变量扩展（参考 Shaft template：系列/日期/分辨率等），保持"无 / 无正则"安全约束
- 复用 DownloadManager ownership/recovery/revision 契约

## Acceptance Criteria

- [ ] 中断后可从已下载字节续传；恢复记录可观测
- [ ] 批量任务在任务页可管理；caption 随文件落盘
- [ ] sink/transport/recovery 测试覆盖续传与取消

## References

- Shaft：`download/`（backend/template/importer/aria2 目录可看不照搬）
- 本仓：`lib/core/download/`、spec `backend/release-artifacts` 相关契约

## Dependencies

无。
