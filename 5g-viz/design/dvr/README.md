# DVR

本目錄收錄 DVR 行為的 canonical 文件，聚焦於 session、播放狀態與 replay 流程。

適合的情況包括：

- 需要查 session artifact、DVR 狀態機、replay runtime 與 pseudo-live 細節
- 需要確認 replay 與 live 在 runtime 上的真正差異

若目前需求是先理解操作意義與使用者可觀察到的行為，先讀：

- [`../../ui-workflows/dvr-controls.md`](../../ui-workflows/dvr-controls.md)
- [`../../mental-model/live-vs-replay-data-paths.md`](../../mental-model/live-vs-replay-data-paths.md)

主要文件：

- [overview.md](overview.md)：DVR 的 cross-layer 模式、狀態機與各層責任分工
- [session.md](session.md)：session 錄製、`meta.json` / `events.jsonl` / `topology.yaml` 與查詢介面
- [replay.md](replay.md)：replay 啟動、Prometheus backfill、pseudo-live 與一致性邊界
