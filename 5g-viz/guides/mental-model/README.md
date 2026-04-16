# Mental Model

本目錄收錄 `5g-viz` 的概念層文件。

這一層處理的不是單一畫面區塊，而是跨區塊的共通問題：

- event、snapshot、metrics 分別代表什麼
- live 與 replay 為什麼畫面相似，但資料路徑不同
- 哪些 event 只影響 UI，哪些也會進 Prometheus / Grafana
- 某些現象為什麼是設計結果，不是 accidental mismatch

## 閱讀順序

1. [Events, Snapshots, Metrics](./events-snapshots-metrics.md)  
   說明 topology、event log、Grafana 各自依賴的資料層。

2. [Live Vs Replay Data Paths](./live-vs-replay-data-paths.md)  
   說明兩種模式在事件來源、歷史重建與 chart 路徑上的差異。

3. [UI-only Vs Metric Events](./ui-only-vs-metric-events.md)  
   說明哪些事件只改變畫面，哪些事件也會進 metrics。

## 與其他層的分工

- `start-here/`：先建立整體定位與畫面概念
- `ui-workflows/`：先理解各區塊與操作流程
- `mental-model/`：開始回答「為什麼會這樣」
- `troubleshooting/`：把常見現象整理成可直接查找的問題清單

## 對應的 deeper reference

- event / DVR / history buffer：[`../../design/frontend/events-and-dvr.md`](../../design/frontend/events-and-dvr.md)
- state snapshot：[`../../design/backend/state.md`](../../design/backend/state.md)
- metrics：[`../../design/backend/metrics.md`](../../design/backend/metrics.md)
- replay / pseudo-live：[`../../design/dvr/replay.md`](../../design/dvr/replay.md)
