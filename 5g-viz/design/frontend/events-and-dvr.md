# Events And DVR

本文描述 `frontend/events.js` 目前如何管理 session、事件來源、timeline 與 DVR 控制。

## 1. 這層的責任

`events.js` 是前端控制平面，負責：

- 啟動時取得 session 與 Grafana 設定
- 決定 live / replay 兩種前端模式
- 維護 `_events` 緩衝、timeline 與 event log
- 控制 `LIVE / PAUSED / SCRUBBING / PLAYING / RESUMING` 狀態切換
- 同步 topology、event log、Grafana iframe 的時間視角

它不直接解讀 topology schema；所有 topology 操作都透過 `window.Topology` 完成。

## 2. Bootstrap 順序

前端啟動時會先等待：

- `TopologyReady`
- `GET /api/grafana-config`
- `GET /api/session-info`

之後依 `session-info.mode` 分兩條路徑：

### live

- 初始 DVR 狀態設為 `LIVE`
- 建立 `/ws`
- timeline 右邊界會隨新事件持續向前推進

### replay

- 初始 DVR 狀態設為 `PAUSED`
- 先透過 `GET /api/events` 把 session events 載入前端記憶體
- 不建立一般事件 WebSocket
- timeline 初始位置停在 session 起點

## 3. DVR 狀態機

目前使用五個顯式狀態：

| 狀態 | 說明 |
|---|---|
| `LIVE` | 即時視角 |
| `PAUSED` | 停在某個時間點 |
| `SCRUBBING` | 正在拖曳 timeline |
| `PLAYING` | 依事件時間差往前播放 |
| `RESUMING` | 正在切回 live 或重新對齊狀態 |

## 4. 事件緩衝

所有非 `state_snapshot` 事件都會進 `_events`。

特性：

- `_events` 是 live DVR 與 replay playback 的共同資料池
- `_ensureSorted()` 只在需要時排序
- `state_snapshot` 不進 `_events`

這代表 `_events` 不是單純的 event log 顯示來源，而是：

- static snapshot 重建來源
- replay 播放來源
- live scrub 歷史來源

## 5. Live 模式的資料來源

live 主要依賴 `/ws`。

### 連線建立後

server 會先送一份 `state_snapshot`。

前端只會在特定狀態套用它，例如：

- `LIVE`
- `RESUMING`

若使用者正在 paused / scrubbed 視角，新的 snapshot 不會直接覆蓋目前畫面。

### 一般事件

一般事件會：

1. 進入 `_events`
2. 更新 timeline 邊界
3. 僅在 `LIVE` 狀態時立即 `dispatch(event)`

因此 live paused / scrubbed 期間，事件仍會持續累積，只是不即時改變目前畫面。

## 6. Replay 與 live DVR 行為

### Pause

- live：停住的是觀看視角，不是 backend runtime
- replay：停住的是播放位置

兩種模式下，前端都會重建該時間點的 topology 與 event log tail。

### Scrub

拖曳 timeline 時，前端會批次執行：

- `Topology.renderStaticSnapshot(...)`
- `_renderLogTailAt(targetMs)`

live 下若本地 `_events` 不夠早，還會向 `/api/events` 補抓歷史。

### Play

從 `PAUSED` 進入 `PLAYING` 時：

1. 取消 scrub render
2. 以 `_findFirstIndexAfter(_timelinePosMs)` 找到播放起點
3. 依相鄰事件原始時間差安排 `setTimeout`
4. 若目前畫面上已有仍 active 的 edge / pulse，先續播剩餘 `remainingMs`

目前已沒有 replay speed；播放節奏固定跟著事件原始時間差。

### Go Live

`Go Live` 只在 live mode 有意義。它會：

1. 嘗試取 `/api/state`
2. 套用 backend 權威 snapshot
3. timeline 跳到最新位置
4. 恢復即時 dispatch

若 `/api/state` 失敗，才退回用 `_events` 在最新時間點做一次靜態重建。

## 7. Replay 與 Grafana 的耦合

replay mode 的 topology 播放與 Grafana 圖表不是同一條資料流。

目前前端在 replay 下只維護兩種 Grafana 模式：

- `BACKFILL`
- `RELATIVE_PLAY`

### `BACKFILL`

- 原始 session
- 固定絕對時間窗
- 用於 paused / scrubbed

### `RELATIVE_PLAY`

- 原始 session
- historical relative range
- 用於 replay `PLAYING`

也就是說，replay 播放現在不再協調 pseudo-live session，而是切換 query 時間語意。

## 8. Event Log 行為

event log 目前有兩套輸出模式：

- live append：最多保留 200 行
- static tail：paused / scrub 時重畫指定時間點之前最近 50 筆

`state_snapshot` 不顯示在 event log 中。

## 9. Keyboard timeline step

前端支援：

- `ArrowLeft`
- `ArrowRight`

每次移動的秒數由 `Step` 控制，並存進 `localStorage`。

特性：

- 只影響 keyboard scrub
- 不等於 playback speed
- timeline slider 自己有 focus 時也會走這條 step scrub，而不是只吃原生 range input 行為

## 10. 目前限制

- live 歷史補抓目前仍以 append / sort 為主，沒有完整 dedupe
- replay 會把整個 session 的 event list 載入記憶體
- 播放 scheduler 仍是單一 `setTimeout` 鏈，不是更細粒度的播放引擎
