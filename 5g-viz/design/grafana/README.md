# Grafana

本目錄收錄 Grafana 與 Prometheus 整合行為的 canonical 文件，聚焦於 dashboard 建立、資料來源與查詢渲染語意。

適合的情況包括：

- 需要查 dashboard setup、panel query、datasource 與 rendering 語意
- 需要確認 live / replay / pseudo-live 在 Grafana 層的差異

若目前需求是先理解主畫面 chart 的用途與行為，先讀：

- [`../../guides/ui-workflows/grafana.md`](../../guides/ui-workflows/grafana.md)
- [`../../guides/mental-model/events-snapshots-metrics.md`](../../guides/mental-model/events-snapshots-metrics.md)

主要文件：

- [setup.md](setup.md)：Grafana / Prometheus 的環境前提、datasource 建立與 dashboard 生成流程
- [rendering.md](rendering.md)：panel query、`session` variable、annotations 與 live / replay / pseudo-live 的圖表語意

相關前端嵌入行為見：

- [../frontend/grafana-embed.md](../frontend/grafana-embed.md)
