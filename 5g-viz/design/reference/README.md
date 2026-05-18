# Reference

> 注意：本子目錄混合了現況 schema reference 與少量 migration-oriented 文件。現況請優先以 `backend/profiles.md`、`topology-yaml.md` 與 `5g-viz/README.md` 為準。

本目錄收錄設定與 schema 參考文件，聚焦於 topology config、profile config 與跨層欄位語意。

這一層處理的是：

- config schema
- env / profile 契約
- cross-layer 設定欄位的實際格式

若需要整個 deep reference 的主題索引，先看：

- [`../../reference/README.md`](../../reference/README.md)

主要文件：

- [topology-yaml.md](topology-yaml.md)：`topology.yaml` 的頂層區塊、`event_reactions` 與 cross-layer 契約
- [env-config.md](env-config.md)：用舊 `.env` 名稱對照今天的 `config.yaml` 欄位；偏 migration reference
