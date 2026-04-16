# Events, Snapshots, Metrics

## 這份文件在說什麼

本文件說明 `5g-viz` 裡最容易混在一起的三種資料：

- event
- `state_snapshot`
- metrics

三者彼此相關，但用途不同。  
只要這三層的分工清楚，多數畫面現象都會比較容易解釋。

## 簡介

可以先把三者理解成：

```text
event           = 發生了什麼變化
state_snapshot  = 現在已經成立的狀態
metrics         = 某些事件被投影成可查詢的時間序列
```

## Event 是什麼

event 是系統裡最核心的共同資料模型。

它來自：

```text
遠端 log
  -> parser / rules
  -> 結構化 event
```

event 的用途包括：

- 驅動 Topology 的邊動畫、pulse、持久 class
- 顯示在 Event Log
- 成為 replay / scrub / playback 的歷史來源
- 讓部分事件再被轉成 metrics

因此 event 比較像：

- 系統把某件事情辨識成「有語意的事件」之後的標準形式

## `state_snapshot` 是什麼

`state_snapshot` 不是 parser 從 log 直接產生的事件。

它是 backend 依據目前狀態推導出的快照，主要用來回答：

- 現在這一刻畫面應該站在哪個狀態

它的用途包括：

- live 新連線時的初始畫面同步
- `Go Live` 時回到 backend 的目前狀態

它不提供的內容包括：

- 完整事件歷史
- 邊動畫
- timeline
- replay 播放順序

`state_snapshot` 的重點不是描述「剛剛發生了什麼」，而是描述「此刻已經成立了什麼」。

## Metrics 是什麼

metrics 是 event 的第二層投影，不是另一套獨立事件系統。

只有一部分 event type 會被寫成 Prometheus metrics，例如：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `model_swap`

這些 metrics 最後會被：

```text
Prometheus
  -> Grafana
  -> iframe
```

顯示成時間序列圖。

metrics 最適合表達的是：

- 流量數值
- prediction / ground truth
- deviation
- retrain counter

## 三者與畫面區塊的對應

### Topology

Topology 主要讀的是：

- event
- `state_snapshot`

其中：

- live 新連線與 `Go Live` 偏向先吃 `state_snapshot`
- 一般互動、pause、scrub、play 偏向建立在 event 上

### Event Log

Event Log 主要讀的是：

- event

它不以 `state_snapshot` 為主，也不直接顯示 metrics。

### Grafana

Grafana 主要讀的是：

- metrics

它不直接讀 event buffer，也不直接讀 `state_snapshot`。

## 為什麼同一件事會在不同區塊看起來不一樣

因為三個區塊不是同一層資料的不同皮膚，而是不同投影。

### 同一筆 event 對 Topology 和 Grafana 的影響不同

有些 event 只會改變 Topology，不會進 metrics。

結果就是：

- Topology 有反應
- Event Log 有事件
- Grafana 沒有對應曲線

### `state_snapshot` 可以讓 Topology 立刻就位，但 Event Log 不會跟著補歷史

`state_snapshot` 解決的是目前狀態同步，不是歷史回放。

結果就是：

- 新分頁的 Topology 先對齊到目前狀態
- Event Log 不會自動長出整段過去事件

### replay `paused` 和 replay `playing` 的 Grafana 不是同一條資料路徑

`paused` 主要看原始 session backfill。  
`playing` 主要看 pseudo-live session。

結果就是：

- Topology 看起來都像在回看同一段歷史
- Grafana 在不同播放狀態下，底層實際上不是同一組樣本

## 哪些問題最適合先用這個心智模型回答

- 為什麼新分頁只看到當前狀態
- 為什麼 Topology 有反應，但 Grafana 沒線
- 為什麼 replay 的圖在 paused 與 playing 間不完全一樣
- 為什麼 `Go Live` 不是單純把 timeline 拉到最右邊

## 對照閱讀

- Event History：[`../ui-workflows/event-history.md`](../ui-workflows/event-history.md)
- Grafana：[`../ui-workflows/grafana.md`](../ui-workflows/grafana.md)
- deeper reference：
  - [`../../design/backend/state.md`](../../design/backend/state.md)
  - [`../../design/backend/metrics.md`](../../design/backend/metrics.md)
  - [`../../design/frontend/events-and-dvr.md`](../../design/frontend/events-and-dvr.md)
