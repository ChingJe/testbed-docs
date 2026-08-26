# Phase 1 Production Flat Config Migration Detailed Plan

日期：2026-08-26

狀態：Completed；Infrastructure commit `072748b`；implementation、testbed runtime acceptance 與獨立
commit gate 全部通過

本次實作與驗收結果見
[Production Flat Config Migration Runtime Acceptance — 2026-08-26](../../records/flat-federated-learning/validation/production-flat-config-migration-runtime-acceptance-2026-08-26.md)。

## 1. 目的

讓既有 three-NWDAF production Flat deployment 在 pinned 新版 NWDAF／PyMTLF components 上使用有效的
config schema，同時保持原本的 participant discovery、collection trigger、degradation trigger、network
placement 與 business flow。

本 Phase 是 configuration compatibility 與 regression 工作，不是 static Flat 或 HFL implementation。
完成後應得到一條可作為後續新 topology 對照基線的新版 production Flat flow。

## 2. 進入條件

- parent branch：`feat/r18-hierarchical-federated-learning`；
- parent starting checkpoint：`8311883`；
- target NWDAF：`6aed268`；
- target PyMTLF：`36166f0`；
- PyAnLF 維持：`6a4d94a`；
- component pin baseline 已由獨立 commit `8311883` 完成；Phase 1 config implementation 使用新的獨立
  diff／commit；
- 不使用 parent `make test` 通過取代 PyMTLF native config validation。

若 target revisions 在 implementation 前移動，先重新檢查 config schema 與 component tests，再更新本
Phase baseline。

## 3. 必須保持的既有行為

| Deployment | 既有責任 | Phase 1 保持方式 |
| --- | --- | --- |
| NWDAF-A／PyMTLF-A | analytics provider、model consumer、FL Client | 維持 Client；collection 由 consumer subscription 間接觸發 |
| NWDAF-B／PyMTLF-B | analytics provider、model consumer、FL Client | 維持 Client；collection 由 consumer subscription 間接觸發 |
| NWDAF-C／PyMTLF-C | model owner、Model Provision provider、Model Monitor coordinator、FL Server | 維持 production Flat autonomous coordinator |

固定不變：

- three-NWDAF／two-Client／one-Server topology；
- A／B／C 的現有 IP、port、callback、ADRF／MongoDB 與 workspace paths；
- participant source 仍為 Model Monitor scopes；
- training trigger 仍為 accuracy degradation；
- A／B training-data collection 仍由 analytics consumer subscription chain 觸發；
- Model Provision、Model Monitor、validation、publication 與 generation cutover 的既有 ownership；
- 現有 scenario stimulus 與 full-core acceptance 語意。

## 4. 已確認的 schema gap

使用 target PyMTLF native loader 檢查目前 default configs，已確認：

- A／B／C 的 legacy `runtime.mode: fl_client | fl_server` 不再有效；
- A／B 缺少 `federated_learning.client.training_data.collection_trigger`；
- A／B 的 client-local fitting config 帶有不再接受的 `epochs`；
- C 是 Server-only profile，但缺少 explicit autonomous orchestration；
- C 缺少明確的 degradation／private training trigger selection。

Parent repository tests 目前未捕捉這些錯誤，因此 Phase 1 必須補上 pinned PyMTLF schema validation。

## 5. Target config contract

### 5.1 PyMTLF-A／B

- `runtime.mode` 遷移為 `federated`；
- 保留 FL Client engine；
- 明確設定 `training_data.collection_trigger: consumer_subscription`；
- 保留目前允許且仍由 Client 擁有的 fitting parameters；
- 移除 Client 不再擁有的 `epochs`，不得以 unknown-field tolerance 繞過 strict validation；
- 不配置 autonomous orchestration、topology 或 training trigger。

### 5.2 PyMTLF-C

- `runtime.mode` 遷移為 `federated`；
- 保留 FL Server engine；
- 明確設定 `orchestration.mode: flat`；
- 明確設定 `orchestration.participant_source: monitor_scopes`；
- 明確啟用 degradation training trigger 並停用 private API training trigger；
- 不配置 static topology 或 hierarchical strategy；
- 將需要由 Server Training Subscribe 決定的 client training effort 放入 Server-owned
  `client_training` contract，保持現有 intended epochs，不靜默採用新 default。

## 6. 預計修改範圍

主要候選檔案：

- `config/default/pymtlf-a.yaml`；
- `config/default/pymtlf-b.yaml`；
- `config/default/pymtlf-c.yaml`；
- `scripts/host/config-render.py`；
- `scripts/host/config-check.py`；
- config／renderer／repository test fixtures；
- 必要時的 scenario-specific generated config assertions。

Implementation 前先以實際 code search 確認所有 PyMTLF config producers／consumers；本清單不是修改未列
檔案的預先授權。若需要改 Compose topology、VM definition、NF placement 或 component source，停止本
Phase 並回報 scope expansion。

## 7. Implementation slices

### Slice 1. Characterization and ownership mapping

- 保存三份 current rendered PyMTLF configs 與 config identity；
- 列出 current renderer 對 A／B／C 欄位的 ownership；
- 用 pinned PyMTLF loader 重現並固定既有 rejection evidence；
- 確認原本 `epochs: 18` 實際由哪個 request builder 使用，避免只為通過 schema 而改變 training effort。

### Slice 2. Minimal config migration

- 只修改第 5 節列出的必要欄位；
- 同步 default configs、renderer 與 generated fixtures；
- 不新增 static topology、private collection profile 或第四個 Client；
- 讓三份 rendered configs 通過 PyMTLF native loader。

### Slice 3. Validation hardening

- 在 repository tests 加入 pinned source schema validation；
- 加入 production Flat mode／participant source／trigger ownership assertions；
- 保留 existing config hash、Compose identity、device policy 與 Vagrant guards；
- 對 legacy runtime mode、缺 collection trigger、Server-only 無 orchestration 建立 deterministic rejection
  cases。

### Slice 4. Host and process smoke

- render default 與 CPU smoke configs；
- 驗證 Compose config、artifact build identity 與 service startup config；
- 啟動前記錄 exact revisions、dirty flags 與 rendered config hash；
- 不在 config／repository checks 尚未通過時啟動 VM 或 services。

### Slice 5. Existing production Flat runtime regression

- 使用原有 A／B／C topology 與 scenario stimulus；
- 驗證 Model Provision、Monitor subscription、accuracy reports 與 degradation decision；
- 驗證 participant scopes 仍由 monitor flow 產生；
- 驗證 A／B consumer-triggered collection、federated training、weighted aggregation、publication 與
  monitor generation cutover；
- 驗證 cleanup、failure diagnostics 與 stale runtime state handling。

實際 VM／service lifecycle 操作前另行列出 commands、影響與 fresh-state policy；本計畫本身不授權清除
既有 VM、container、volume、ADRF 或 MongoDB state。

## 8. Verification matrix

| Layer | Verification | Gate |
| --- | --- | --- |
| Source identity | component locks、gitlinks、Compose revisions | exact target revisions，無 stale build identity |
| Native config | pinned PyMTLF `load_settings()` 逐份載入 A／B／C | 全部通過，無 ignored unknown field |
| Repository | `make test`、`git diff --check` | PASS |
| Compose | baseline／CPU rendered config 與 `ml-compose-check.py` | PASS，service topology仍為既有 deployment |
| Process | A／B／C PyMTLF startup 與 health | 無 config validation／route ownership error |
| Business flow | existing full-core Flat scenario | collection→training→publication→cutover 完成 |
| Cleanup | subscriptions、workspaces、callbacks、process state | 無非預期 stale active resource |

## 9. Phase completion criteria

只有同時滿足下列條件才完成 Phase 1：

1. Component pin baseline 已獨立 commit。
2. A／B／C default 與 rendered configs 通過 pinned PyMTLF native loader。
3. Parent repository／Compose tests 通過。
4. Existing production Flat process startup 與 full-core business flow 在 exact revisions 上通過。
5. 實際 config hash、artifact identity、commands、environment 與 runtime evidence 已寫入 record。
6. Diff 經 user review，config migration 使用獨立 commit。

若只完成 config／host validation，狀態應標為 `Implementation Verified；Testbed Runtime Pending`，不能提前
進入 Phase 2 或宣稱 production Flat 已恢復完整 acceptance。

## 10. Implementation result

2026-08-26 已連續完成 Slice 1–5：

- A／B／C default configs、renderer 與 strict checks 已遷移至 pinned PyMTLF schema；
- A／B 使用 `federated` Client、`consumer_subscription` collection trigger，C 使用
  `federated` Server、`flat + monitor_scopes` orchestration 與 degradation trigger；
- Server-owned client training effort 維持 `epochs: 18`；
- native config、repository、Compose 與 disposable CPU container lifecycle checks 均通過；
- exact pinned revisions 上的 production Flat full-core flow 已完成 collection、兩輪 FedAvg、publication、
  two-scope adoption、generation cutover 與 post-cutover accuracy；
- teardown 後兩筆 Consumer resources 為 `deleted`、Model Monitor `active=0`、五個 containers 與 23 個
  Guest services 均停止，完成證據保留。

Runtime 另確認兩個 integration hardening 點：Guest 內先前安裝的 NWDAF binary 可能落後 synced source，
因此本輪需做一次 targeted NWDAF rebuild；aggregate ML startup failure 現在會明確呼叫 rollback。PyAnLF
啟動時也觀察到約 15 秒 Model Provision 503 retry window，之後自行恢復並完成全部 business flow；此項
不阻擋本次 acceptance，但應在後續 phase 將 provider business readiness 納入更精確的 startup gate。

Phase 1 的 user review／獨立 commit gate 已由 Infrastructure commit `072748b` 完成。Phase 2 可作為下一個
implementation phase，但本次 completion 不代表 Phase 2 已開始。

## 11. 明確不包含

- 第四個 Flat Client；
- static Flat topology 或 private collection；
- Root／Branch／Leaf processes；
- VM／network capacity expansion；
- FedProx migration；
- Flat／HFL performance comparison；
- destructive runtime cleanup 或 clean rebuild。
