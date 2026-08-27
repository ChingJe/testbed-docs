# Experiments

本目錄保存新版 NWDAF testbed 的實驗設計，不保存舊 `5g-infra/experiments/exp*.yaml` replay
時間線。

每個實驗至少應明確記錄：

- research question 與可宣稱的結果；
- topology、component revisions、config identity 與 runtime artifacts；
- trigger、traffic／data stimulus 與時間軸；
- success、failure、timeout、restart 與 cleanup matrix；
- aggregation、publication、cutover 與其他 business evidence；
- 未執行層級、support fakes 與 benchmark 限制。

實際執行完成後，run record 與結果移至 `records/`，本目錄保留核准的實驗定義。

目前定義：

- [Static Flat controlled flow](static-flat-controlled-flow.md)
- [Static Hierarchical controlled flow](static-hierarchical-controlled-flow.md)
