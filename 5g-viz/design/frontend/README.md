# Frontend

本目錄收錄 `5g-viz` 前端行為的 canonical 文件，聚焦於拓樸渲染、事件播放控制與 Grafana 嵌入。

適合的情況包括：

- 需要查 Topology、event buffer、timeline、scrub、playback 的實際前端行為
- 需要確認 Grafana iframe 與 DVR controls 的同步邏輯

若目前需求是先理解畫面區塊與操作，先讀：

- [`../../guides/ui-workflows/`](../../guides/ui-workflows/README.md)

主要文件：

- [topology.md](topology.md)：`topology.js` 的拓樸初始化、`event_reactions` 解讀、filter 與靜態快照重建
- [events-and-dvr.md](events-and-dvr.md)：`events.js` 的 session bootstrap、事件來源、timeline、scrub 與播放控制
- [grafana-embed.md](grafana-embed.md)：Grafana iframe 的 URL 組合、session 切換與時間窗口同步
