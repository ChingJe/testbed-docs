# Hierarchical Federated Learning Component Baseline Update

日期：2026-08-26

狀態：Ready for User Review；最新 component pins、repository tests 與 component-level targeted tests
已完成，parent 更新尚未 commit；testbed config migration 尚未開始

## 1. 目的

在新版 testbed parent repository 的 integration branch 固定目前 explicit Flat／Hierarchical
orchestration、private collection 與 static collected-data verification 所使用的 component revisions，
作為後續 testbed config migration、topology adaptation 與 experiment design 的可重現來源基線。

本階段只更新 component source identity。既有 production Flat runtime config 尚未遷移，新 static
Flat／Hierarchical profiles 亦尚未建立；沒有啟動 VM 或 containers，也不宣稱新版 components 已通過
testbed runtime validation。

## 2. Repository 與 revision

Parent repository：

- branch：`feat/r18-hierarchical-federated-learning`；
- committed checkpoint：`7b9a3e6`；
- 本次 parent diff：兩個 submodule gitlinks、`components.lock.yaml` 與 `compose.yaml` 的 revision
  identity，保持 unstaged／uncommitted。

Component baseline：

| Path／repository | Committed checkpoint | Current target | 本次來源能力 |
| --- | --- | --- | --- |
| `NFs/nwdaf` | `3279891` | `6aed268` | MTLF-private UDM、SMF Event Exposure 與 ADRF storage relays |
| `ML/PyMTLF` | `e9aa223` | `36166f0` | explicit orchestration、private collection、area-scoped provenance 與 dataset evidence |
| `ML/PyAnLF` | `6a4d94a` | `6a4d94a` | consumer-triggered collection baseline；本次無 revision 變更 |
| `nwdaf-resources` | `39ced28` | `8347377` | explicit runners、private collection E2E、static Flat／HFL collected-data scenarios |

`nwdaf-resources` 是獨立 repository，不是 parent submodule pin。其 local branch 已對齊
`origin/feat/r18-hierarchical-federated-learning`。ADRF、NRF、SMF、UDM、UDR、UPF 及其他 NFs／ML
repositories 沒有與本 workstream 相關的新 revision，因此不更新 parent identity。

## 3. 已確認的 source 變更

### 3.1 Explicit orchestration

- autonomous owner 由 `orchestration.mode: flat | hierarchical` 明確選擇；
- participant source 明確區分 production `monitor_scopes` 與 experiment `static`；
- Server／Client engines 的存在不再用來推論 top-level coordinator；
- Flat 與 HFL 共用 federated training request API；
- static Flat 有 exact participant topology，HFL 維持 Root／Branch／Leaf static topology。

### 3.2 Private training-data collection

- data-owning Client／Leaf 可選 `consumer_subscription` 或 `private_api` collection trigger；
- private collection 由 configured Internal Group、network area、DNN、S-NSSAI 與 sampling profile
  建立；
- PyMTLF 經 containing NWDAF relays 解析 UDM／serving SMF、管理 SMF subscription、接收 UPF
  callback，並寫入 ADRF 或 MongoDB；
- collection 維持到 caller DELETE；有 stored data 時保留 descriptor，restart 後可清理 exact
  remote resources；
- dataset artifact 保存 actual collected-data evidence 與 origin provenance。

### 3.3 Local verification runners

- static Flat 使用四個 Clients 與既有 FedAvg；
- HFL 使用 Root、兩個 Branches、四個 Leaves 與既有 FedProx；
- 兩條 local scenario 各自驗證 collection、dataset、aggregation、publication 與 cleanup；
- support UDM／SMF 與 callback replay evidence 不等同 real SMF／UPF／UE testbed validation。

## 4. 驗證結果

| Verification | Result |
| --- | --- |
| Parent `make test` | PASS；包含 syntax、config contracts、dataset、Compose identity、CPU config 與 `vagrant validate` |
| Parent `git diff --check` | PASS |
| PyMTLF targeted tests | PASS；config、collection、Flat coordinator 與 FL Server 共 `193 passed`；一項 dependency deprecation warning |
| NWDAF `go test ./internal/mtlf/...` | PASS |
| VM／container／business E2E | 未執行；屬於後續 testbed migration 與 runtime validation |

第一次在受限環境執行 Vagrant／loopback tests 時分別受到 user Vagrant state 與 local socket sandbox
限制；允許測試讀取 Vagrant state／開啟 loopback socket 後通過，沒有修改 VM 或 service lifecycle。

## 5. 已確認的 testbed config gap

Parent repository tests 驗證的是 testbed-owned config contract，尚未使用新版 PyMTLF typed loader。
以 `36166f0` 原生 loader 檢查目前三份 default PyMTLF config 時均被拒絕：

- legacy `runtime.mode: fl_client | fl_server` 必須遷移到新版 federated runtime；
- 既有 A／B Client 缺少明確的 `training_data.collection_trigger`；
- 既有 A／B Client 仍在 local training block 保存由 Server-owned request 決定的 `epochs`；
- 既有 C Server 缺少 explicit Flat orchestration 與 training trigger。

這些是 config migration requirement，不是 component source regression，也不能因 parent `make test`
通過而視為已處理。

## 6. 後續工作

後續 testbed migration、static scenario topology 與驗證順序由
[Flat and Hierarchical Testbed Scenario Migration Plan](flat-hierarchical-scenario-migration.md) 管理。
Component baseline 在 parent diff 經 review／commit 後可轉為 record；在 real testbed validation 完成前，
不宣稱新版 Flat 或 HFL 已通過 testbed runtime acceptance。
