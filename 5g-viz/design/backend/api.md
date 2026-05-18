# API

本文整理 `5g-viz` 目前提供的 HTTP API、WebSocket 與 Grafana proxy 行為。內容以 `backend/app.py` 為準。

## 1. 整體結構

目前 `5g-viz` 對外主要分成五類介面：

- 觀測輸出：`/metrics`
- session / topology / state 查詢：`/api/*`
- live 事件推播：`/ws`
- Grafana 同源代理：`/grafana/*`
- 前端靜態頁面：`/`

注意：

- 舊的 `/api/replay/play`、`/api/replay/pause`、`/api/replay/speed` 已移除
- replay orchestration 現在由 CLI `uv run run.py ...` 與 runtime context 決定，而不是靠 browser-facing mutation API

## 2. `GET /metrics`

`/metrics` 直接回傳 `prometheus_client.generate_latest()`，是 Prometheus scrape endpoint。

特性：

- 無 query 參數
- live 與 replay 都可存在，但主要承接 live exporter metrics
- replay 的歷史樣本不是靠這個 endpoint 補回，而是靠 app 啟動時的 remote write backfill

## 3. Grafana 相關 API

### `GET /api/grafana-config`

回傳前端嵌入 Grafana 所需的最小資訊：

```json
{
  "base": "/grafana",
  "uid": "nwdaf-traffic"
}
```

前端用它來組 iframe URL。

### `/grafana` 與 `/grafana/{path:path}`

這兩條是 HTTP proxy，會把同源 `/grafana/*` 請求轉發到真正的 Grafana upstream。

用途：

- 讓前端能透過 `8765` 同源載入 Grafana iframe
- 支援 iframe 裡的後續 API / asset 請求

### `WS /grafana/{path:path}`

Grafana 的 websocket 路徑也透過 backend proxy。

## 4. Session 與 topology 查詢 API

### `GET /api/session-info`

回傳目前程序載入的 session 狀態：

```json
{
  "mode": "live" | "replay",
  "session_id": "...",
  "start_time": "...",
  "end_time": "..." | null
}
```

差異：

- live：`end_time = null`
- replay：`end_time` 來自 session `meta.json`

### `GET /api/sessions`

列出 `sessions/` 目錄中的 session 清單，主要來自各 session `meta.json`。

目前會回傳的主要欄位包括：

- `session_id`
- `start_time`
- `end_time`
- `event_count`
- `profile`
- `corrupted`

若目前程序正在 live 錄製，API 會直接用記憶體中的最新 metadata 與事件數更新正在錄製中的 session 條目。

### `GET /api/topology-config`

回傳目前前端應使用的 topology config。

來源：

- live：目前 profile 的 `profiles/<profile>/topology.yaml`
- replay：目前 session 目錄內保存的 `topology.yaml`

### `GET /api/state`

回傳目前 `state.snapshot()`，格式類似：

```json
{
  "type": "state_snapshot",
  "nf_status": { ... },
  "node_classes": { ... }
}
```

主要用於：

- live 新 client 連線後的初始畫面
- `Go Live` 回到目前權威狀態

## 5. `GET /api/events`

`/api/events` 是 live / replay 共用的事件查詢 API。

支援參數：

- `session`
- `from`
- `to`
- `limit`
- `offset`

回傳格式：

```json
{
  "events": [ ... ],
  "total": 123,
  "has_more": true
}
```

實作特性：

- 若查的是目前 active session，優先使用記憶體中的 `_events`
- 若查的是舊 session，從 `sessions/<id>/events.jsonl` 載入並快取
- `from/to` 逐筆比對 event `time`
- 時間格式錯誤回 `400`
- session 不存在回 `404`

這條 API 在兩種情境下都很重要：

- replay bootstrap：前端一次載入 session events
- live scrub：若本地 `_events` 不夠早，前端會補抓較早歷史

## 6. `WS /ws`

`/ws` 是目前唯一的 live 事件推播通道。

連線建立後，server 會：

1. `accept()`
2. 把 websocket 放進 `_clients`
3. 立刻送一份 `state.snapshot()`
4. 進入 keep-alive receive loop

當 live pipeline 有新事件時，backend 會：

1. 先 `state.apply_event(event)`
2. 再把 event JSON 廣播給所有 client

注意：

- replay 一般事件不依賴這條 WebSocket
- `state_snapshot` 是 live 初始同步與 `Go Live` 的權威狀態來源，不是完整歷史回放

## 7. 錯誤邊界與目前限制

- `session-status`、`overwrite`、`skip` 等 replay decision flow 不走 HTTP，而走 CLI
- `GET /api/events` 對舊 session 會依賴磁碟 `events.jsonl`；若檔案缺失，前端無法補歷史
- backend 目前不提供 browser-facing Prometheus mutation API
- Grafana proxy 只負責轉發，不解讀 dashboard query 語意
