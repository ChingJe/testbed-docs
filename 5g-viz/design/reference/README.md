# Reference

本目錄收錄設定與 schema 參考文件，聚焦於 topology config 與 profile 環境變數。

這一層處理的是：

- config schema
- env / profile 契約
- cross-layer 設定欄位的實際格式

若需要整個 deep reference 的主題索引，先看：

- [`../../reference/README.md`](../../reference/README.md)

主要文件：

- [topology-yaml.md](topology-yaml.md)：`topology.yaml` 的頂層區塊、`ssh_sources`、`event_reactions` 與 cross-layer 契約
- [env-config.md](env-config.md)：profile `.env`、runtime env 與目前實際有被程式消費的環境變數
