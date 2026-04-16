# UI Workflows

本目錄收錄 `5g-viz` 的畫面區塊說明與操作流程文件。

這一層的重點不是實作細節，而是：

- 主畫面各區塊各自代表什麼
- 各區塊在 live / replay 下如何運作
- 常見操作會讓哪些區塊一起變化
- 常見觀察情境該先看哪一塊

## 閱讀順序

1. [Topology](./topology.md)  
   說明 topology 顯示的是什麼、哪些內容是持久狀態、哪些內容是瞬時效果。

2. [Event Log](./event-log.md)  
   說明 event log 的資料來源、用途，以及它與 topology 的關係。

3. [Event History](./event-history.md)  
   說明 event 何時到前端、前端保存多久、scrub 何時直接用本地事件、何時需要向後端補抓。

4. [Grafana](./grafana.md)  
   說明 chart 區塊的 panel、資料層與模式差異。

5. [DVR Controls](./dvr-controls.md)  
   說明 Pause、Play、Scrub、Go Live、Chart Window 的語意與效果。

## 與 `start-here/` 的分工

- `start-here/`：先建立系統定位、模式差異與主畫面概念
- `ui-workflows/`：開始進入各區塊與操作流程
- `mental-model/`：解釋這些畫面現象背後的概念邊界
- `troubleshooting/`：整理最常見的使用情境與誤解
- `design/*`：需要 deeper implementation / runtime / reference 時再往下讀

## 對應的 deeper reference

- topology / filter / snapshot 重建：[`../../design/frontend/topology.md`](../../design/frontend/topology.md)
- events / DVR / timeline：[`../../design/frontend/events-and-dvr.md`](../../design/frontend/events-and-dvr.md)
- Grafana iframe / session / time window：[`../../design/frontend/grafana-embed.md`](../../design/frontend/grafana-embed.md)
- replay / pseudo-live：[`../../design/dvr/replay.md`](../../design/dvr/replay.md)
