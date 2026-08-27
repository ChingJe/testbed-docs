# Static Flat Controlled Flow 實驗定義

日期：2026-08-27

狀態：Completed；real run、required regression、user review 與 source commits 已完成

## 1. 研究問題與可宣稱邊界

本實驗驗證 Phase 2 static Flat deployment 能否在同一個 operator-provided `RUN_ID` 下完成下列 vertical flow：

```text
4 private collections
→ 4 retained 2-SUPI snapshots
→ 1 manual Server request
→ exact 4-Client, 2-round sample-weighted FedAvg
→ final validation
→ ADRF and Server catalog publication
→ exact training-resource cleanup
```

成功只代表這條 static Flat lifecycle、identity、state 與 cleanup contract 在指定 revisions 和三台 VM testbed 上
閉合。不宣稱 model quality benchmark、production throughput、Flat 優於 HFL，或 FedAvg／FedProx 的公平比較。

## 2. 權威輸入

| 輸入 | 契約 |
| --- | --- |
| Deployment | `testbed.static-flat.yaml`；完整 1 Server／4 Clients／8 UE／5 PyMTLF source |
| Scenario | `experiments/examples/fl-closure-smoke/scenario.yaml`；2 local epochs、2 rounds、300-second closure budget |
| Generated runtime | 一個新 `CONFIG_DIR`；manifest、native configs、Compose 與 topology 必須由同一 render 產生 |
| Run identity | operator 產生並保存的 canonical lowercase UUIDv4 `RUN_ID` |
| Model family | selected Server native config；預期唯一 seed family 為 `ue-communication-default` |
| Data partitions | manifest 中 positions 1–4；每個 owner 恰有兩個 SUPI，八個 SUPI 全域互斥 |

不建立 host ledger、current-run pointer、第二份 endpoint／profile inventory或新的 scenario source。Generated config、
dataset、container、volume、collection ledger 與 run evidence 是 runtime artifacts，不納入 source commit。

## 3. 必要 identity snapshot

每個 real run 在第一個 mutating request 前保存：

- infrastructure parent revision、所有 component revisions與各 repository dirty flag；
- selected `TESTBED`、scenario path／hash、`CONFIG_DIR` tree hash 與 manifest generated identity；
- Compose image IDs、image revision labels、container config-set／config-hash labels與 accelerator mapping；
- dataset manifest／profile hashes；
- Host CPU、`MemAvailable`、disk、GPU／VRAM與 selected capacity gate result；
- Host OS provider process inventory、declared VM identity、provider state與 Guest active config hash。

Real provider observations只能在 approved sandbox-outside Host context執行，且第一個 runtime observation 必須是
Host OS process inventory，不得先執行 Vagrant／VirtualBox query。

## 4. 標準時間線

### 4.1 Runtime 準備

1. Render 並 validate新的 static Flat `CONFIG_DIR`；驗證 dataset、capacity與 component lock。
2. Exact核對三台 VM及 Host provider process inventory後，以既有 lifecycle啟動 runtime。
3. 確認27個 Guest units、八個 UE sessions、五個 unique NWDAF registrations與五個 healthy PyMTLF containers。
4. 確認 static deployment未啟動 production Consumer subscription chain。

### 4.2 Collection

```text
make fl-collection-start TESTBED=testbed.static-flat.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make fl-collection-status TESTBED=testbed.static-flat.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make fl-collection-stop TESTBED=testbed.static-flat.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
```

Start 前必須 exact核對 selected／active config identity、四個 Client health、四個 manifest owners、profile及 4×2
SUPI ownership。Status保存每個 owner 的 state、resolved UE／peer counts、record／observation counts、descriptor與
cleanup欄位。Stop只有在四份 resource均滿足下列條件時成功：

- `state=RETAINED`；
- `descriptorState=RETAINED`；
- `activePeerResourceCount=0`；
- `pendingCleanupPeerResourceCount=0`；
- `cleanupPending=false`。

每個 owner必須至少達到 scenario `minimumSamples`，並以 real SMF／UPF callbacks證明兩個 SUPI均有 current-run
observations。Support callback fixture只可作 deterministic test evidence。

### 4.3 Training 與 publication

```text
make fl-training-start TESTBED=testbed.static-flat.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make fl-training-status TESTBED=testbed.static-flat.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make ml-status TESTBED=testbed.static-flat.yaml CONFIG_DIR=<dir>
```

Training start必須拒絕非 retained、cleanup未完成、資料不足、超出 preparation window、wrong config、unhealthy
coordinator或 family mismatch。成功 run保存：

- top-level request的 `mode=flat`、`participantSource=static`、state、rounds及 candidate digest；
- exact四個 discovered Client identities與四個 standard ML Model Training resource locations；
- 每輪四份 positive sample-count local artifacts及 sample-count-weighted aggregate identity；
- 四份 final validation results、aggregate summary、`gate_would_accept`與 rejection reasons；
- ADRF instance／transaction／resource identity、published model ID、final artifact digest與 Server catalog generation；
- `required_scopes=0`及 terminal `COMPLETE`；
- 四個 create和四個 terminal DELETE的一對一 evidence、無 cleanup failure及無 stale current-run resource。

### 4.4 Retention、stop 與 reset

取得完整 evidence後才執行 ordinary `experiment-stop`。Ordinary stop保留 descriptors、ADRF model、catalog、
artifacts、containers、volumes、datasets與 VM。Guarded reset另作 destructive verification；它必須 exact清除本
topology collection／training／publication state，並在下一次 Server啟動後恢復 canonical seed artifact identity。

同一 Client在 preparation window內若同時保有兩個可匹配的 private collection groups，dataset resolver會
fail closed並回報 ambiguous groups；training request schema也刻意沒有可繞過這項檢查的 dataset selector。因此，
失敗後以新 `RUN_ID`重跑或連續執行第二次實驗前，必須先保存既有 evidence，再等待 descriptor TTL清理，或依
`reset-show`精確清單執行 selected guarded reset。不能只建立新 collection、任意挑一份 retained group，或放寬
component ambiguity guard。Operator command能 preflight本次 `RUN_ID`的四份 resource，但現有 API沒有列出其他
retained groups的 inventory；fresh-run程序必須明確管理這個前置狀態。

## 5. Failure 與 restart 矩陣

| 案例 | 直接證據 | 必要結果 |
| --- | --- | --- |
| Missing／invalid `RUN_ID` | command stderr；HTTP call fixture count | 在任何 API call前失敗 |
| Wrong topology／config identity | manifest／container labels | fail closed；不發 mutating request |
| Partial collection create | per-owner POST／DELETE trace | 只 rollback本次已嘗試的 exact owners；列出未收斂資源 |
| Ambiguous collection／training POST | timeout＋same-ID GET／retry trace | 不換 request identity、不建立第二個 run |
| Insufficient observations | per-owner counts | 禁止 training；保留診斷 evidence |
| Collection cleanup pending | DELETE／GET progression | 不宣稱 retained ready |
| Expired descriptor | stop time與 preparation window | 使用新 `RUN_ID`重做 collection |
| Duplicate／conflicting training | 409 problem與 existing resource | 不覆寫 active run或 family |
| Participant／round／validation failure | API state、bounded logs與 artifacts | terminal failure；保存 evidence並清理已建立 resources |
| Publication retry／failure | journal、ADRF及 catalog identity | retry時保持 in-progress；terminal failure不得更新 current catalog |
| Client／Server restart | process generation、request visibility與 resource inventory | fence舊 generation；不得把 stale state當新 run |
| Active-training experiment stop | shutdown logs與 resource inventory | process停止不能取代 exact cleanup evidence |
| Cleanup evidence不足 | API／logs／inventory contradiction | 即使 top-level為 `COMPLETE`仍標為 verification incomplete |

## 6. Run evidence 範本

```text
RUN_ID:
STARTED_AT / ENDED_AT:

REVISIONS_AND_DIRTY_FLAGS:
SELECTED_TESTBED_SCENARIO_CONFIG_HASH:
DATASET_AND_IMAGE_IDENTITIES:
HOST_CAPACITY_AND_PROVIDER_PROCESS_INVENTORY:
GUEST_ACTIVE_CONFIG_AND_RUNTIME_INVENTORY:

OWNER_1 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:
OWNER_2 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:
OWNER_3 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:
OWNER_4 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:

TRAINING_REQUEST_AND_PARTICIPANTS:
ROUND_1_SAMPLE_COUNTS_AND_AGGREGATE:
ROUND_2_SAMPLE_COUNTS_AND_AGGREGATE:
FINAL_VALIDATION_AND_GATE_WOULD_ACCEPT:
ADRF_PUBLICATION_AND_SERVER_CATALOG:
TRAINING_CREATE_DELETE_AND_CLEANUP:

FAILURE_RESTART_STOP_RESET_EVIDENCE:
PRODUCTION_FLAT_REGRESSION:
UNEXECUTED_LAYERS_AND_LIMITATIONS:
```

## 7. 驗收條件

實驗只有在 Phase 3 plan 的 completion criteria 全部具備 direct evidence、repository full checks 與 production
Flat regression 通過、mandatory review 及 user review 完成後才通過。上述 non-Git criteria 已完成，verified
run evidence 保存於
[Static Flat controlled-flow validation record](../records/hierarchical-federated-learning/static-flat-controlled-flow-validation-2026-08-27.md)；
PyMTLF `7479629` 與 Infrastructure `e5b1d44` 保存 reviewed source。Synthetic、container-only 或 component unit
evidence 不能替代 real 三 VM collection、training、ADRF publication 與 cleanup evidence。
