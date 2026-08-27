# Static Hierarchical Controlled Flow 實驗定義

日期：2026-08-28

狀態：Completed；real three-VM validation、mandatory review、user review 與 Infrastructure source commit
已完成

## 1. 研究問題與可宣稱邊界

本實驗驗證Phase 2 static Hierarchical deployment能否在同一個operator-provided `RUN_ID`下完成：

```text
4 Leaf private collections
→ 4 retained 2-SUPI snapshots
→ 1 manual Root request
→ exact 2 Branches × 2 Leaves
→ 2 rounds of Leaf FedProx fitting
→ 2 lower-tier sample-weighted Branch aggregates per round
→ 1 upper-tier sample-weighted Root aggregate per round
→ 4-Leaf final validation
→ ADRF and Root catalog publication
→ exact 2 upper + 4 lower resource cleanup
```

成功只代表這條static Hierarchical lifecycle、identity、two-tier aggregation、publication與cleanup contract在指定
revisions和三台VM testbed上閉合。不宣稱model-quality benchmark、communication cost、production throughput、
HFL優於Flat，或FedAvg／FedProx及topology差異的單一因果效果。

Hierarchy assignment duplicate GET、transport optimization與component remediation不屬於本實驗的functional
acceptance，也不阻塞本實驗執行。

## 2. Source-confirmed contract

目前pinned source固定下列直接契約：

- Root使用既有`POST／GET /internal/v1/federated-learning/training-requests`，回應
  `mode=hierarchical`、`participantSource=static`與`triggerSource=private_api`；沒有HFL-only top-level API。
  「Manual Root request」指operator主動發送request，wire value不是`manual`。
- Root是唯一top-level coordinator；generated topology和NRF discovery必須exact解析兩個Branches，每個Branch
  exact解析兩個Leaves，`complete_required`不得縮小participant set。
- Branch同時使用upstream Client與lower-tier Server engine，但不擁有training data、不建立private collection，
  也不自主建立另一個Root request。
- 四個Leaves沿用private collection API，各自只解析一個Internal Group與兩個互斥SUPI。
- 每輪四個Leaves執行FedProx local fitting；兩個Branches先依positive sample counts聚合，再由Root依兩個
  Branch effective sample counts聚合。
- Root成功路徑建立兩個upper resources；兩個Branches合計建立四個lower resources。
- Final validation涵蓋四個Leaves；validated candidate由Root發布至ADRF並commit durable catalog。
- Static manual request沒有Model Monitor scopes，因此成功時`required_scopes=0`，不執行PyAnLF reprovision或
  generation cutover。

Component source現有直接runtime logs可證明Root request acceptance／failure、hierarchy preparation dispatch、
Leaf local result／validation、publication及upper／lower resource create／DELETE。Top-level state與round progress由
private request API擁有；Branch／Root aggregate identity、sample weighting與lineage由artifact metadata擁有，
不能從缺少對應event的log推測。

## 3. 權威輸入

| 輸入 | 契約 |
| --- | --- |
| Deployment | `testbed.static-hierarchical.yaml`；完整1 Root／2 Branches／4 Leaves／8 UE／7 PyMTLF source |
| Scenario | `experiments/examples/fl-closure-smoke/scenario.yaml`；2 local epochs、2 rounds、300-second closure budget |
| Generated runtime | 一個新`CONFIG_DIR`；manifest、native configs、Compose與topology由同一render產生 |
| Run identity | operator產生並保存的canonical lowercase UUIDv4 `RUN_ID` |
| Model family | selected Root native config；預期唯一seed family為`ue-communication-default` |
| Data partitions | manifest positions 1–4；每個Leaf恰有兩個SUPI，八個SUPI全域互斥 |
| Hierarchy | generated topology＋current NRF discovery；exact 2 Branches及每Branch exact 2 Leaves |

不建立host ledger、current-run pointer、第二份endpoint／profile／topology inventory或新的scenario source。
Generated config、dataset、containers、volumes、runtime ledgers、workspaces與run evidence都是runtime artifacts。

## 4. 必要identity snapshot

每個real run在第一個mutating request前保存：

- Infrastructure parent revision、所有component revisions及各repository dirty flag；
- selected `TESTBED`、scenario path／hash、`CONFIG_DIR` tree hash與manifest generated identity；
- Compose image IDs、OCI revision labels、container config-set／config-hash labels與accelerator mapping；
- dataset manifest／profile hashes；
- Host CPU、`MemAvailable`、disk、GPU／VRAM與selected capacity gate result；
- Host OS provider process inventory、declared VM identities、provider state與Guest active config hash；
- 七個NWDAF NRF identities、八個UE Registration／PDU Sessions與七個PyMTLF readiness。

Real provider observations只能在approved sandbox-outside Host context執行，第一個runtime observation必須是Host OS
process inventory，不得先執行Vagrant／VirtualBox query。

## 5. 標準時間線

### 5.1 Runtime準備

1. Render並validate新的static Hierarchical `CONFIG_DIR`；驗證dataset、capacity與component lock。
2. Exact核對三台VM、Host provider process inventory及selected／active identity後啟動runtime。
3. 確認29個Guest units、八個UE sessions、七個unique NWDAF registrations與七個healthy PyMTLF containers。
4. 確認static deployment沒有啟動production Consumer subscription chain。

### 5.2 Leaf collection

```text
make fl-collection-start TESTBED=testbed.static-hierarchical.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make fl-collection-status TESTBED=testbed.static-hierarchical.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make fl-collection-stop TESTBED=testbed.static-hierarchical.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
```

Start前exact核對四個Leaf health、manifest owners、profiles、Internal Groups及4×2 SUPI ownership。Root與Branches
不得收到collection request。每個Leaf至少達到scenario `minimumSamples`，且兩個SUPI都必須有current-run real
SMF／UPF observations。Stop只有在四份resources同時滿足下列條件時成功：

- `state=RETAINED`；
- `descriptorState=RETAINED`；
- `activePeerResourceCount=0`；
- `pendingCleanupPeerResourceCount=0`；
- `cleanupPending=false`。

### 5.3 Root training與publication

```text
make fl-training-start TESTBED=testbed.static-hierarchical.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make fl-training-status TESTBED=testbed.static-hierarchical.yaml CONFIG_DIR=<dir> RUN_ID=<uuid>
make ml-status TESTBED=testbed.static-hierarchical.yaml CONFIG_DIR=<dir>
```

Training start必須拒絕non-retained、cleanup未完成、資料不足、expired descriptor、wrong config、unhealthy Root、
family mismatch或非exact topology。成功run保存：

- top-level request identity、`mode=hierarchical`、`participantSource=static`、state、round progress及candidate digest；
- Root accepted request與plan identity、exact兩Branch／四Leaf discovery及preparation outcomes；
- Root建立的兩個upper resource locations，以及Branches建立的四個lower resource locations；
- 每輪四份positive-sample Leaf local artifacts、兩份Branch aggregates與一份Root aggregate的direct artifact metadata；
- subordinate identities、positive sample counts、aggregate digest與parent／round lineage exact一致；
- 四份Leaf final validation、per-Branch與aggregate summary、`gate_would_accept`及rejection reasons；
- ADRF instance／transaction／resource identity、published model ID、final artifact digest與Root catalog generation；
- `required_scopes=0`及terminal `COMPLETE`；
- 兩個upper與四個lower create／terminal DELETE一對一，沒有cleanup failure或stale current-run state。

### 5.4 Retention、stop與reset

取得完整evidence後才執行ordinary `experiment-stop`。Ordinary stop保留descriptors、ADRF model、Root catalog、
artifacts、containers、volumes、datasets與VM。Guarded reset必須exact清除七個ML volumes，以及selected topology的
collection、training、hierarchy-plan、publication與ADRF state；下一次Root啟動後恢復canonical seed identity，
其他topology assets不得被清除。

## 6. Failure與restart矩陣

| 案例 | 直接證據 | 必要結果 |
| --- | --- | --- |
| Missing／invalid `RUN_ID` | command stderr；HTTP fixture count | 任何API call前失敗 |
| Wrong topology／config identity | manifest／container labels | fail closed；不發mutating request |
| Invalid Leaf owner inventory | manifest、native profiles與topology | Root／Branch不被當data owner；不發collection |
| Partial collection create | per-Leaf POST／DELETE trace | 只rollback本次exact identity；列出未收斂資源 |
| Insufficient／expired／ambiguous retained data | counts、stop time、descriptor inventory | 禁止training；不fallback或猜測snapshot |
| Ambiguous Root POST | timeout＋same-ID GET／retry | 不換request identity、不建立第二個plan |
| Branch／Leaf discovery mismatch | NRF／topology identity | complete-required terminal failure；不得縮小topology |
| Partial hierarchy preparation | per-node outcomes與resource trace | 保存outcome並cleanup已建立upper／lower resources |
| Leaf fitting／lower aggregation timeout | state、logs與artifacts | terminal failure；不得使用partial／stale aggregate |
| Missing／conflicting Branch aggregate | artifact contract | Root拒絕upper aggregation |
| Final validation failure | Leaf／Branch validation artifacts | 不publication、不更新catalog |
| Publication retry／failure | journal、ADRF與catalog identity | retry保持in-progress；terminal failure不冒充published |
| Root／Branch／Leaf generation change | process generation與request／resource inventory | fence舊generation；不得把stale plan當新run |
| Cleanup evidence不足 | API／logs／inventory contradiction | 即使top-level `COMPLETE`仍為verification-incomplete |
| Guarded reset | exact scope與seed identity | HFL state清空、其他topology保留、seed確定性恢復 |

## 7. Run evidence範本

```text
RUN_ID:
STARTED_AT / ENDED_AT:

REVISIONS_AND_DIRTY_FLAGS:
SELECTED_TESTBED_SCENARIO_CONFIG_HASH:
DATASET_AND_IMAGE_IDENTITIES:
HOST_CAPACITY_AND_PROVIDER_PROCESS_INVENTORY:
GUEST_ACTIVE_CONFIG_AND_RUNTIME_INVENTORY:

LEAF_1 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:
LEAF_2 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:
LEAF_3 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:
LEAF_4 SUPIS / COLLECTION_COUNTS / RETAINED_STATUS:

ROOT_REQUEST / PLAN / API_STATUS:
BRANCH_AND_LEAF_ADMISSION:
UPPER_2_AND_LOWER_4_RESOURCE_LOCATIONS:
ROUND_1 LEAF_RESULTS / BRANCH_AGGREGATES / ROOT_AGGREGATE:
ROUND_2 LEAF_RESULTS / BRANCH_AGGREGATES / ROOT_AGGREGATE:
FINAL_VALIDATION_AND_GATE_WOULD_ACCEPT:
ADRF_PUBLICATION_AND_ROOT_CATALOG:
UPPER_LOWER_CREATE_DELETE_AND_CLEANUP:

FAILURE_RESTART_STOP_RESET_EVIDENCE:
STATIC_FLAT_REGRESSION:
UNEXECUTED_LAYERS_AND_LIMITATIONS:
```

## 8. 驗收條件

實驗只有在Phase 4 plan的completion criteria全部具備direct evidence、repository full checks與Phase 3 static Flat
regression通過、mandatory initial review及user review完成後才通過。上述criteria已完成，
evidence保存於
[Static Hierarchical controlled-flow validation record](../records/hierarchical-federated-learning/static-hierarchical-controlled-flow-validation-2026-08-28.md)。
Synthetic、fixture或container-only evidence不能替代real三VM SMF／UPF collection、NRF hierarchy admission、
two-tier training、ADRF publication與cleanup。
