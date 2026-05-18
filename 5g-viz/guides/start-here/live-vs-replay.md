# Live Vs Replay

`live` 與 `replay` 是 `5g-viz` 最容易混淆的兩種模式。

## 核心差異

- `live`：實驗仍在進行，UI 對當前 runtime 做觀測與時間控制
- `replay`：實驗已錄製完成，UI 以既有 session 建立唯讀 runtime

這個差別會影響：

- 事件從哪裡來
- timeline 怎麼長
- 新分頁看到什麼
- Grafana 查的是哪一類資料
- Pause / Play / Go Live 的意義

## 兩種模式的主要特徵

### Live

`live` 模式對應的是仍在進行中的實驗。

主要特徵包括：

- 新事件會持續進來
- topology 會自己動
- event log 會自己往下長
- Grafana 在即時視窗內跟著現在時間更新
- pause / scrub 會改變前端觀看視角，但 backend 不會因此停止收事件

`live` 的關鍵不是畫面顯示即時，而是 backend runtime 本身仍在往前推進。

### Replay

`replay` 模式對應的是已錄製完成的一次實驗。

主要特徵包括：

- timeline 一開始就有明確起點與終點
- 前端不是等新事件推進，而是先載入既有 session 事件
- topology 與 event log 主要是在既有事件上做重建與播放
- Grafana 預設看的是這個 session 在 Prometheus 中的歷史資料

`replay` 的性質比較接近：

> 用當時錄下來的 event 與 topology config，再打開一次同樣的觀測畫面。

## 兩者在哪些地方最不一樣

| 面向 | live | replay |
|---|---|---|
| 事件來源 | 遠端 VM log 持續被 parse 成 event | 既有 `events.jsonl` |
| 前端啟動方式 | 連 `/ws`，之後吃即時事件 | 先載入 `/api/events`，不靠 WebSocket 作為主要資料來源 |
| timeline | 右邊界會持續往前推 | 一開始就有錄製範圍 |
| Pause 的意義 | 暫停的是前端觀看視角，不是 backend runtime | 暫停的是 replay 播放位置 |
| Go Live | 有，因為後面真的還有最新狀態 | 沒有，因為沒有「更即時的現在」可回去 |
| 新分頁 | 先拿目前 `state_snapshot` | 載入這個 replay session 的資料 |

## Pause 的語意在兩種模式下不同

### 在 live

按下 Pause 時：

- topology / event log 會停在某個時間點
- 但 backend 仍然在：
  - tail log
  - parse event
  - 寫 session
  - 更新 metrics
  - 廣播新事件

差別只是前端不再立刻把新事件 dispatch 到目前畫面。

因此 live pause 的語意比較接近：

> 實驗繼續往前跑，但觀看視角暫時停在某個時間點。

### 在 replay

按下 Pause 時：

- 沒有新的實驗正在後面發生
- 畫面停在 replay 的某個位置
- topology 與 event log 會停在那個時間點的重建結果
- Grafana 也切回看那個 replay 時間位置對應的資料

因此 replay pause 的語意比較接近：

> 暫停的是這次回看的播放位置。

## Replay 的 chart 並不是一直用同一條資料

需要先區分兩種情況：

- replay `paused` / `scrubbed` 時，Grafana 看的比較像「原始 session 的固定歷史時間窗」
- replay `playing` 時，Grafana 仍看原始 session，但時間窗改成 historical relative range，讓圖看起來像正在平滑播放

這就是為什麼 replay 看起來像一種模式，但 chart 在不同播放狀態下其實不是完全同一種查詢語意。

## 為什麼新分頁在 live 不會自動重播完整歷史

因為新 live client 進來時，backend 先給的是「目前狀態」的 snapshot，不是完整歷史回放。

新分頁的第一個目標是：

- 快速同步到現在的 topology 狀態

而不是：

- 從 session 開頭把所有動畫重新演一遍

因此，歷史內容的回看依賴 DVR / scrub / event history，而不是新分頁連線時的自動補演。

## 簡化記法

- `live`：後面真的還在長，前端只是在不同時間視角之間切換
- `replay`：後面沒有新資料，前端在既有 session 上做重建、停留與播放

後續閱讀：[Screen Tour](./screen-tour.md)
