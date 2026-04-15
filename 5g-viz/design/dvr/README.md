# DVR

本目錄收錄 DVR 行為的 canonical 文件，聚焦於 session、播放狀態與 replay 流程。

主要文件：

- [overview.md](overview.md)：DVR 的 cross-layer 模式、狀態機與各層責任分工
- [session.md](session.md)：session 錄製、`meta.json` / `events.jsonl` / `topology.yaml` 與查詢介面
- [replay.md](replay.md)：replay 啟動、Prometheus backfill、pseudo-live 與一致性邊界
