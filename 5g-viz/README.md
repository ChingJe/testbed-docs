# 5g-viz

`5g-viz` 是 5G testbed 的觀測與重播介面。

主畫面主要由三個觀測面構成：

- topology：哪些 NF 正在互動、哪些流程正在發生
- event log：剛剛發生了哪些結構化事件
- Grafana：哪些 traffic / prediction / deviation metrics 正在變化

系統同時支援兩種執行情境：

- `live`：實驗仍在進行，topology、event log、Grafana 會對當前 runtime 持續更新
- `replay`：實驗已錄製完成，畫面以既有 session 重建與回放當時的過程

第一次閱讀建議不要先從 `design/*` 開始。

## 建議閱讀順序

1. [Guides](./guides/README.md)
2. [What Is 5g-viz](./guides/start-here/what-is-5g-viz.md)
3. [Terminology](./guides/start-here/terminology.md)
4. [Live Vs Replay](./guides/start-here/live-vs-replay.md)
5. [Screen Tour](./guides/start-here/screen-tour.md)
6. [UI Workflows](./guides/ui-workflows/README.md)
7. [Mental Model](./guides/mental-model/README.md)
8. [Common Scenarios](./guides/troubleshooting/common-scenarios.md)
9. [Reference](./reference/README.md)

上述路徑讀完後，再依需求進入 deeper reference：

- 想看系統與資料路徑：[`design/overview/`](./design/overview/README.md)
- 想看 DVR / replay：[`design/dvr/`](./design/dvr/README.md)
- 想看前端行為：[`design/frontend/`](./design/frontend/README.md)
- 想看 backend / state / metrics：[`design/backend/`](./design/backend/README.md)

## 這個目錄怎麼看

- [`guides/`](./guides/README.md)：給讀者的 human-facing 入口層
- [`guides/start-here/`](./guides/start-here/README.md)：第一次接觸這個系統時的入口
- [`guides/ui-workflows/`](./guides/ui-workflows/README.md)：依畫面區塊與操作流程理解系統
- [`guides/mental-model/`](./guides/mental-model/README.md)：解釋 event、snapshot、metrics 與 live / replay 的概念邊界
- [`guides/troubleshooting/`](./guides/troubleshooting/common-scenarios.md)：整理常見現象與最常見解釋
- [`reference/`](./reference/README.md)：deep reference 導覽與主題索引
- [`design/`](./design/README.md)：深層設計與 implementation reference
- [`plans/`](./plans/README.md)：歷史規劃與設計探索
- [`notes/`](./notes/README.md)：實作 / 會議 / 內部筆記，不是 onboarding 主入口

## 三個基礎前提

1. topology、event log、Grafana 看到的不是同一條資料路徑。  
   它們彼此相關，但不是同一層的不同畫法。

2. `live` 與 `replay` 長得很像，但底層資料流不一樣。  
   replay `paused` 與 replay `playing` 現在都查原始 session，但 chart 在兩種狀態下仍不是同一種時間窗語意。

3. 這套系統的可攜 replay 核心不是 Prometheus TSDB，而是 session 目錄。  
   實際保存的是 `meta.json`、`events.jsonl`、`topology.yaml` 這三個檔案。
