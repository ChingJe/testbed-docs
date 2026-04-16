# Event Log

## 這個區塊在做什麼

Event Log 是主畫面中最接近「系統目前把哪些事情判定為事件」的區塊。

它顯示的不是原始 VM log line，而是 parser 產生的結構化事件。

因此 Event Log 的主要用途是：

- 確認剛剛到底是哪一種事件發生
- 檢查 event payload 裡有哪些欄位
- 對照 Topology 與 Grafana 各自對應到哪些事件

## Event Log 內的資料來源

Event Log 的直接來源是 event，不是原始文字 log。

高層來看，資料路徑是：

```text
遠端 log
  -> parser / rules
  -> 結構化 event
  -> Event Log
```

這代表 Event Log 的內容已經過一層語意化處理。

同一類原始 log 在畫面上通常不再以全文顯示，而是被整理成固定 `type` 與 payload。

## Event Log 最適合用來觀察什麼

### 1. Topology 剛剛對應到哪一種事件

當 Topology 發生 edge flash 或 pulse 時，Event Log 通常可用來確認：

- 事件類型是什麼
- 該事件是否真的被 parser 產生
- payload 裡是否帶有相關欄位

### 2. 圖表異常是不是因為根本沒有 metrics event

若 Grafana 沒有線，但 Event Log 中只有 UI-only 事件，通常代表：

- 事件有發生
- 但不是會進 metrics 的那一類事件

### 3. Replay 某個時間點以前到底發生過哪些事件

Pause / scrub 狀態下，Event Log 可以用來看該時間點之前最近一段事件尾端。

這讓 replay 與 live DVR 不只看畫面，也能看事件序列本身。

## Live 與 Replay 下的 Event Log 差異

### Live

Live 模式下，Event Log 主要以即時 append 的方式更新。

只要前端仍處於 live 視角，新事件就會被持續加到 log 裡。

當畫面暫時離開 live 視角時：

- 新事件仍會被接收並存入前端事件緩衝
- 但 log 不一定立刻跟著當前最新事件往下長

### Replay

Replay 模式下，Event Log 主要建立在既有 session 事件之上。

主要行為包括：

- 初始事件來自 `/api/events`
- play 時沿著既有事件往前播放
- pause / scrub 時改顯示指定時間點之前的事件尾端

Replay Event Log 的特點是：

- 它不是在等新的 live event
- 它是在既有事件集合上切換觀看位置

## Pause / Scrub 時 Event Log 發生什麼事

這是 Event Log 最重要的使用情境之一。

當畫面進入 pause 或 scrub 時，Event Log 會從：

- 即時 append 模式

切換成：

- 某個時間點之前的 event tail

所以這個區塊在不同狀態下有兩種閱讀方式：

- live append：追目前發生的事件
- static tail：觀察某個時間點之前最近一段事件

## Event Log 與 `state_snapshot` 的關係

`state_snapshot` 對畫面重建很重要，但它不是 Event Log 的主要閱讀對象。

Event Log 主要關心的是一般事件序列。  
`state_snapshot` 更接近：

- 新連線的狀態同步
- Go Live 時的權威狀態恢復

因此：

- 看到 Event Log 裡沒有 `state_snapshot`，是正常行為
- 這不代表畫面沒有狀態同步機制

## Event Log 適合與哪個區塊一起讀

### 與 Topology 一起讀

這是最常見的搭配。

用途是：

- 確認某個邊動畫對應哪一種事件
- 看某個節點狀態變化前後，事件序列如何排列

### 與 Grafana 一起讀

用途是：

- 確認圖表上的線是否有對應的 metrics event
- 區分 UI-only event 與 metric event

## 常見誤解

### Event Log 沒有某個畫面效果，不代表畫面是錯的

有些畫面上的持久狀態來自：

- state 重建
- event_reaction 累積效果

不一定會在當前 log 視窗中剛好看到對應事件。

### Event Log 有事件，不代表 Grafana 一定會有線

只有一部分 event type 會進 metrics 層。

### Event Log 顯示的是 event，不是原始文字 log

若需要回頭查原始 log 格式與 parser rule，應往 deeper reference 看。

## 對照閱讀

- Topology：[`./topology.md`](./topology.md)
- Event History：[`./event-history.md`](./event-history.md)
- Grafana：[`./grafana.md`](./grafana.md)
- deeper reference：
  - [`../design/frontend/events-and-dvr.md`](../design/frontend/events-and-dvr.md)
  - [`../design/backend/parser.md`](../design/backend/parser.md)
