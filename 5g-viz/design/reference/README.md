# Reference

> 注意：本子目錄中 `env-config.md` 等文件主要記錄 pre-refactor 的 `.env` / `start.sh` 設定模型。現況已改為 `config.yaml` + `run.py`，因此這裡應優先當 historical reference 閱讀。

本目錄收錄設定與 schema 參考文件，聚焦於 topology config 與舊設定模型。

這一層處理的是：

- config schema
- env / profile 契約
- cross-layer 設定欄位的實際格式

若需要整個 deep reference 的主題索引，先看：

- [`../../reference/README.md`](../../reference/README.md)

主要文件：

- [topology-yaml.md](topology-yaml.md)：`topology.yaml` 的頂層區塊、`event_reactions` 與 cross-layer 契約
- [env-config.md](env-config.md)：歷史的 profile `.env` / runtime env 設定模型
