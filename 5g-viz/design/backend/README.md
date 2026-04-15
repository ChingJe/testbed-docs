# Backend

本目錄收錄 `5g-viz` 後端行為的 canonical 文件，聚焦於事件收集、解析、狀態管理、API 與 metrics。

主要文件：

- [collector.md](collector.md)：live 模式下的 SSH log 收集器、queue contract 與 reconnect 行為
- [parser.md](parser.md)：logrus log 解析流程、rule registry 與事件產生方式
- [state.md](state.md)：`state_snapshot` 的資料模型，以及它如何重用 `event_reactions`
- [api.md](api.md)：HTTP API、WebSocket、session 查詢與 replay 控制介面
- [metrics.md](metrics.md)：live metrics、replay backfill 與 pseudo-live remote write
- [profiles.md](profiles.md)：profile `.env`、`topology.yaml` 與它們對 backend 的影響
