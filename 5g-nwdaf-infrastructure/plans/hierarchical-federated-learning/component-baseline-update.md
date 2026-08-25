# Hierarchical Federated Learning Component Baseline Update

日期：2026-08-25

狀態：Ready for User Review；component pins 與 repository-level verification 已完成，尚未 commit

## 1. 目的

在新版 testbed parent repository 建立獨立 integration branch，固定已完成 local
multi-process verification 的 HFL component revisions，作為後續 testbed architecture
adaptation 與 experiment design 的可重現基線。

本階段不調整 three-VM／three-NWDAF flat topology，不啟動 VM 或 containers，也不宣稱 HFL
已通過 testbed validation。

## 2. 精確範圍

Parent repository：

- base：`main@7d0a36c`；
- branch：`feat/r18-hierarchical-federated-learning`。

Component updates：

| Path | Previous | Target |
| --- | --- | --- |
| `NFs/nwdaf` | `c53f058`／`feat/r18-federated-learning` | `3279891`／`feat/r18-hierarchical-federated-learning` |
| `ML/PyAnLF` | `08798f1`／`feat/r18-federated-learning` | `6a4d94a`／`feat/r18-federated-learning` |
| `ML/PyMTLF` | `7e8ab7f`／`feat/r18-federated-learning` | `e9aa223`／`feat/r18-hierarchical-federated-learning` |

同步更新：

- 三個 parent submodule gitlinks；
- `.gitmodules` 中 NWDAF／PyMTLF branch hints；
- `components.lock.yaml` 中三個 branch／commit entries；
- `compose.yaml` 中兩個 PyAnLF 與三個 PyMTLF `COMPONENT_REVISION` build arguments。

UPF、ADRF、NRF、SMF、UDM、UDR 及其他 submodules 維持既有 revisions。

## 3. 驗證結果

第一次 `make test` 直接指出 `compose.yaml` 還保留舊 PyAnLF／PyMTLF source identities；只更新
五個 `COMPONENT_REVISION` values 後，沒有改變 Compose service topology、ports、device policy
或 runtime config。

| Verification | Result |
| --- | --- |
| `make test` | PASS；包含 syntax、config contracts、dataset、Compose identity、CPU config 與 `vagrant validate` |
| 使用暫存 intended index 的唯讀 preflight | PASS；16 個 component locks 一致，submodule worktrees clean |
| `git diff --check` | PASS |
| Host resources | PASS；唯一 warning 為 free swap `0 MiB < 1024 MiB` |
| VM／container／business E2E | 未執行；不屬於本階段 |

## 4. 後續工作

以下項目是獨立後續 plans，不因本次 component baseline 完成而視為已核准或已驗證：

1. 盤點現有 three-NWDAF／three-VM／five-container deployment 與 HFL requirements；
2. 設計 Root／Branch／Leaf placement、NF capabilities、process composition 與 network／ports；
3. 調整 config、renderer、lifecycle、observability 與 validation tooling；
4. 固定實驗 topology、trigger、stimulus、aggregation、failure／restart 與 evidence matrix；
5. 使用精確 testbed revisions 執行 required runtime validation。

在使用者完成 working-tree review、核准 repository-separated commit proposal 並完成後續 testbed
validation 前，本 workstream 不得標示為 Completed。
