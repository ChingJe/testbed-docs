# 5g-viz Data Flow

本文描述目前 runtime 下 `5g-viz` 的兩條主要資料路徑：

- live：從遠端 VM log 即時進入 topology、event log 與 Grafana
- replay：從磁碟上的 session 還原事件，並用原始 session metrics 驅動 Grafana

若要看模組分工，請先讀：

- [`system.md`](./system.md)
- [`../backend/api.md`](../backend/api.md)
- [`../backend/metrics.md`](../backend/metrics.md)
- [`../frontend/events-and-dvr.md`](../frontend/events-and-dvr.md)

## 1. 兩條資料面

目前系統始終分成兩個彼此相關、但不相同的資料面：

1. event / topology 資料面
   - 來源是 parser 產生的結構化事件
   - 影響 event log、topology 動畫、DVR timeline、state snapshot
2. metrics / chart 資料面
   - 來源是部分 event type 再投影成 Prometheus metrics
   - 影響 Grafana iframe 內的 traffic、deviation、annotation

這個分界在 replay 特別重要：拓樸播放與圖表查詢會共享同一份 session，但實際更新方式不同。

## 2. Live：端到端資料流

### 1. Collector tail 遠端 log

`services/collector.py` 依 `profiles/<profile>/config.yaml` 內的 `collector.sources` 連到遠端 VM，對每個 log source 執行 `tail -F`，把每一行推進共用 `asyncio.Queue`。

推進 queue 的資料格式仍然很薄：

```json
{
  "source": "free5gc" | "nwdaf",
  "line": "<raw log line>"
}
```

### 2. Queue consumer 解析成事件

live 模式下，`backend.app` 內的 queue consumer loop 會：

1. 從 queue 取出一行 log
2. 呼叫 `replay/parser.py` 做基礎欄位解析
3. 依 `rules/*` 產生結構化 event
4. 對 event 做三類 side effects：
   - append 到記憶體 `_events`
   - 寫入目前 session 的 `events.jsonl`
   - 更新 state、metrics，並透過 WebSocket 推到前端

### 3. Event 更新 state 與前端

live 模式下，event 進入後端後會先進 `runtime/state.py`，再經 `/ws` 推給前端。

前端 `frontend/events.js` 會：

1. 建立 `/ws` 連線
2. 先接收一份 `state_snapshot`
3. 再持續接收新事件

`frontend/topology.js` 依 event reactions 更新：

- node class
- flash edge
- pulse
- paused/scrubbed static snapshot

### 4. Event 投影成 metrics

只有部分 event type 會進 Prometheus，例如：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `retrain_done`
- `model_swap`

這些 handler 定義在 `rules/nwdaf.py`，並由 `rules/__init__.py` 匯總成 metric registry。

live 模式的路徑是：

```text
event
  -> metric handler
  -> prometheus_client exporter state
  -> /metrics
  -> Prometheus scrape
  -> Grafana dashboard
```

## 3. Replay：啟動資料流

### 1. CLI 決定 replay policy

目前 replay 由：

```bash
uv run run.py replay sessions/<session_id> --profile <profile> --backfill=auto|overwrite|skip
```

啟動。

在啟動 app 前，`run.py` 會：

1. 載入 profile `config.yaml`
2. 檢查 Prometheus 是否已常駐並可 reload
3. 透過 `services/replay_session_service.py` 檢查：
   - 本地 session 是否存在
   - Prometheus 中是否已有該 session
4. 依 `--backfill` policy 決定：
   - `auto`: 已存在則 reuse，否則 backfill
   - `overwrite`: 先刪 session series，再 backfill
   - `skip`: 不寫 Prometheus，僅播放 topology / event

### 2. Lifespan 載入 session

replay 模式下，`backend.app` 啟動後會：

1. 載入 `meta.json`
2. 載入 `events.jsonl`
3. 載入 session 自帶的 `topology.yaml`
4. 依事件重建 `runtime/state.py`
5. 視 backfill policy 決定是否把原始 session metrics 寫入 Prometheus

replay 不會啟動：

- SSH collector
- queue consumer
- live WebSocket event 生產

前端改走：

- `GET /api/session-info`
- `GET /api/events`

## 4. Replay：前端資料流

### 1. Session 事件一次載入本地

replay 模式下，前端會先把整份 `_events` 載入記憶體，建立自己的 timeline。

這條資料面之後完全由前端控制：

- `Play`
- `Pause`
- `Scrub`
- keyboard step scrub
- residual edge/pulse continuation on resume

也就是說，replay topology 並不是由後端持續 push，而是前端對本地事件集合重新取樣與重播。

### 2. Grafana 查同一個原始 session

目前 replay 不再有 pseudo-live session，也沒有 `MetricPlayer`。

Grafana 在 replay 下只有一個 canonical session：

- `var-session=<original_session_id>`

差別只在時間窗：

- paused / scrubbed：用絕對 `from/to`
- playing：用 historical relative `from=now-(offset+window)`、`to=now-offset`

因此 replay 播放期間：

- topology 是前端本地重播
- chart 是 Grafana 對原始 session 的歷史相對查詢

## 5. Prometheus 資料流

目前 Prometheus 已是長駐服務，不再是 disposable replay cache。

其資料來源有兩條：

1. live scrape `/metrics`
2. replay backfill remote write

它們共同寫進同一個持久 TSDB，並靠 `session` label 區分資料集合。

### replay backfill 的角色

replay backfill 的目的是讓這份 session 在 Prometheus 裡以「原始時間戳」可查。

它不負責播放時的即時映射，也不再建立額外 pseudo session。

### overwrite / delete 的角色

當使用者用 `--backfill=overwrite` 時，`run.py` 會先呼叫 Prometheus admin API 刪除該 session 的 `nwdaf_*` series，再重寫 backfill。

這讓 replay session 可以被明確重灌，而不是依賴每次啟動全域清空 TSDB。

## 6. Live 與 Replay 的關鍵差異

| 面向 | live | replay |
|---|---|---|
| 事件來源 | 遠端 log 即時解析 | 磁碟上的 `events.jsonl` |
| 事件進前端方式 | WebSocket push | `GET /api/events` 一次載入 |
| topology 更新 | 事件到達即時 react | 本地 timeline 重播 |
| metrics 來源 | `/metrics` scrape | remote write backfill |
| 播放中的圖表 | `now-window ~ now` 查 live session | historical relative 查原始 replay session |
| Prometheus session policy | 新 live session 持續寫入 | reuse / overwrite / skip |

## 7. 目前的設計結論

目前 `5g-viz` 的資料流已收斂成：

- 一套 event model 同時餵 topology、event log、session recording
- 一套 metrics projection 把部分 event type 投影成 Prometheus series
- replay 不再有獨立 chart runtime；只是以前端 timeline + 原始 session metrics 重建歷史觀察

這也是目前文件中的 canonical 心智模型。若看到 `MetricPlayer`、pseudo-live、`/api/replay/play`、`start.sh` cleanup 等描述，應視為 pre-refactor historical design。
