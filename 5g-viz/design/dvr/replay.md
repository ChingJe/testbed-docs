# Replay

> Historical note: this document is primarily a pre-refactor replay runtime reference. It still describes `start.sh`, replay cleanup on startup, pseudo-live, `MetricPlayer`, and replay control APIs that are no longer part of the current system.

本文描述 `5g-viz` 的 replay runtime，以及它如何把同一份 session 同時重建為 topology 播放與 Grafana 圖表。

## 1. Replay 啟動做了什麼

`./start.sh --replay <session_path>` 的啟動流程如下（`start.sh` 本身的行為與 live 模式相同；差異在 app lifespan 根據 `SESSION_MODE` 進行不同的初始化）：

1. `start.sh` 重啟 Prometheus（帶 `--web.enable-admin-api` 與 `--web.enable-remote-write-receiver`）
2. `start.sh` 透過 admin API 刪除所有 `nwdaf_*` series 並清理 tombstones
3. `start.sh` 啟動 FastAPI app（帶 `SESSION_MODE=replay`、`SESSION_PATH` 環境變數）
4. lifespan：清除所有 `_live_.*` pseudo-session series（startup cleanup）
5. lifespan：載入 replay session（`events.jsonl` 與 `meta.json`）
6. lifespan：重建 state
7. lifespan：執行 replay backfill（以原始 timestamp remote write 進 Prometheus）
8. lifespan：建立 `MetricPlayer`
9. lifespan：執行 Grafana setup

這表示 replay 不是在 live session 上「暫停再重放」，而是另起一個專用 runtime。

## 2. Replay 期間哪些東西不會啟動

replay mode 下不會啟動：

- SSH collector
- queue processor
- live WebSocket 事件生產

`/ws` endpoint 雖然仍存在，但一般不作為 replay 前端資料來源。前端會在 bootstrap 時直接走：

- `GET /api/session-info`
- `GET /api/events`

並自動進入 `PAUSED` 狀態。

## 3. Replay Backfill

replay 啟動後，backend 會先把 session 內的 metric events 重建成 Prometheus series，再用 remote write 寫進本機 Prometheus。

這條路徑的目的，是讓 Grafana 可以查到：

- 原始 session ID
- 原始事件時間戳

對應的歷史曲線。

### 何時會跳過

若未開 `FORCE_BACKFILL`，backend 會先查：

```promql
count(nwdaf_ground_truth_ul_bytes{session="<session_id>"})
```

有結果就跳過 backfill。

### 寫入條件

這條路徑依賴：

- Prometheus 已開 `--web.enable-remote-write-receiver`
- Python 可匯入 `snappy`

任一條件不成立時，Grafana replay 圖表可能不可用，但 topology replay 仍可繼續。

## 4. 為什麼 Replay 需要 Pseudo-Live

若 replay 播放期間只靠改 Grafana iframe 的絕對 `from/to` 來追播放位置，每次都會造成 iframe reload。

為了避免播放中頻繁 reload，系統把 replay Grafana 分成兩種模式：

- `BACKFILL`
- `PSEUDO_LIVE`

### `BACKFILL`

用於：

- replay 初始 paused 狀態
- scrub 後停下來的狀態
- replay 播放結束或 pause 之後

此時 iframe 查的是：

- `var-session=<orig_session>`
- 絕對 `from/to`

### `PSEUDO_LIVE`

用於 replay `PLAYING`。此時 iframe 改查：

- `var-session=<pseudo_session>`
- `from=now-<window>m&to=now&refresh=5s`

也就是說，播放期間 Grafana 看的不再是原始 session，而是 backend 重新映射到現在的一條新 session。

## 5. `MetricPlayer` 的角色

`MetricPlayer` 是 replay pseudo-live 的核心元件。

它在初始化時會先從 `_events` 篩出 `METRIC_EVENT_TYPES`，只保留真正會影響 Grafana 的那些事件。之後播放時分兩段進行：

### Pre-seed

開始播放時，`MetricPlayer.play(from_time, speed, window_min)` 會先把 playhead 前一段視窗內的 metric event 映射到現在之前，寫進一個新的 pseudo session。

目的：

- 讓 Grafana 一切到 `now-window ~ now` 就有可看的歷史資料
- 避免播放剛開始時圖表一片空白

目前這段 pre-seed 有三個關鍵細節：

- 歷史回看範圍是 `window_ms * speed`
- 每筆 event 的映射時間為：

```text
mapped_ts = now_ms - int((from_ms - event_ms) / speed)
```

- 只有 `event_ms >= from_ms - window_ms * speed` 的事件會被映射進 pre-seed

也就是說，播放速度越快，為了填滿同樣寬度的 `now-window ~ now` 圖窗，需要回看的原始歷史範圍就越大。

### Emit loop

接著 `_emit_loop()` 會：

1. 按原始事件時間差與播放速度計算 sleep
2. 把到期事件映射成新的 wall clock timestamp
3. 批次 remote write 到 Prometheus

因此 pseudo-live 並不是重播 topology 事件給 Grafana，而是重播 metric event 的時間分佈。

### `_retrain_prefix` 與 cursor

`MetricPlayer` 初始化時，除了保存排序後的 `_metric_events`，還會建立：

- `_metric_times`：每筆 metric event 的原始毫秒時間
- `_retrain_prefix`：到每個 cursor 為止累積了多少個 `retrain_trigger`

`update_speed(...)` 時，backend 會用：

```python
cursor = bisect_right(self._metric_times, current_ms)
retrain_total = self._retrain_prefix[cursor]
```

快速定位新的播放起點與對應的 retrain counter 狀態，而不是重新線性掃描整份事件。

## 6. Replay 控制 API

前端在 replay 模式下會額外用到三個 API：

- `POST /api/replay/play`
- `POST /api/replay/pause`
- `POST /api/replay/speed`

### `play`

`play` 會：

- 驗證參數
- 產生新的 `pseudo_session`
- 停掉先前 active stream
- 做 pre-seed
- 啟動新的 emit loop

前端拿到 `pseudo_session` 後，才切 Grafana iframe。

### `pause`

`pause` 只會停止目前 active 的 pseudo-live stream。若收到的是舊 token，會回 `204`，不去影響當前播放。
之後 backend 還會刪掉該 `pseudo_session` 的 `nwdaf_*` series，避免 `_live_...` 長期留在 Grafana
session 下拉選單裡。

### `speed`

`speed` 會：

- 依 `current_time` 重新定位 cursor
- 計算到該點為止的 `retrain_total`
- cancel 舊 task
- 以新速度重建 emit loop

也就是說，變速不是原 loop 內原地改 sleep，而是重新對齊後續播放。

## 7. 前端與 `MetricPlayer` 如何協調

replay `PLAYING` 時，前端與 backend 各自維護一條播放節奏：

- 前端：`_tickPlayback()` 推進 topology 與 event log
- backend：`MetricPlayer` 推進 pseudo-live metrics

兩者只在下列節點做弱同步：

- play 開始
- pause
- speed 變更
- chart window 變更

這種設計的含義是：

- topology 與 Grafana 不追求毫秒級嚴格同步
- 更重視操作簡單與播放體感

## 8. 一致性邊界

replay 有一個重要現況：`PAUSED` 和 `PLAYING` 看到的 Grafana 圖，底層不是同一組樣本。

### `PAUSED`

- 查原始 replay session backfill
- 使用原始事件時間戳

### `PLAYING`

- 查本次播放的 pseudo session
- 使用映射到 `now` 的新時間戳

因此：

- 同一段歷史在 `PAUSED` 與 `PLAYING` 間可能不完全一致
- 同一段歷史在不同次 `PLAYING` 間，也可能因為 wall clock 相位不同而長得不完全一樣

目前系統把這視為可接受的 tradeoff：`PAUSED` 偏向忠實歷史，`PLAYING` 偏向近似 live 體感。

## 9. 已知不對稱之處

### `model_swap`

live mode 的 metric handler 會在 `model_swap` 時刪掉舊 deviation series。replay backfill 與
`MetricPlayer` 無法直接重播 exporter `remove()`，所以改在 `model_swap` 時對目前 active models
寫一筆 `NaN` sample。

結果是：

- replay graph 也會在 swap 當下截斷舊 model 線段
- 新 model 第一筆 `accuracy` 進來前，會保留和 live 一致的 cold-start gap


## 10. 圖表視窗與播放控制

目前 replay 還有一個特殊點：Chart window 改變時，若正在 `PLAYING`，前端不只會 reload iframe，還會：

1. pause 目前 pseudo-live
2. 以目前 playhead 和新 window 重啟一個 pseudo-live
3. 取得新的 pseudo session
4. 繼續播放

原因是 pre-seed 範圍本身就依賴 window 大小。

## 11. 目前限制

- replay 前端會把整個 session 的事件一次載入記憶體；超長 session 時記憶體與初始等待時間會上升
- pseudo-live 不保證與 paused backfill 完全數值一致
- `MetricPlayer` 只處理 metric event，不處理 topology event；兩條播放面是分開推進的
- replay 仍是單一 active player 設計，多個使用者不應同時操作同一個 replay backend
- 每次 `start.sh` 啟動都清除所有 `nwdaf_*` series；Prometheus 不是累積式 session 資料倉，Grafana 上可查的歷史範圍取決於當次啟動後寫入的內容
