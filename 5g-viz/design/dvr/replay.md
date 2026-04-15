# Replay

本文描述 `5g-viz` 的 replay runtime，以及它如何把同一份 session 同時重建為 topology 播放與 Grafana 圖表。

## 1. Replay 啟動做了什麼

`./start.sh --replay <session_path>` 目前會把系統帶進一個和 live 不同的啟動路徑：

1. 清空 managed Prometheus TSDB
2. 重新啟動 Prometheus，並開啟 remote write receiver
3. 啟動 FastAPI app
4. 在 lifespan 中載入 replay session
5. 重建 state
6. 執行 replay backfill
7. 建立 `MetricPlayer`
8. 執行 Grafana setup

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

### Emit loop

接著 `_emit_loop()` 會：

1. 按原始事件時間差與播放速度計算 sleep
2. 把到期事件映射成新的 wall clock timestamp
3. 批次 remote write 到 Prometheus

因此 pseudo-live 並不是重播 topology 事件給 Grafana，而是重播 metric event 的時間分佈。

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

live mode 的 metric handler 會在 `model_swap` 時刪掉舊 deviation series，但 replay backfill 與 `MetricPlayer` 都沒有真正重播這個刪除動作。

結果是：

- replay graph 的 deviation panel 和 live 可能不完全等價
- 目前主要靠 Grafana query 只取最新 deviation series 來壓低差異

### Replay 啟動時清空 TSDB

這有助於讓 replay 驗證結果乾淨、可重現，但也表示 Prometheus 不是長期累積所有 session 的資料倉，而是每次 replay 啟動都可被重置的暫態環境。

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
- replay run 內舊 pseudo sessions 仍會留在 TSDB 中，只是 Grafana 不再查它們
