# Backend

> 注意：本子目錄已先重寫 `api.md`，但其他文件仍混有 pre-refactor 背景。現況請優先對照 `api.md`、`5g-viz/README.md` 與 `guides/*`。

本目錄收錄 `5g-viz` 後端行為的技術文件，聚焦於事件收集、解析、狀態管理、API 與 metrics。

適合的情況包括：

- 需要查 collector、parser、state、API、metrics 的實際責任邊界
- 需要確認 `state_snapshot`、replay backfill 與 Prometheus 寫入路徑

若目前需求是先理解畫面上的效果，先讀：

- [`../../guides/ui-workflows/`](../../guides/ui-workflows/README.md)
- [`../../guides/mental-model/`](../../guides/mental-model/README.md)

主要文件：

- [api.md](api.md)：目前 HTTP API、WebSocket、Grafana proxy 與 session 查詢介面
- [profiles.md](profiles.md)：目前 `config.yaml` / `topology.yaml` profile 模型
- [collector.md](collector.md)：SSH log 收集器；仍含部分 historical wiring 描述
- [parser.md](parser.md)：logrus log 解析流程；事件語意仍有效
- [state.md](state.md)：`state_snapshot` 模型；模組路徑敘述略帶 historical 背景
- [metrics.md](metrics.md)：metrics / replay backfill；仍保留較多 historical remote-write 模型背景
