# Grafana

> 注意：本子目錄仍混有 pre-refactor 文件，部分內容會提到 pseudo-live、`start.sh` 與舊 replay rendering 模型。現況請先以 guides 與 `5g-viz/README.md` 為準。

本目錄收錄 Grafana 與 Prometheus 整合行為的技術文件，聚焦於 dashboard 建立、資料來源與查詢渲染語意。

適合的情況包括：

- 需要查 dashboard setup、panel query、datasource 與 rendering 語意
- 需要確認 live / replay 在 Grafana 層的差異，以及歷史 pseudo-live 模型

若目前需求是先理解主畫面 chart 的用途與行為，先讀：

- [`../../guides/ui-workflows/grafana.md`](../../guides/ui-workflows/grafana.md)
- [`../../guides/mental-model/events-snapshots-metrics.md`](../../guides/mental-model/events-snapshots-metrics.md)

主要文件：

- [setup.md](setup.md)：Grafana / Prometheus 的環境前提、datasource 建立與 dashboard 生成流程
- [rendering.md](rendering.md)：panel query、`session` variable、annotations 與 live / replay 圖表語意（含歷史 pseudo-live 背景）

相關前端嵌入行為見：

- [../frontend/grafana-embed.md](../frontend/grafana-embed.md)
