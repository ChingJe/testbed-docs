# Common Scenarios

## 這份文件在說什麼

本文件整理 `5g-viz` 最常見的幾種現象與解釋。

重點不是列出所有 debug 手法，而是先回答：

- 這種現象通常代表什麼
- 應先對照哪個區塊
- 哪些差異是系統設計本來就存在的

## 1. Topology 有反應，但 Grafana 沒線

### 最常見的解釋

發生的是 UI-only event，不是 metric event。

### 先看哪裡

- Event Log
- Grafana

### 典型原因

- `sbi_call`、`adrf_*`、`threshold_breach` 這類事件只會驅動畫面
- Grafana 正在看錯 session
- Grafana 正在固定時間窗，不在 `now-window ~ now`

### 補充說明

Topology 與 Grafana 不是同一條資料層。  
Topology 有反應，只能證明 event 被辨識並投影到 UI，不等於該事件一定會寫成 metrics。

## 2. 新開 live 分頁，只看到當前狀態

### 最常見的解釋

新連線先同步的是 `state_snapshot`，不是整段歷史重播。

### 先看哪裡

- Topology
- Event History

### 補充說明

這個行為的重點是：

- 先把畫面放到目前狀態
- 再開始接收之後的新事件

若需要觀察過去如何走到這個狀態，應改用 DVR 與 event history。

## 3. live `Pause` 之後，畫面停住，但後端仍在往前跑

### 最常見的解釋

`Pause` 停的是前端觀看視角，不是 backend runtime。

### 先看哪裡

- DVR Controls
- Event History

### 補充說明

這段期間：

- WebSocket 事件仍會持續進前端
- `_events` 仍會持續累積
- 只是前端暫時不把新事件即時 dispatch 到目前畫面

## 4. replay `paused` 的圖，和 replay `playing` 的圖不完全一樣

### 最常見的解釋

兩者查的時間窗語意不同。

### 先看哪裡

- Grafana
- Live Vs Replay Data Paths

### 補充說明

- replay `paused` 主要看原始 session 的固定歷史時間窗
- replay `playing` 主要看原始 session 的 historical relative 視窗

因此同一段歷史在兩種狀態下，不保證完全一致。

## 5. replay 為什麼需要 backfill

### 最常見的解釋

沒有 backfill，Grafana 無法直接查到原始 session 的歷史時間序列。

### 先看哪裡

- Grafana
- Live Vs Replay Data Paths

### 補充說明

replay 啟動時，backend 會先把原始 session 的 metric events 寫回 Prometheus。  
這樣 replay 在停住或 scrub 時，Grafana 才能直接顯示原始時間軸上的歷史圖。

## 6. replay `Play` 剛開始時，Grafana 為什麼不是空白

### 最常見的解釋

因為 replay `playing` 會切成 historical relative 視窗，不再是停住時那張固定歷史圖。

### 先看哪裡

- Grafana
- Live Vs Replay Data Paths

### 補充說明

播放中圖表會跟著 `now` 平滑滑動，所以剛按播放時，視覺上看到的已經不是原本停住那張固定圖。

## 7. 改 Chart Window 時，replay 播放中的 chart 為什麼會像重啟

### 最常見的解釋

因為這不只是 iframe 換一個時間窗，也會改變 replay `playing` 使用的 relative query。

### 先看哪裡

- DVR Controls
- Grafana

### 補充說明

Chart Window 會影響 replay `playing` 的 relative query 範圍。  
因此播放中改變 window，圖表會明顯重新同步到新的查詢視窗。

## 8. Event Log 有事件，但畫面上的過去時間點和預期不一致

### 最常見的解釋

目前看的可能是：

- live 的本地事件緩衝
- live 補抓後的歷史
- replay 已載入的既有 session events

### 先看哪裡

- Event History
- DVR Controls

### 補充說明

在 live 下拖曳較早時間時，前端可能會先用本地 `_events`，不夠時再向 `/api/events` 補抓。  
因此畫面是否已涵蓋足夠早的歷史，會影響 scrub 結果。

## 9. 某段 replay 歷史中，Topology 有動作，但 deviation panel 看起來和 live 不完全等價

### 最常見的解釋

`model_swap` 在 replay path 沒有完全重播 live path 中的 series 清理行為。

### 先看哪裡

- Grafana
- metrics / replay reference

### 補充說明

目前 live metrics path 會在 `model_swap` 時清理舊 deviation series。  
replay backfill 主要保留原始歷史寫入結果，因此和 live runtime 不保證完全一致。

## 對照閱讀

- [`../mental-model/events-snapshots-metrics.md`](../mental-model/events-snapshots-metrics.md)
- [`../mental-model/live-vs-replay-data-paths.md`](../mental-model/live-vs-replay-data-paths.md)
- [`../mental-model/ui-only-vs-metric-events.md`](../mental-model/ui-only-vs-metric-events.md)
