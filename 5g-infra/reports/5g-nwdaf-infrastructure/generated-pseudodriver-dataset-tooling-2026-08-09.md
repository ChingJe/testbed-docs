# Generated PseudoDriver Dataset Tooling

日期：2026-08-09

## 結果

Infrastructure 已能從 topology 與 committed traffic profiles 產生、稽核及規劃分發
Path A/B PseudoDriver Parquet。產物位於 ignored `.generated/`，不提交、不隨 Vagrant
rsync，也不依賴 `nwdaf-resources` 或 go-upf 內建 `pre_data`。

| Path | UE IP | 模式 | Rows | Bytes | Dataset time |
| --- | --- | --- | ---: | ---: | --- |
| A | `10.60.0.1`–`10.60.0.3` | boundary 後 degraded | 27,000 | 841,634 | `0..4499s` |
| B | `10.61.0.1`–`10.61.0.3` | boundary 後 stable | 27,000 | 841,634 | `0..4499s` |

本次 dataset set ID 為
`3cc771b6d283ceee5927e3986dbe1920039e72ce69575389c10556a82a8be4a2`。
這個 ID 包含 generator source、profile、topology-derived UE IP 與目前 ML/monitor contract；
後續任一輸入變更都應得到新 ID，因此文件不把它當作永遠固定的名稱。

## Contract

- boundary：`3000s`，提供 100 個 30 秒歷史觀測；
- model preparation：`seq_length=30`、`out_seq_len=1`、validation ratio `0.1`，最少需要
  62 個 observation；目前可得到 36 個 training 與 4 個 validation samples；
- stable lead-in：`600s`，高於 5 個 reference reports × 90 秒；
- degraded tail：`900s`，高於 3 hits × 90 秒；
- profile 不保存實際 IP，IP 由 Path `uePool` 與三個 UE 推導。

## 驗證

已通過：

- `make config-check`；
- alternate rendered config 的 `config-render` + `config-check`；
- `make dataset-generate`、`dataset-check`、`dataset-show`；
- 兩次獨立生成 manifest 完全相同；
- 對 Path A Parquet 附加資料後，checker 正確拒絕；
- `make dataset-stage-plan` 正確將 A/B 對應到 `path-a`/`path-b`；
- Python compile、Go format/build、shell syntax、`git diff --check`。
- 完整 `make preflight`：0 failures；僅保留未選 provider 與低 free swap 兩項既知 warning。

本輪沒有執行 `dataset-stage apply`、啟動 VM、建立 subscription 或量測 replay RAM。
下一個 runtime gate 應先在已關閉服務的 Path VM staging，接著以真實 PDU Session IP、
UPF log 與 callback payload 確認 dataset row 確實被匹配。
