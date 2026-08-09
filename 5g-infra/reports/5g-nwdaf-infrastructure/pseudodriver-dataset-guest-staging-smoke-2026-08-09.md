# PseudoDriver Dataset Guest Staging Smoke

日期：2026-08-09

## 範圍

驗證 Infrastructure generated dataset 可分發到兩台既有 Path VM，並由 Guest 在沒有啟動
任何 5G process 的情況下完成 role-specific verification 與 active symlink 切換。本輪不測
UPF replay、PDU Session、subscription、callback 或 FL。

## 前置狀態

- Provider：既有 VirtualBox VM，以 `VAGRANT_DEFAULT_PROVIDER=virtualbox` 明確選用；
- 初始與結束狀態：`core`、`path-a`、`path-b` 都是 `poweroff`；
- preflight：0 failures、1 warning；warning 是 free swap 約 1 MiB；
- Host：`MemAvailable` 約 29 GiB，workspace 與 Docker filesystem 約 165 GiB free；
- VM 啟動後：三台均為 0 active `5g-nwdaf@*.service`，consumer inactive；
- Vagrant rsync 正常；Guest Additions 6.0 與 Host VirtualBox 6.1 的版本提示未阻擋本輪功能。

## 正向 staging

執行 `make dataset-stage`，set ID 為：

```text
3cc771b6d283ceee5927e3986dbe1920039e72ce69575389c10556a82a8be4a2
```

| Guest | Manifest role | UE IP | Rows | Bytes | SHA-256 |
| --- | --- | --- | ---: | ---: | --- |
| Path A | `path-a` | `10.60.0.1`–`.3` | 27,000 | 841,634 | `4e221ac0b2197be0dad4bbfb20b34f2849d67f8c79d3115169eeb95c613d29da` |
| Path B | `path-b` | `10.61.0.1`–`.3` | 27,000 | 841,634 | `76a887d5b37dbce0710250785756884f9bcf080695be4b0a06bb4f329fa6e0e9` |

兩台 Guest 的實際 `sha256sum` 都等於 manifest，artifact owner 均為
`5g-nwdaf:5g-nwdaf`。`datasets/active` 都指向 guest-local
`datasets/sets/<set-id>`；同一 set ID 在不同 Guest 只包含各自 role artifact。

## Role-isolation negative test

把 Path A archive 暫時上傳到 Path B，並要求 Path B activation。Guest 如預期回報：

```text
dataset identity does not match target machine
```

Activation 失敗後暫存 archive/staging directory 被清理，Path B 的原 active symlink 保持不變。

## 資源與清理

| Guest | MemAvailable after staging | Guest free disk | Dataset disk | Set directories |
| --- | ---: | ---: | ---: | ---: |
| Path A | 2640 MiB | 35,578 MiB | 1 MiB | 1 |
| Path B | 2627 MiB | 35,578 MiB | 1 MiB | 1 |

驗證後以 graceful halt 關閉三台 VM，最終 `vagrant status` 全部為 `poweroff`。Host generated
dataset 保持 ignored；沒有建立 commit、container、subscription 或 experiment run record。

## 尚未證明

- UPF 能載入 active Parquet 並把 rows 匹配到實際 PDU Session IP；
- historical phase 與 live phase 的 30 秒 Event Exposure notification；
- Path A degradation／Path B stable 行為；
- replay scan/aggregation peak RAM 與 512 MiB headroom gate；
- NWDAF → PyAnLF callback、accuracy report、monitor 或 federated retraining。
