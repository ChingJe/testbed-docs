# DVR Controls

## 這個區塊在做什麼

DVR Controls 是主畫面的時間控制層。

它不是只控制 Topology，也不是只控制 Grafana。  
這一排 controls 會一起影響：

- Topology 現在顯示哪個時間點
- Event Log 現在顯示哪一段
- Grafana 現在查哪個時間窗、哪個 session

因此，DVR Controls 比較像主畫面的時間視角控制台。

## 控制項一覽

目前主要控制項包括：

- `Pause`
- `Play`
- `Go Live`
- timeline
- speed
- chart window
- reset chart

## `Pause`

`Pause` 的作用是暫停目前的觀看視角。

### 在 live

`Pause` 代表：

- 前端畫面停在某個時間點
- backend runtime 仍繼續收事件、寫 session、更新 metrics

也就是說，Pause 不會讓 live runtime 停止，只會讓前端先停在某個觀察位置。

### 在 replay

`Pause` 代表：

- replay 停在某個播放位置
- Topology 與 Event Log 顯示該時間點的重建結果
- Grafana 回到該 replay 位置對應的資料視窗

## `Play`

`Play` 的作用是從目前時間點往後重播既有事件。

### 在 live

Live 下的 `Play` 主要作用是：

- 把 pause / scrub 期間停住的視角往後推進
- 播放追到目前 live 邊界後，再回到 live 視角

### 在 replay

Replay 下的 `Play` 除了推動 Topology 與 Event Log 之外，還會啟動 pseudo-live chart 路徑。

這代表 replay `Play` 不只是單純重播前端事件，還會改變 Grafana 的資料來源。

## `Go Live`

`Go Live` 只對 live mode 有意義。

它的作用是：

- 結束 pause / scrub / playback 的歷史視角
- 把畫面切回目前 runtime 的最新狀態

高層來看，`Go Live` 會讓三個區塊重新對齊到「現在」：

- Topology：回到目前權威狀態
- Event Log：回到近期 live tail
- Grafana：回到 `now-window ~ now`

Replay 沒有這個按鈕的等價物，因為 replay 沒有「更即時的現在」可以回去。

## Timeline

Timeline 是選擇時間點的主控制項。

### 在 live

Live 下的 timeline 右邊界會持續往前推。

這代表 timeline 的右端代表：

- 目前已知最新的 live 邊界

### 在 replay

Replay 下的 timeline 範圍主要來自錄製 session 的起訖時間。

這代表 timeline 的兩端都是固定錄製範圍。

### Scrub

拖曳 timeline 時，畫面會進入 scrub。

這時：

- Topology 會重建某個時間點的靜態畫面
- Event Log 會顯示該時間點之前最近一段事件
- Grafana 會切到對應的時間窗

在 live 下，若前端緩衝不夠早，還可能補抓較早的事件。

## `Speed`

`Speed` 只在播放狀態下有明顯效果。

它控制的是：

- 重播事件的推進速度

在 replay 模式下，`Speed` 也會影響 pseudo-live chart 的節奏，因為 chart 播放不是獨立於 playback speed 的。

## `Chart Window`

`Chart Window` 控制的是：

- Grafana 顯示的時間窗寬度

這個控制項與 Topology / Event Log 不同步縮放；它只影響圖表視窗。

### 在 live 與 paused 狀態

這通常只是改變目前圖表的查詢範圍。

### 在 replay playing 狀態

這會連帶影響 pseudo-live chart 路徑，因為：

- pre-seed 的範圍與 window 大小耦合

所以 replay playing 下改 Chart Window，實際上比一般狀態更接近一次小型重啟。

## `Reset Chart`

`Reset Chart` 的作用是：

- 把 Chart Window 恢復到預設值
- 強制重新同步 Grafana 到目前模式對應的預設圖窗

這個操作不等於 `Go Live`。  
它只處理 chart window 與 iframe 同步，不處理整體時間視角。

## DVR Controls 與三個觀測面的關係

### 對 Topology

- 控制當前畫面是 live dispatch、靜態重建，還是播放中的逐步推進

### 對 Event Log

- 控制 log 是即時 append，還是某個時間點之前的 event tail

### 對 Grafana

- 控制圖表看的是現在、固定時間窗，還是 replay 的 pseudo-live 路徑

## 常見誤解

### `Pause` 不是系統暫停

在 live 下，Pause 只代表前端視角停住，不代表 backend runtime 停止。

### `Go Live` 不是 replay 的功能

Replay 沒有一個對應的「回到最新 live 狀態」操作。

### `Chart Window` 不等於 timeline zoom

它只控制 Grafana 的圖表視窗，不直接改變 Topology 或 Event Log 的縮放方式。

## 對照閱讀

- Topology：[`./topology.md`](./topology.md)
- Event Log：[`./event-log.md`](./event-log.md)
- Event History：[`./event-history.md`](./event-history.md)
- Grafana：[`./grafana.md`](./grafana.md)
- deeper reference：
  - [`../design/frontend/events-and-dvr.md`](../design/frontend/events-and-dvr.md)
  - [`../design/dvr/overview.md`](../design/dvr/overview.md)
  - [`../design/dvr/replay.md`](../design/dvr/replay.md)
