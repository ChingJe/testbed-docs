# Overview

本文從功能層描述 `5g-viz` 的 DVR。這裡的 DVR 不只是前端控制列，而是一組跨越 session 錄製、事件重建、Grafana 視窗切換與 replay metric pipeline 的整體機制。

## 1. DVR 的定位

`5g-viz` 的 DVR 目標是讓使用者能在同一套 UI 中做三件事：

- 在 live 實驗進行中暫停、拖曳與回放既有事件
- 將一次實驗錄成可重播的 session
- 事後以 replay 模式重看 topology、event log 與 Grafana 曲線

其中：

- topology / event log 以 event stream 為主
- Grafana 曲線以 Prometheus metrics 為主
- session 錄製則以 `events.jsonl` 為核心持久化格式

## 2. 兩種執行模式

### Live

live mode 仍維持 collector -> parser -> broadcast 的即時管線，但額外多了：

- 寫入 `events.jsonl`
- 寫入 `meta.json`
- 複製當前 `topology.yaml` 到 session 目錄

使用者進入 pause / scrub / play 時，backend 不會停下來；只是前端暫停直接消費最新事件。

### Replay

replay mode 不啟動 collector，也不依賴 WebSocket 來取得新事件。它會：

- 從 session 目錄載入 `events.jsonl`
- 重建 state
- 將 metric events backfill 到 Prometheus
- 建立 `MetricPlayer` 以支援 pseudo-live 播放

因此 replay 比較像「以既有事件檔重新開一個唯讀 runtime」，而不是 live runtime 的縮時版。

## 3. Live DVR 操作路徑

live mode 下的 DVR 不是另一個 backend 模式，而是前端暫時改變「如何消費 live 事件」。

### Pause

當使用者按下 Pause：

- frontend 狀態從 `LIVE` 進入 `PAUSED`
- backend 仍持續 collect、parse、寫 JSONL、更新 metrics、broadcast
- 前端繼續把新事件存入 `_events`，但不再即時 dispatch 到 topology / log
- Grafana 從 `now-window ~ now` 切成以目前停點為右邊界的 trailing absolute window

### Scrub

當使用者拖曳 timeline：

- frontend 進入 `SCRUBBING`
- topology 改由 `Topology.renderStaticSnapshot(...)` 重建指定時間點
- event log 改為顯示該時間點之前最近一段 tail
- 若目前前端 buffer 不夠早，會向 `/api/events` 補拉該 live session 的歷史事件

### Play

當使用者從 live paused 狀態按下 Play：

- frontend 依事件原始時間差與 `_playSpeed` 重播既有 `_events`
- backend 仍持續向前產生新事件，但前端先不即時消費它們
- 若播放追到目前 live 邊界，前端會自動回到 `LIVE`

### Go Live

當使用者按下 Go Live：

- frontend 進入 `RESUMING`
- 優先呼叫 `/api/state` 取得 server 端最新 `state_snapshot`
- 套用 `Topology.applySnapshot(...)`
- timeline 跳到最新位置，event log 重新顯示近期 live tail
- Grafana 恢復 `now-window ~ now`

若 `/api/state` 失敗，前端才退回用本地 `_events` 做一次靜態重建。

### 這條路徑的重點

live DVR 的本質是：

- backend 一直往前跑
- frontend 暫時脫離即時視角
- 使用者隨時可再接回權威 live state

因此它和 replay 最大的差異不是 UI，而是資料來源始終仍有「正在進行中的 live 流」。

## 4. 前端 DVR 狀態

前端目前把 DVR 明確分成五個狀態：

| 狀態 | 主要意義 |
|---|---|
| `LIVE` | 即時顯示最新事件 |
| `PAUSED` | 停在某一時間點 |
| `SCRUBBING` | 使用者正在拖曳 timeline |
| `PLAYING` | 從目前時間點往前播放後續事件 |
| `RESUMING` | 正在切模式，例如 Go Live 或重啟 replay pseudo-live |

但這五個狀態只定義前端消費事件的方式，不表示 backend pipeline 本身會跟著停下來。

## 5. 三條需要一起協調的資料面

DVR 之所以複雜，是因為它同時協調三條不同的資料面。

### Topology

- 來源是結構化 events
- `LIVE` 時直接 dispatch
- `PAUSED / SCRUBBING` 時用 `Topology.renderStaticSnapshot(...)` 做靜態重建
- `PLAYING` 時依事件時間差重播動畫

### Event Log

- 也是以 events 為來源
- live append 與 paused tail render 共用同一份 `_events` 緩衝

### Grafana

- 來源不是 event 本身，而是 Prometheus 裡的 metrics
- live 與 paused 看的時間窗不一樣
- replay `PAUSED` 看的是原始 replay session backfill
- replay `PLAYING` 看的是 pseudo-live session

這三條資料面共享同一個播放位置，但不共享同一條底層資料路徑。

## 6. 各層責任分工

| 層 | 主要責任 |
|---|---|
| `main.py` | 決定 live / replay 模式、建立 session、載入 replay session、提供 `/api/events`、`/api/state`、`/api/replay/*` |
| `frontend/events.js` | 管理 DVR 狀態機、timeline、事件緩衝、播放控制與 Grafana 模式切換 |
| `frontend/topology.js` | 依 event reactions 重建 paused / scrub 的靜態拓樸 |
| `metric_player.py` | replay `PLAYING` 期間產生 pseudo-live metrics |
| `grafana_setup.py` | 確保 dashboard 與 datasource 存在 |

這種分工也決定了 canonical 文件的切法：

- 前端操作細節在 [../frontend/events-and-dvr.md](../frontend/events-and-dvr.md)
- session 落盤與查詢介面在 [session.md](./session.md)
- replay backfill 與 pseudo-live 在 [replay.md](./replay.md)

## 7. 時間語意

DVR 目前同時用到兩種時間：

- event 原始時間：來自 JSONL / replay session
- 真實現在時間：用於 live mode 與 replay pseudo-live 映射

這帶來一個重要區分：

- paused / scrub 時，使用者看到的是某個歷史時間點的重建結果
- replay playing 時，topology 播放仍沿用歷史事件順序，但 Grafana 看到的是映射到 `now` 的 pseudo-live metrics

也就是說，`PLAYING` 的 Grafana 體驗追求的是「近似 live」，不是「與 paused 視窗完全同一條時間軸」。

## 8. 目前邊界

目前 DVR 已完成的範圍包括：

- live session 錄製
- replay 載入
- `/api/sessions`、`/api/events`、`/api/state`
- timeline pause / scrub / play / go-live
- replay backfill
- replay pseudo-live 與速度控制

尚未做成正式產品化功能的部分包括：

- 專用的 export / import CLI 或 UI 流程
- 更細的 playback checkpoint / snapshot 加速機制
- 對 replay pseudo-live 與 paused backfill 完全一致的數值保證

## 9. 閱讀順序

若要沿著 DVR 主線閱讀，建議順序是：

1. [session.md](./session.md)
2. [replay.md](./replay.md)
3. [../frontend/events-and-dvr.md](../frontend/events-and-dvr.md)
4. [../frontend/grafana-embed.md](../frontend/grafana-embed.md)
