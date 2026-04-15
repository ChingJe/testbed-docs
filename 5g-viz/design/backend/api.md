# API

本文整理 `main.py` 目前提供的 HTTP API、WebSocket 與 session 查詢行為。

## 1. 整體結構

`5g-viz` 的 backend 是一個單一 FastAPI app，對外主要分成四類介面：

- 觀測輸出：`/metrics`
- 設定與 session 查詢：`/api/*`
- replay 控制：`/api/replay/*`
- live 事件推播：`/ws`

此外，前端靜態檔案掛在：

- `/`

## 2. `GET /metrics`

`/metrics` 直接回傳 `prometheus_client.generate_latest()` 的內容，是 Prometheus 的 scrape endpoint。

特性：

- 沒有額外 query 參數
- 不區分 live / replay；只要程序內已有 metric series，就能被 scrape
- replay mode 的圖表資料不是靠這個 endpoint 回補歷史時間戳，而是靠 remote write；`/metrics` 主要承接 live 寫入的 gauges / counters

## 3. 設定與 session 查詢 API

### `GET /api/grafana-config`

回傳：

```json
{
  "base": "<GRAFANA_BASE>",
  "uid": "<dashboard_uid>"
}
```

前端用它組合 Grafana iframe URL。

### `GET /api/session-info`

回傳目前程序載入的 session 資訊：

```json
{
  "mode": "live" | "replay",
  "session_id": "...",
  "start_time": "...",
  "end_time": "..." | null
}
```

差異：

- live mode：`end_time` 固定為 `null`
- replay mode：`end_time` 會回傳錄製 session 的結束時間

### `GET /api/topology-config`

直接回傳目前載入的 `_topo_config`。這份 payload 同時被：

- `frontend/topology.js`
- backend `state.py`

視為共同設定來源。

### `GET /api/state`

直接回傳 `state.snapshot()`，形式是：

```json
{
  "type": "state_snapshot",
  "nf_status": { ... },
  "node_classes": { ... }
}
```

## 4. Session 清單與事件查詢

### `GET /api/sessions`

列出 `sessions/` 目錄中的所有 session。

回傳元素主要來自各 session 的 `meta.json`，必要時會補算：

- `session_id`
- `start_time`
- `end_time`
- `event_count`

特別行為：

- 若目前程序在 live mode，且遍歷到的 session 正是正在錄製中的 `_current_session_dir`，API 會直接用記憶體中的 `_current_session_meta` 與 `_events` 生成最新狀態
- 若錄製中已被標記為 `corrupted`，回傳結果也會反映這個旗標

### `GET /api/events`

支援參數：

- `session`：要查詢的 session ID；省略時使用目前 active session
- `from`：起始時間，ISO 8601
- `to`：結束時間，ISO 8601
- `limit`：單次最多回傳 50000 筆
- `offset`：分頁偏移

回傳格式：

```json
{
  "events": [ ... ],
  "total": 123,
  "has_more": true
}
```

實作特性：

- 若 `session` 是目前 active session，直接讀記憶體中的 `_events`
- 若是舊 session，會從 `sessions/<session>/events.jsonl` 載入並放進 `_session_events_cache`
- `from` / `to` 會逐筆比對 event 的 `time`
- 時間戳格式錯誤時回 `400`
- session 不存在時回 `404`

## 5. Replay 控制 API

這組 API 只有在 replay mode 且 `_metric_player` 已建立時才可用，否則會回 `404`。

### `POST /api/replay/play`

request body：

```json
{
  "from_time": "<ISO 8601>",
  "speed": 1.0,
  "window": 3
}
```

行為：

- 啟動新的 pseudo-live metric stream
- 回傳新建立的 `pseudo_session`

response：

```json
{
  "pseudo_session": "_live_<session_id>__<token>"
}
```

若 `from_time` 不合法、`speed <= 0` 或 `window < 1`，回 `400`。

### `POST /api/replay/pause`

request body：

```json
{
  "pseudo_session": "..."
}
```

行為：

- 若 pseudo session 正在播放，停止它並回 `{"status": "paused"}`
- 若找不到對應的 active pseudo session，回 `204 No Content`

### `POST /api/replay/speed`

request body：

```json
{
  "pseudo_session": "...",
  "speed": 2.0,
  "current_time": "<ISO 8601>"
}
```

行為：

- 取消目前 emit loop
- 以新的速度與 playhead 位置重建 loop

若 pseudo session 不存在，回 `204 No Content`；若參數不合法，回 `400`。

## 6. WebSocket：`/ws`

`/ws` 是目前唯一的事件推播通道。

連線建立後，server 會：

1. `accept()`
2. 把 websocket 放進 `_clients`
3. 立刻送一份 `state.snapshot()`
4. 進入 `receive_text()` loop，主要用來維持連線

當 live pipeline 有新事件時，`_broadcast(event)` 會：

1. 先 `state.apply_event(event)`
2. 再把 event JSON 廣播給所有 client

目前特性如下：

- 實際上只有 live mode 會持續產生新事件並推送到 `/ws`
- replay mode 雖然 endpoint 仍存在，但 backend 不會重跑 `_process_queue()`，因此一般不作為 replay 前端資料來源
- 若某個 client 發送失敗，server 會把它從 `_clients` 集合移除

## 7. 靜態前端：`/`

FastAPI 最後會：

```python
app.mount("/", StaticFiles(directory="frontend", html=True), name="frontend")
```

因此：

- `/` 會提供 `frontend/index.html`
- 這個 mount 放在最後，避免攔截 `/ws`

這也是目前前端與 API 部署在同一個 process、同一個 origin 下的原因。

## 8. Session 快取與記憶體模型

API 層目前同時依賴兩種記憶體資料：

- `_events`：目前 active session 的完整事件列表
- `_session_events_cache`：查詢過的舊 session 事件快取

這帶來的效果是：

- active session 的事件查詢不需要重讀磁碟
- 同一個舊 session 被查多次時，也不必重複 parse JSONL

但也代表長時間查詢多個 session 時，程序記憶體會累積更多 cached event list。

## 9. 目前限制

- 目前所有 API 都沒有認證與授權機制
- `/api/events` 的篩選是在 Python 迴圈中逐筆處理，沒有更進一步的索引
- `/ws` 目前沒有 server-side 主動 ping/pong 機制，只靠 `receive_text()` 維持連線
- replay 控制 API 只處理 metrics 播放，不會回放原始事件到 WebSocket
