# Backend

> 注意：本子目錄仍混有 pre-refactor 文件，部分內容會提到 replay control API、pseudo-live remote write、profile `.env` 與 `main.py`。現況請先對照 `5g-viz/README.md` 與 `guides/*`。

本目錄收錄 `5g-viz` 後端行為的技術文件，聚焦於事件收集、解析、狀態管理、API 與 metrics。

適合的情況包括：

- 需要查 collector、parser、state、API、metrics 的實際責任邊界
- 需要確認 `state_snapshot`、replay backfill 與 Prometheus 寫入路徑

若目前需求是先理解畫面上的效果，先讀：

- [`../../guides/ui-workflows/`](../../guides/ui-workflows/README.md)
- [`../../guides/mental-model/`](../../guides/mental-model/README.md)

主要文件：

- [collector.md](collector.md)：live 模式下的 SSH log 收集器、queue contract 與 reconnect 行為
- [parser.md](parser.md)：logrus log 解析流程、rule registry 與事件產生方式
- [state.md](state.md)：`state_snapshot` 的資料模型，以及它如何重用 `event_reactions`
- [api.md](api.md)：HTTP API、WebSocket、session 查詢與歷史 replay 介面
- [metrics.md](metrics.md)：live metrics、replay backfill 與歷史 remote-write 模型
- [profiles.md](profiles.md)：歷史 profile `.env` / `topology.yaml` 模型，以及目前 topology config 背景
