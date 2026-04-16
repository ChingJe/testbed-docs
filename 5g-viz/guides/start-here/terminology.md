# Terminology

本文收錄 `5g-viz` guides 中幾個較常出現、且在閱讀時可能需要先釐清的名詞。

本文件不試圖解釋整個系統的所有術語，只整理與 `start-here`、`ui-workflows`、`mental-model` 閱讀路徑最相關的概念。

## 1. Event 與狀態相關名詞

### `event`

`event` 是 `5g-viz` 最核心的中介資料單位。

它是 parser 將原始 log 轉成的結構化事件，後續會被不同層使用：

- topology 依 event 產生畫面反應
- event log 顯示的就是 event
- 部分 event 會再進一步轉成 metrics
- replay 也是以 session 中保存的 event 為基礎做重建

因此，在本專案裡，event 不是單純「畫面上的一個動畫」，而是整個系統最重要的中介層。

### `state_snapshot`

`state_snapshot` 是後端目前拓樸狀態的快照。

它描述的是「現在這一刻 topology 應該長什麼樣」，而不是完整歷史。

典型用途包括：

- 新的 live client 連上時，先同步目前狀態
- `/api/state` 回傳目前後端認定的 topology 狀態

它不是 event history，也不是 replay。

### `event_reactions`

`event_reactions` 指 topology config 中定義的事件反應規則。

它決定某種 event 進來後，畫面或狀態要做什麼反應，例如：

- `flash_edge`
- `pulse`
- `add_class`
- `remove_class`

它不是 parser rule。  
parser rule 決定 log 會被辨識成哪種 event；`event_reactions` 則決定該 event 之後會怎麼影響 UI 與部分狀態。

## 2. `live`、`replay` 與 `DVR`

### `live`

`live` 指實驗仍在進行中的模式。

在這個模式下：

- 後端仍會持續收到新事件
- topology、event log、Grafana 都可能持續更新
- 前端可以暫停觀看視角，但 backend runtime 仍會繼續往前跑

### `replay`

`replay` 指使用既有 session 重建歷史觀測過程的模式。

在這個模式下：

- 事件來源不是遠端 runtime，而是既有 session 中保存的事件
- 前端看的不是「現在還在發生的實驗」，而是已錄製完成的一段歷史
- topology、event log 與 chart 都是依這段歷史資料重建或播放

### `DVR`

`DVR` 指前端的時間控制機制。

它包含：

- `Pause`
- `Play`
- `Scrub`
- `Go Live`
- speed
- chart window

它的重點不是影片播放器本身，而是讓 topology、event log 與 Grafana 可以切換不同時間視角。

## 3. DVR 狀態機

`DVR` 在前端不是單一按鈕，而是一組狀態機。

常見狀態包括：

- `LIVE`
- `PAUSED`
- `PLAYING`
- `SCRUBBING`

不同模式下，這些狀態的語意會略有不同，但可以先用下面方式理解。

### `LIVE`

`LIVE` 表示前端視角跟著目前最新狀態走。

在 live 模式下，這代表：

- timeline 右邊界會繼續往前推
- 畫面會跟著新事件更新

### `PAUSED`

`PAUSED` 表示前端停在某個時間點。

在 live 模式下，停住的是觀看視角，不是 backend runtime。  
在 replay 模式下，停住的是播放位置。

### `PLAYING`

`PLAYING` 表示前端正在沿時間軸往前播放。

這個狀態在 replay 中最直觀；在 live 中則比較像從某個較早位置一路追到最新時間。

### `SCRUBBING`

`SCRUBBING` 表示使用者正在拖曳 timeline，前端依新的時間位置重建畫面。

它強調的是「跳到某個時間點看畫面」，不是持續播放。

## 4. DVR 常見操作名詞

### `pause`

`pause` 指暫停目前觀看視角。

在 live 中：

- backend 仍持續收事件
- 前端只是暫時不把新事件即時套到目前畫面

在 replay 中：

- 畫面停在 replay 的某個時間位置

### `play`

`play` 指沿目前時間位置往前播放。

在 replay 中，這代表從目前 playhead 繼續播放歷史。  
在 live 中，通常代表從較早的觀看位置一路追到最新時間。

### `scrub`

`scrub` 指透過 timeline slider 跳到某個時間點，並重建該時間點附近的畫面。

### `go live`

`go live` 指把前端視角拉回目前最新狀態。

它只有在 live 模式下有明確意義，因為只有 live 後面還有持續往前跑的最新狀態。

### `chart window`

`chart window` 指 Grafana 目前顯示的時間視窗大小，例如最近幾分鐘。

它控制的是圖表顯示範圍，不等於整個 session 長度，也不等於整個 timeline 範圍。

## 5. Replay 圖表相關名詞

### `backfill`

`backfill` 指 replay 啟動時，先把原始 session 的 metric events 寫回 Prometheus。

它的目的，是讓 Grafana 能直接查到這次 replay session 的歷史時間序列，而不是只能看當前播放點附近的資料。

### `pseudo-live`

`pseudo-live` 指 replay 播放時，將歷史 metric 重新映射到現在時間後持續送入 Prometheus 的做法。

它的目的，是讓 replay `playing` 時的 chart 體驗更接近 live，而不是只看一張固定歷史圖。

### `pre-seed`

`pre-seed` 指 pseudo-live 開始前，先補一段播放點之前的 metrics 到目前 chart window 內。

它的作用是避免剛按播放時，Grafana 一開始只有空白視窗。