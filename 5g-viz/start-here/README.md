# Start Here

本目錄收錄 `5g-viz` 的第一層導覽文件。

內容重點是主畫面、執行模式、操作流程與可觀察現象。`design/*` 仍保留為 deep technical reference。

## 閱讀順序

1. [What Is 5g-viz](./what-is-5g-viz.md)  
   說明系統定位、解決的問題，以及 topology、event log、Grafana 三個主要觀測面。

2. [Live Vs Replay](./live-vs-replay.md)  
   說明兩種模式在資料來源、時間語意、畫面行為與圖表行為上的差異。

3. [Screen Tour](./screen-tour.md)  
   依畫面區塊說明各部分的角色與用途。

## 後續閱讀方向

- 畫面區塊與操作流程：[`../ui-workflows/`](../ui-workflows/README.md)
- 概念邊界與資料路徑：[`../mental-model/`](../mental-model/README.md)
- 常見現象與解釋：[`../troubleshooting/common-scenarios.md`](../troubleshooting/common-scenarios.md)
- 系統組件與端到端資料流：[`../design/overview/`](../design/overview/README.md)
- pause / replay / pseudo-live：[`../design/dvr/`](../design/dvr/README.md)
- topology / DVR / Grafana iframe 的實作行為：[`../design/frontend/`](../design/frontend/README.md)
- state、API、metrics 與 session 行為：[`../design/backend/`](../design/backend/README.md)

## 本層不展開的內容

- 不先從 function、handler、state machine 名稱開始
- 不把 `main.py`、`events.js`、`topology.js` 的實作順序直接搬成章節骨架
- 不在第一層就展開 remote write、pre-seed、rule registry、protobuf 這類低層細節
