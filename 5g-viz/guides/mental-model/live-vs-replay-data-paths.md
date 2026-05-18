# Live Vs Replay Data Paths

## 這份文件在說什麼

本文件說明 `live` 與 `replay` 為什麼畫面很像，但資料路徑不同。

這個差異不只是啟動方式不同，而是會直接影響：

- 事件從哪裡來
- 前端如何取得歷史
- `state_snapshot` 怎麼用
- Grafana 查的是哪種 session

## 簡介

可以先把兩種模式簡化成：

```text
live   = runtime 仍在往前跑，前端邊收新事件邊看
replay = session 已經固定，前端在既有事件集合上重建與回放
```

## Live 路徑

### Event 來源

live 的一般事件主要來自：

```text
遠端 log
  -> collector
  -> parser
  -> event
  -> /ws
  -> frontend
```

前端在 live 下會持續收到新事件。

### 初始狀態

live 新連線時，後端會先送：

- `state_snapshot`

這讓前端畫面可以先對齊目前狀態，再接著吃新的 live event。

### 歷史視角

live 下若只是正常即時觀看，前端靠本地 `_events` 持續累積。

live 下若拖曳 timeline 回看較早時間：

- 先優先使用前端本地事件
- 若本地歷史不夠早，再向 `/api/events` 補抓更早區段

因此 live 的歷史視角是：

- runtime 持續往前跑
- 前端暫時離開 live 視角
- 必要時再向後端補歷史

### Grafana

live 下 Grafana 通常查的是：

- 原始 live session
- `now-window ~ now`

## Replay 路徑

### Event 來源

replay 的事件不是從 `/ws` 持續推送。

replay 啟動時，前端主要靠：

- `/api/session-info`
- `/api/events`

把既有 session 的事件載入本地記憶體。

### 初始狀態

replay 沒有一條正在往前長的 live stream 要接上。

前端的起點比較接近：

- 載入整份 session events
- 在既有時間範圍內建立 timeline
- 從起點或指定位置重建畫面

### 歷史視角

replay 下的 pause、scrub、play 都建立在既有 `_events` 上。

這代表：

- 沒有新 live event 在背後持續產生
- 也沒有 live scrub 那種「不夠早就再補抓一些歷史」的主要使用情境

### Grafana

replay 的 Grafana 有兩條資料路徑：

#### replay `paused`

Grafana 查的是：

- 原始 replay session
- backfill 到 Prometheus 的歷史資料

#### replay `playing`

Grafana 查的是：

- 原始 replay session
- historical relative time range

這是 replay 最重要的不對稱之一：資料來源沒換，但查詢時間語意換了。

## 為什麼 replay 不能完全照 live 的方式做

如果 replay 完全照 live 的方式前進，會遇到兩個問題：

1. 原始 session 已經結束，不存在一條真的還在往前跑的 live runtime。
2. Grafana 若只靠拖動絕對 `from/to` 追播放位置，播放中會頻繁 reload iframe。

所以 replay 目前採用的是混合設計：

- Topology / Event Log：前端本地事件回放
- Grafana paused：原始 session 歷史查詢
- Grafana playing：原始 session + historical relative range

## 哪些現象來自這個路徑差異

### live `Pause` 後 backend 仍在往前跑

因為 live runtime 沒停，只是前端離開即時視角。

### replay `Pause` 與 replay `Play` 的 chart 性質不同

因為兩者雖然都查同一個 session，但時間窗語意不同。

### 新分頁只看到目前狀態

這是 live 路徑的結果，不是 replay 路徑的結果。

### replay 啟動時沒有 WebSocket 持續送一般事件

因為 replay 前端的主要來源本來就是 `/api/events`。

## 對照閱讀

- Live Vs Replay：[`../start-here/live-vs-replay.md`](../start-here/live-vs-replay.md)
- Event History：[`../ui-workflows/event-history.md`](../ui-workflows/event-history.md)
- Grafana：[`../ui-workflows/grafana.md`](../ui-workflows/grafana.md)
- deeper reference：
  - [`../../design/overview/data-flow.md`](../../design/overview/data-flow.md)
  - [`../../design/dvr/replay.md`](../../design/dvr/replay.md)
  - [`../../design/frontend/events-and-dvr.md`](../../design/frontend/events-and-dvr.md)
