# Phase 3 Static Flat Flow Detailed Plan

日期：2026-08-27

狀態：Completed；Slice 1–6 implementation、required verification、user review 與 repository-separated source commits 已完成

最近更新：2026-08-27

## 1. 目的

在 Phase 2 已完成的一 Server／四 Clients、八 UE、五個獨立 NWDAF／PyMTLF runtime foundation 上，跑通
static Flat 的完整 controlled flow：

```text
operator 啟動四份 private collection
→ 四個 Client 各自蒐集兩個 SUPI 的 observations
→ operator 停止 collection 並保留四份 descriptor
→ operator 對 Server 建立 manual training request
→ Server 以 static topology 選出四個 Clients
→ 四個 Clients 使用各自 retained snapshot fitting
→ Server 進行兩輪 sample-count-weighted FedAvg
→ final validation
→ ADRF storage 與 Server model catalog publication
→ exact training-resource cleanup
```

本 Phase 只證明 static Flat 的 collection、training、aggregation、publication 與 cleanup contract 能在實際三台
VM testbed 上閉合，不作 performance benchmark，也不把結果和 Phase 4 的 HFL／FedProx 解讀為純 topology
差異。

本計畫受 [testbed development policy](../../development_policy.md) 約束。Component behavior 以目前 pinned
Go NWDAF、PyMTLF、SMF、UPF 與 ADRF source／tests 為準；本 Phase 預設不修改 component architecture 或 API。

## 2. 已完成基線與 core assumptions

### 2.1 Phase 2 baseline

- Infrastructure baseline：`fc360fc`（manifest-driven static FL topologies）。
- Provider safety baseline：`531e335`（sandbox-outside guard 與 Host process preflight）。
- Static Flat 已在既有 `core`、`path-a`、`path-b` 三台 VM 實際驗證：一個 Server、四個 Clients、八個 UE、
  五個 healthy PyMTLF containers、五個 unique NRF registrations、selected-config lifecycle、exact reset 與
  deterministic seed restoration。
- Phase 2 最終保留三台 VM；experiment Guest units、Consumer 與 ML containers 已停止，datasets、containers、
  volumes、databases 與 canonical seed source 保留。
- Phase 3 開始前仍須重新記錄實際 parent/component revisions、dirty flags、selected config hash、image identity、
  Host capacity 與 current runtime inventory；上述完成證據不代替新 run 的 identity evidence。

### 2.2 Source-confirmed component contract

目前 pinned source 已具備下列 Phase 3 building blocks：

1. Static Flat Server 使用 `orchestration.mode: flat`、`participant_source: static` 與既有 static topology planner；
   不配置 hierarchy strategy，因此沿用 Flat FedAvg。
2. Server private trigger 為
   `POST /internal/v1/federated-learning/training-requests`，payload 只接受 canonical UUIDv4 `requestId` 與
   `modelFamilyId`；GET resource 可讀取 mode、participant source、state、round progress、candidate digest 與
   sanitized failure。
3. 每個 Client private collection API 為
   `POST／GET／DELETE /internal/v1/training-data-collections/{requestId}`；Client 只接受自己的
   `data-owner-N` profile。
4. Collection `DELETE` 的語意是停止／清理 remote collection resources，並把已儲存資料的 descriptor 保留為
   `RETAINED`；它不是刪除 retained training evidence。Descriptor 預設 retention 為 3600 秒。
5. Server training request 不會替四個 Clients 建立 private collection。它只在 descriptor 已存在後，透過 containing
   NWDAF 建立四個標準 ML Model Training resources，執行 preparation、rounds、validation，最後再 DELETE 這四個
   training resources。
6. Static request 的 `required_cutover_scope_keys` 為空，因此成功 publication 後直接進入 `COMPLETE`；本 Phase
   沒有 production Consumer／Model Monitor scopes、PyAnLF reprovision 或 cutover acceptance。
7. Publication 會建立 final bundle、透過 containing NWDAF 將 model 儲存至 ADRF、commit Server durable catalog，
   並記錄 participant sample counts、validation summary、ADRF reference 與 artifact digest。
8. 同一 requestId／model family 的 retry 是 idempotent；同一 requestId 改綁 family 或另有 active top-level request
   時回傳 conflict。Terminal top-level status 是 process-local 且有 TTL；containing NWDAF generation change 或
   PyMTLF restart 會清除 top-level request view。

若 implementation characterization 證明任一 core assumption 為 false，或完成流程必須新增／改變 component API、
persistence、public SBI 或 architecture ownership，Phase 3 必須停止、更新計畫並取得使用者確認；不得在 testbed
repository 內加平行 workaround。

## 3. Phase-specific decisions proposed for approval

本計畫提案以下 operator contract。使用者核准本計畫後，才授權在後續 implementation 建立這些入口：

1. 新增兩組 manifest-driven Make lifecycle targets；每次都使用同一 selected `TESTBED`／`CONFIG_DIR` pair：
   - `make fl-collection-start／status／stop TESTBED=... CONFIG_DIR=... RUN_ID=...`：操作 selected static data
     owners 的 private collection；
   - `make fl-training-start／status TESTBED=... CONFIG_DIR=... RUN_ID=... [MODEL_FAMILY_ID=...]`：操作
     selected static top-level coordinator 的 manual training request。
2. Phase 3 只允許 selected deployment 為 `flat + static`。命令名稱保留 Phase 4 可擴充空間，但本 Phase 不實作
   HFL endpoint selection；對 static HFL 或 production Flat 必須 fail closed。
3. Operator 必須明確提供一個 canonical UUIDv4 `RUN_ID`，並在四個 Client-local collection namespaces 與
   Server-local training namespace 重用同一值，作為整個 run 的 correlation identity。
4. 不新增 host-side request database、ledger 或 hidden current-run pointer。Operator／run record 必須保存
   `RUN_ID`；未提供時命令拒絕，不暗中生成難以重現的 identity。
5. Model family 從 selected coordinator native config 解析。恰有一個 seed family 時可直接使用；若存在零個或
   多個 family，operator 必須明確提供 `MODEL_FAMILY_ID`，且該值必須匹配 selected config。
6. Collection commands 由 manifest exact inventory 找出四個 Clients、published endpoints 與各自 profile；不得
   維護另一份 Client／port／profile 清單。
7. `fl-collection-start` 對 transient ambiguous response 以同一 `RUN_ID` 安全 retry；若只部分建立，必須對本次
   exact clients 執行 bounded cleanup，列出未收斂資源並 fail closed，不可繼續 training。
8. `fl-collection-stop` 必須等四個 requests 全部進入 `RETAINED`，且每個均為
   `descriptorState=RETAINED`、`activePeerResourceCount=0`、`cleanupPending=false`，才宣告 collection ready。
9. `fl-training-start` 在四份 retained collection status、selected／active config identity、coordinator health 與
   seed family 未通過前拒絕 POST。它不內嵌 collection、traffic 或 auto-stop 行為。
10. 不新增 top-level cancel／DELETE API。Training 進入 active 後，正常操作以 status 等待 terminal；必要的
    experiment shutdown 走既有 `experiment-stop`，由 component shutdown／generation fencing 執行 participant
    cleanup，並保留 failure evidence。
11. 擴充現有 static `experiment-status`／`ml-status`，讓它呈現 static Flat 最新 current-run milestones；不再以
    `milestones=not-evaluated` 作為 Phase 3 最終狀態。Exact request state 仍由帶 `RUN_ID` 的
    `fl-collection-status`／`fl-training-status` 擁有。

這些 commands 是既有 lifecycle 的新 entrypoints，屬於 architecture decision；它們只封裝已存在的 private APIs、
manifest identity checks 與 evidence interpretation，不建立新的 component contract 或 config source。

## 4. Authoritative inputs、artifacts 與 owners

| Item | Owner／source | Phase 3 use |
| --- | --- | --- |
| Deployment topology | `testbed.static-flat.yaml` | 唯一 static Flat NF、placement、endpoint、UE ownership 與 Host ML inventory source |
| Experiment input | `experiments/examples/fl-closure-smoke/scenario.yaml` | 第一輪 bounded flow 的 traffic、sampling、2 local epochs、2 fitting rounds 與 300-second closure budget；不新增 Phase 3-only scenario source |
| Generated native config／manifest／Compose | selected `CONFIG_DIR` | processes 實際消費的 config、exact lifecycle inventory、collection profiles、topology 與 config identity |
| Run identity | operator-provided `RUN_ID` | 同一 UUIDv4 關聯四個 Client collection requests 與一個 Server training request；由 operator record 保存 |
| Model family | selected Server native config | 目前預期為 `ue-communication-default`；不在 Makefile 維護第二份 family truth |
| Collection request state | 各 Client PyMTLF private collection ledger | create／collect／cleanup／retained descriptor lifecycle owner |
| Raw and retained data | SMF／UPF、ADRF、Client Mongo collection 與 descriptor | 每個 Client 只擁有自己兩個 SUPI 的 actual collected snapshot |
| Training resource state | Server／Client NWDAF and PyMTLF processes | 四個 standard ML Model Training resources、round fitting、notifications 與 exact DELETE |
| Aggregate artifacts | Server PyMTLF FL workspace | round inputs、四份 local updates、sample-count-weighted FedAvg global artifacts 與 candidate digest |
| Published model | ADRF＋Server durable model catalog | final bundle、ADRF transaction/reference、新 model identity 與 current family revision |
| Run evidence | command output、bounded logs、artifact metadata 與 docs record | revisions、dirty flags、config/image/dataset identity、request states、sample counts、rounds、publication、cleanup 與 limitations |

Ignored `config/local/...`、generated dataset、containers、volumes 與 runtime ledgers 是 run artifacts，不納入 source
commit。Experiment definition／acceptance 說明放在 `testbed-docs/5g-nwdaf-infrastructure/experiments/`；通過 required
evidence 與 user review 的執行結果才放入 `records/`。

## 5. Canonical operator flow

### 5.1 Prepare and start runtime

1. 由 `testbed.static-flat.yaml` 與既有 `fl-closure-smoke` scenario 建立新 `CONFIG_DIR`。
2. 執行 config、dataset、capacity、component lock 與 provider diagnostics；real provider calls 只能在 approved
   sandbox-outside Host context 執行。
3. 記錄 repository／component revisions、dirty flags、selected config hash、dataset manifest/hash、container
   image IDs、Host capacity 與 provider／OS process inventory。
4. 以既有 `vm-up`／`experiment-start` lifecycle 啟動 27 個 Guest units、八個 UE sessions 與五個 PyMTLF
   containers；static deployment 不啟動 production Consumer subscriptions。
5. 確認 selected／active config identity、五個 NWDAF NRF profiles、八個 UE Registration／PDU Sessions 與五個
   PyMTLF readiness 全部一致。

### 5.2 Collect and retain four data partitions

1. Operator 產生並保存一個新的 canonical UUIDv4 `RUN_ID`。
2. `fl-collection-start` 對 manifest 中四個 exact data owners 各送一個 POST；requestId 相同，profile 分別為
   `data-owner-1` 至 `data-owner-4`。
3. 等四個 requests 進入 `COLLECTING`，確認每個 owner 解析自己的 Internal Group、兩個 SUPI 與當前 SMF／UPF
   targets，沒有跨 owner overlap。
4. 讓 current-run deterministic UPF stimulus 經實際 SMF／UPF collection path 產生 callbacks；support callback
   fixture 只能用於 focused tests，不可替代此 evidence。
5. 等每個 owner 的 record／observation counts 達到 scenario minimum，並證明兩個 SUPI 均有本 run data。
6. `fl-collection-stop` 對四個 exact requests 送 DELETE，等待 remote peer cleanup 完成並把四份 descriptors
   retained。這一步只停止 collection resources，不刪除 descriptor 或已儲存 data。
7. Training 必須在 retained descriptor TTL 與 configured preparation window 內開始；過期時必須使用新
   `RUN_ID` 重做 collection，不得把 stale data 冒充 current run。

### 5.3 Run static Flat training

1. `fl-training-start` 讀取 selected Server endpoint／family，確認四份 collection preconditions 後，對
   `/internal/v1/federated-learning/training-requests` POST 同一 `RUN_ID`。
2. Server topology planner 必須 exact 選出 Client 1–4；NRF discovery 回傳的 identity／capability／endpoint 必須
   和 topology assignment 一致。
3. Server 建立四個標準 ML Model Training resources；每個 Client 只解析自己的 private retained descriptor，
   並固定 actual collection time window。
4. 四個 Clients 完成 preparation 後執行兩輪 local fitting。每輪 Server 收齊四份 update，以 artifact metadata
   中的 positive sample count 執行 sample-count-weighted FedAvg；不允許缺 participant 或 silent fallback。
5. Server 將 round 2 global artifact 作為 candidate，要求四個 Clients 完成 final validation，保留 per-client 與
   aggregate validation summary。
6. Performance gate 依 bounded smoke scenario 保持 disabled；仍必須記錄 `gate_would_accept` 與 rejection reasons，
   不能把 disabled gate 說成 model quality 已通過 benchmark。
7. Server 將 validated candidate 儲存至 ADRF、commit durable family catalog，得到新 model ID、artifact digest、
   ADRF instance／transaction／resource identity；因 required scopes 為空，top-level request 應進入 `COMPLETE`。
8. Server DELETE 四個 training resources。Evidence 必須證明四個 create 和四個 terminal DELETE 一一對應，
   且沒有 `cleanup_failure`、stale active resource 或 current-run correlation。

### 5.4 Stop, retain and reset

- Successful flow 不自動停止 processes；operator 先取得完整 evidence，再執行既有 `experiment-stop`。
- Ordinary stop 保留 collection ledger／descriptors、ADRF model、Server catalog、artifacts、containers、volumes、
  datasets 與 VMs。
- `RUN_ID` 不由 repository 隱式保存；run record 必須保存它。新的 collection run 必須使用新的 UUIDv4。
- Guarded reset 仍由 selected manifest exact scope 清除五個 ML volumes、ADRF／catalog／collection state 與
  training artifacts，並在下一次 Server startup 驗證 canonical seed restoration。

## 6. Production baseline stage disposition

Phase 3 的 canonical existing flow 是 production Flat business flow：complete `TESTBED`＋scenario 經同一 renderer／
checker／manifest lifecycle 啟動，經 data stimulus、trigger、participant training、FedAvg、validation、publication、
cleanup 與 status 收尾。Static Flat 不複製這條 pipeline；各 stage disposition 如下：

| Baseline stage | Disposition | Phase 3 contract |
| --- | --- | --- |
| Complete `TESTBED` selection | Reused without semantic change | 使用 Phase 2 `testbed.static-flat.yaml`，不新增 selector／overlay／deployment source |
| Scenario／traffic input | Reused without semantic change | 使用既有 `fl-closure-smoke`；scenario 繼續擁有 timing、traffic、epochs、rounds 與 gate |
| Config render／strict validation／native load | Reused without semantic change | 沿用單一 Phase 2 pipeline；Phase 3 不新增 renderer／checker |
| Generated config／manifest／Compose | Reused without semantic change | 沿用 exact 1 Server／4 Clients／8 UE／5 PyMTLF inventory 與 selected identity |
| VM definition／lifecycle | Reused without semantic change | 沿用三台 VM、provider guard 與 OS process preflight，不新增／重建 VM |
| Guest stage／activate／service lifecycle | Reused without semantic change | 沿用 manifest-driven 27-unit lifecycle、NRF identity 與 wrong-config protection |
| Host ML lifecycle | Reused without semantic change | 沿用 selected five-container Compose artifact、health、logs、stop 與 volume ownership |
| Subscriber／dataset／traffic stimulus | Adapted | 使用八 UE、四個 2-SUPI partitions；不啟動 production Consumer，但保留相同 real SMF／UPF stimulus boundary |
| Collection source | Explicitly replaced | production `consumer_subscription` descriptors 改為四個 operator-controlled private collection requests／retained descriptors |
| Training trigger | Explicitly replaced | degradation trigger 改為 operator private API；新增 manifest-driven `fl-training-*` entrypoint |
| Participant selection | Explicitly replaced | `monitor_scopes` 改為 exact static topology Client 1–4；不由 Consumer demand 決定 |
| Client fitting／Server aggregation | Adapted | 沿用 component-native training transport 與 FedAvg，participant 數由 A／B 兩個改為四個 exact Clients |
| Validation／publication | Adapted | 沿用 final validation、ADRF store 與 catalog commit；static flow required cutover scopes 為空，不執行 reprovision／cutover |
| Status／logs／evidence | Adapted | 擴充 static milestone interpretation，另以 `RUN_ID` 查 collection／training resources 與 artifact metadata |
| Stop／reset／seed recovery | Reused with Phase 3 evidence | 沿用 Phase 2 exact manifest scope；補驗證 collection、publication 與 training state 的 retention／reset |

## 7. Implementation slices

每個 slice 完成 focused verification、mandatory initial review 與 user-review handoff 後才擴張下一個 slice。若使用者
希望依序不中斷完成所有 slices，仍需在 implementation approval 時明確授權；本 planning approval 本身不授權實作。

### Slice 1. Current-flow characterization and experiment contract

- 以 pinned source／tests 固定 private collection、manual training、static selection、FedAvg、publication、cleanup、
  terminal TTL 與 restart semantics；
- 建立 Phase 3 experiment definition，明列 `RUN_ID`、collection→retention→training 時序、closure budget、direct
  evidence 與可宣稱限制；
- 以 component-native focused tests 重驗 source assumptions，不改 component behavior；
- 建立 normative conformance checklist 與 real-run evidence template。

### Slice 2. Manifest-driven private collection lifecycle

- 新增 `fl-collection-start／status／stop`，只由 selected manifest/native config 解析四個 endpoints／profiles；
- 驗證 selected topology、active config labels、Client health、canonical UUIDv4 與 exact 4×2 ownership；
- start retry idempotent，partial success 執行 exact bounded rollback；status 顯示每個 owner state、UE/peer counts、
  records、observations、descriptor 與 cleanup；
- stop 只有在四份 descriptors 均 retained、remote resources 清零且無 cleanup pending 時成功；
- 加入 invalid／empty inventory、wrong config、unknown profile、409、503、timeout、partial create／cleanup 與 stale
  descriptor tests。

### Slice 3. Manual training trigger and static observability

- 新增 `fl-training-start／status`，從 selected coordinator native config 解析 endpoint／model family；
- start preflight 拒絕 collection 尚未 retained、descriptor expired、coordinator unhealthy、wrong config、family mismatch、
  non-Flat／non-static topology；
- ambiguous POST 以同一 request identity 查詢／retry，正確呈現 202、404、409、422、503 與 timeout；
- 擴充 `ml-status`／`experiment-status` 的 static Flat milestone parser，呈現 preparation、rounds、validation、
  publication、complete／failed；
- status/evidence 解析 candidate artifact metadata、四個 participant sample counts、FedAvg aggregate identity、ADRF
  publication log 與 exact training-resource cleanup evidence。

### Slice 4. Four-client successful-flow integration

- 以 controlled fixtures／disposable runtime 驗證四個 collection profiles、四份 retained descriptors、兩輪四-client
  fitting、sample-count-weighted FedAvg、final validation、ADRF publication 與 cleanup；
- 驗證 candidate／published artifact、model family generation、ADRF record 與 current catalog identity 一致；
- 驗證 static completion 沒有 required cutover scopes、PyAnLF reprovision 或 Consumer chain；
- 不以 mock ADRF／containing NWDAF 取代下一個 slice required real boundary。

### Slice 5. Failure, restart and cleanup closure

- Collection：partial provision、callback absence、insufficient data、DELETE retry、cleanup pending、Client restart 與
  retained descriptor recovery；
- Training：duplicate／conflicting request、missing family、participant discovery mismatch、preparation timeout、single
  Client failure、round timeout、validation failure、ADRF publication retry／failure 與 training-resource cleanup failure；
- Restart：Server／Client containing NWDAF generation change 與 PyMTLF stop/restart，不得讓 stale terminal view、
  correlations、training resources 或 collection resources 被當成新 run；
- `experiment-stop` during active training 走既有 shutdown fencing；若 exact cleanup 無法證明，狀態保持 failure／
  verification incomplete，不用 process stopped 取代 cleanup evidence；
- guarded reset 後 collection／training／publication state 清空，canonical seed identity 恢復。

### Slice 6. Real three-VM acceptance, regression and handoff

- 在 approved sandbox-outside Host context 以新 `RUN_ID` 執行完整 §5 flow；
- 取得 real SMF／UPF collection、四份 private retained descriptors、四-client fitting、兩輪 FedAvg、validation、
  ADRF publication 與四個 training-resource cleanup evidence；
- 執行 production Flat `experiment-start/status/stop` regression，確認既有 Consumer／degradation／monitor-scopes／
  cutover flow 沒有被 static entrypoints 改寫；
- 執行 repository full checks、diff review、plan conformance 與 documentation language pass；
- 保持 changes unstaged／uncommitted，停在 `Ready for User Review`，之後另提 commit proposal。

## 8. Failure and recovery contract

| Failure | Required behavior |
| --- | --- |
| Missing／invalid `RUN_ID` | 在任何 API call 前拒絕；不自動產生或猜測 identity |
| Wrong selected config／active labels | collection／training mutating commands 與 status fail closed，列出 selected／actual identity |
| Partial collection POST | 只清理由本次 exact four-client request identity 建立的 resources；列出未收斂 clients，禁止 training |
| Collection callback／records 不足 | 保持 current state 與 evidence，timeout 後明確失敗；不得用 generated dataset 存在取代 actual callback evidence |
| Collection DELETE cleanup pending | 重試 bounded cleanup；未收斂時不得宣稱 descriptor ready 或開始 training |
| Retained descriptor expired | 使用新 `RUN_ID` 重做 collection；不得延長舊 evidence 或 fallback 到別的 source |
| Ambiguous training POST timeout | 以同一 request GET／idempotent retry；不得換 request ID 造成兩個 active requests |
| Active request conflict | 顯示 existing request／family conflict；不停止或覆寫 active run |
| Participant discovery／identity mismatch | Server request terminal failure，cleanup 已建立 resources；不縮小成部分 participants |
| Client fitting／round timeout | terminal failure；保留 request/log/artifact evidence 並 exact DELETE 已建立 training resources |
| Publication transient failure | 沿用 component durable retry；status 保持 in-progress，不把 candidate-ready 當 published |
| Publication terminal failure | 不更新 current catalog、不宣稱 complete；保留 ADRF／journal evidence 供診斷 |
| Server／Client generation change | fence 舊 generation、清理 owned resources 並清除 volatile top-level view；restart 後只能以新 run 或明確 recovery evidence 繼續 |
| Cleanup evidence 不足 | 即使 training state 為 `COMPLETE`，Phase 3 仍保持 verification incomplete |

## 9. Verification matrix

| Layer | Verification | Required gate |
| --- | --- | --- |
| Static source | selected testbed／scenario／native config／manifest exact comparison | 1 Server、4 Clients、8 UEs、4 disjoint owners、5 PyMTLF、FedAvg、2 rounds 一致 |
| Operator lifecycle | synthetic HTTP／manifest fixtures for `fl-collection-*` and `fl-training-*` | idempotency、partial rollback、wrong config、invalid inventory、HTTP failures 與 timeouts fail closed |
| Component contract | pinned PyMTLF focused native tests | private collection retention、manual static request、4-participant-capable FedAvg、publication 與 cleanup semantics 保持 |
| Status／evidence | deterministic log/API/artifact fixtures | static milestones、candidate digest、four sample counts、publication、failure 與 cleanup 不誤判 |
| Repository | focused tests、`make test`、applicable disposable container checks、`git diff --check` | 全部通過；container-only evidence 不宣稱 real 5GC／ADRF acceptance |
| Real collection | approved Host context, real 3-VM SMF／UPF path | 四個 owners 各兩個 SUPI 有 current-run observations，四份 descriptors retained 且 peer cleanup 完成 |
| Real training | approved Host context, five PyMTLF plus five NWDAFs | exact 四 participants、兩輪 local fitting、positive sample counts 與 sample-weighted FedAvg 完成 |
| Real publication | ADRF record＋Server catalog＋artifact metadata | 新 model identity、digest、ADRF reference、validation summary 一致，required scopes 為 0 |
| Real cleanup | collection status、training create/delete evidence、unexpected inventory scan | collection peers 與四個 training resources 無 stale active state 或 cleanup failure |
| Regression | production Flat full lifecycle | Consumer、degradation trigger、monitor scopes、two clients、publication/cutover operator interpretation 不變 |
| Reset／seed | selected guarded reset＋restart | Phase 3 retained state 清空、其他 topology 不被清除、canonical seed digest 恢復 |

所有 real provider commands，包括 status／validate 類 query，都必須使用 approved sandbox-outside execution flow；
sandbox／CI 只可執行 synthetic provider fixtures，不得以真實 Vagrant／VirtualBox invocation 測試 guard。

## 10. Phase completion criteria

只有同時滿足下列條件才完成 Phase 3：

1. 沿用 Phase 2 authoritative `TESTBED`／scenario／renderer／checker／manifest pipeline，沒有新增 config source、
   selector、平行 renderer／checker、VM、service 或 component persistence。
2. Operator 可用一個明確 `RUN_ID` 操作四個 exact private collections 與一個 Server training request，且 retry、
   wrong config、partial failure 與 timeout semantics 明確、fail closed。
3. 四個 Clients 各只解析自己的 2-SUPI Internal Group；八個 SUPI 互斥且 current-run real collection evidence 完整。
4. 四個 collection requests 完成 remote cleanup 並各保留一份未過期 descriptor；training 使用這四份 actual snapshots，
   不 fallback 到 consumer descriptors 或未證明的 Mongo data。
5. Server exact 選出四個 Clients，兩輪均收到四份 positive-sample local updates 並執行 sample-count-weighted FedAvg。
6. 四個 Clients 完成 final validation；candidate artifact、participant/sample metadata 與 aggregate validation evidence
   可直接檢查。
7. Validated candidate 成功儲存至 ADRF 並 commit Server durable catalog；新 model ID、artifact digest、ADRF reference
   一致，top-level request 為 `COMPLETE` 且 required cutover scopes 為 0。
8. 四個 standard ML Model Training resources 全部 DELETE；沒有 stale current-run correlation、cleanup failure 或
   unexpected active resource。
9. Failure、timeout、restart、ordinary stop、retained state 與 guarded reset paths 依 §8 通過；cleanup evidence 不足
   時不得只依 `COMPLETE` 宣告完成。
10. Static status 不再顯示 `milestones=not-evaluated`，exact resource status 與 aggregate milestone summary 不互相矛盾。
11. Production Flat regression、repository full checks 與 required real three-VM evidence 通過。
12. Mandatory initial review、plan conformance、user review 與獨立 commit gates 全部完成。

## 11. Final normative conformance map

### 11.1 Verification identity 與直接證據

本節保存 Phase 3 final evidence。User review 已於 2026-08-28 確認，verified run record 已整理至
[Static Flat controlled-flow validation record](../../records/hierarchical-federated-learning/static-flat-controlled-flow-validation-2026-08-27.md)。

- Real run 的 Infrastructure source baseline 為 `fc360fc4250`，PyMTLF source baseline 為 `36166f04320a`，並以
  reviewed working-tree diff 建置與驗證；該 diff 現已分別由 Infrastructure `e5b1d44` 與 PyMTLF `7479629`
  保存。Static Flat generated config 為
  `phase3-static-flat-v1:4fdd0da11fe41de263ec349a1a28a688031d735672f0f4f605d29e78f34396bf`，dataset
  identity 為
  `68b65b9bacc58ba30d53bc8ece46e0374185d8301f993839b4645fde90830da9`。Static PyMTLF image 為
  `sha256:4680b398c6bfeda4b38127fbf2e1c1d6c82990b33f0a0c9fc42b63cc26c1c9b9`，OCI revision label 為
  `36166f04320ae70674604659786ba73935371426`。
- Real static run 使用 `RUN_ID=61ae473f-85a7-4a67-a0d8-08ddd1c02600`。四個 owners 各解析兩個互斥 SUPI、
  保存 4 records／202 observations，停止後均為 `RETAINED`、peer resource 為 0、descriptor retained 且
  `cleanupPending=false`。
- Training process 為 `fcd7c795-7e22-45fc-a9c8-1a6466092dd8`，exact participants 為 Client 1–4；四個
  Clients 在 rounds 0、1 均各使用 37 samples，並全部完成 final validation。Candidate digest 為
  `3cf53e9c731d8a8adacc6c8f77bbd4a63f489cdd3871484fec895a0137ecf513`；aggregate validation 記錄
  base WAPE `2.3936889687029064`、candidate WAPE `0.19506470978235016` 與
  `gate_would_accept=true`，但 performance gate 仍依 bounded-smoke contract disabled。
- Published model ID 為 `1787842701831`，top-level state 為 `COMPLETE`、`required_scopes=0`；四個 training
  resources exact `created=4 deleted=4 active=0 unknown_deletes=0`，沒有 failure 或 cleanup failure。
- Fresh-state verification 先以 selected guarded reset 清除 static Flat 的五個 ML volumes 與 exact
  ADRF／NRF／model scope，確認其他 topology assets 保留；Server 重新啟動後記錄 canonical seed key
  `a2c796a001e2da2461418f80b01d7d1e33f0e3349c2817d92286f09e67aa6bef`。實際 run 也確認若同一
  Client 同時有多個 matching retained groups，DatasetCoordinator 會依 component contract fail closed；保存
  evidence 後以 selected guarded reset 回到 fresh state，而未新增 dataset selector 或放寬 ambiguity guard。
- Production Flat regression 使用
  `default:4c42dae057fd400f22ac004bd518b6184c133856579454dc07381dd9cfbe5412`、dataset
  `23697bf00ae0560c9f07f8ae451ebb91797943092317aea8cafdb37435c2fd59` 與既有 5-container／
  23-Guest-unit runtime；PyMTLF image 為 `d1140540fde3`，PyAnLF image 為 `9b75d16725a5`。Consumer 兩個
  TAI subscriptions 各收到 47 callbacks；degradation 觸發 process
  `dff9cf15-4c5a-4fb7-866c-49d045936b5f`，兩個 Clients 完成兩輪、publication／adoption／cutover，model
  `1787844693934` 的 post-cutover accuracy 為 `evaluated=true triggered=false`。Cleanup 為
  `created=2 deleted=2 active=0`，最終 `FL RESULT outcome=complete`；ordinary stop 刪除兩個 subscriptions、
  停止五個 ML containers 與全部 23 個 Guest units，VM 與 retained assets 保留。
- Controlled local integration 另直接覆蓋 successful flow、collection restart cleanup、preparation failure
  recovery、round timeout 與 containing-NWDAF generation restart；operator fixtures 覆蓋 invalid inventory、
  wrong config、HTTP errors／timeouts、partial create 與 bounded rollback。Repository full tests、PyMTLF full tests、
  Ruff、disposable five-container CPU lifecycle 與 diff checks 均列入 final review evidence。

| Normative item | Current evidence | Status／open work |
| --- | --- | --- |
| Phase 2 static Flat topology／identity／lifecycle baseline | Current-run config、dataset、Guest／container identity 與 real three-VM inventory | Satisfied；沿用 single pipeline，未新增 selector、VM、service 或 persistence |
| Private collection API 與 retained descriptor semantics | `fl-control.py`／tests，加上 real 4-owner 2-SUPI collection、retention 與 peer cleanup | Satisfied |
| Static manual trigger 與 exact participant selection | Manifest-derived Server／Client mapping、preflight tests 與 real Client 1–4 participant evidence | Satisfied |
| Four-client two-round FedAvg | 四 Clients 每輪各 37 samples、兩輪 aggregate 與 candidate digest | Satisfied |
| ADRF publication／catalog commit／no cutover scopes | Real model `1787842701831`、artifact digest、`COMPLETE` 與 `required_scopes=0` | Satisfied |
| Collection 與 training cleanup | 四份 collection peer cleanup，加上 training `created=4 deleted=4 active=0` | Satisfied |
| Static status／failure interpretation | `ml-status` deterministic parser tests 與 real `outcome=complete ... evidence=static-publication` | Satisfied；static HFL 繼續明確標示未評估，留給 Phase 4 |
| Failure／restart／reset closure | Operator failure fixtures、controlled local restart／timeout flows、real ambiguity fail-closed 與 guarded reset／seed evidence | Satisfied |
| Production Flat regression | Real Consumer→degradation→two-client training→publication／cutover→post-cutover accuracy→stop | Satisfied；既有 production operator interpretation 未改變 |
| Mandatory initial review 與 user review | Initial／targeted review、final conformance 與 2026-08-28 user review 已完成 | Satisfied；PyMTLF `7479629` 與 Infrastructure `e5b1d44` 已提交，本文件 commit 保存最終 record |

## 12. 明確不包含

- Static Hierarchical Root／Branch／Leaf execution；由 Phase 4 管理。
- Flat FedProx、HFL FedAvg、algorithm transport migration 或 controlled performance comparison。
- Production Consumer／Model Provision／Model Monitor participant-selection chain、PyAnLF reprovision 與 generation
  cutover；只做 regression，不加入 static completion path。
- 自動建立 collection、自動觸發 training、自動停止 experiment 或自動 reset 的 single-button workflow。
- Top-level training cancel／DELETE API、durable host-side run ledger、current-run pointer 或 request database。
- 新增 VM、改變 NF placement、合併 NWDAF／PyMTLF identity、共享 writable state 或降低 GPU／memory acceptance。
- 新 component API、public SBI、service、external dependency 或 persistence；若實際需要，先走 decision gate。
- Production benchmark、real application throughput、model-quality superiority 或 Flat-vs-HFL 公平比較宣稱。
