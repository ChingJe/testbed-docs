# Legacy 5G Infrastructure Documents

本分類保存舊 `5G_Infrastructure` testbed 與相關歷史資料，不代表目前
`5G_NWDAF_Infrastructure` 的架構或 component revisions。

主要內容：

- `design/`：Daisy、舊 NWDAF／ML、UPF EES 與 replay integration 設計；
- `ops/`：舊 VM 與實驗室環境的操作、setup、network 及 troubleshooting；
- `experiments/`：舊 `exp1–67` replay／training experiment definitions；
- `reports/`：舊 testbed、Daisy、UPF、replay、migration 與 site inventory reports；
- `testbed-runs/`：保留下來的舊 run logs 與 configs。

新版 Infrastructure 原本位於本分類下的 foundation plan 與 validation reports，已依文件責任
移到 repository 根目錄的 `5g-nwdaf-infrastructure/`。後續不再把新版 plans 或 records 加回
此目錄。

閱讀舊資料時應同時確認日期、實際 repository revision、dirty state、VM identity 與 site-specific
network assumptions。文件中提到的 commands、branches 或 architecture 不應直接套用到新版 testbed。
