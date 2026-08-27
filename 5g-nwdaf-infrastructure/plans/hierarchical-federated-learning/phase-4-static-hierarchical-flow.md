# Phase 4 Static Hierarchical Flow Detailed Plan

日期：2026-08-28

狀態：Planning / Ready for User Review；尚未授權 implementation

最近更新：2026-08-28

## 1. 目的

在 Phase 2 已完成的一 Root／兩 Branches／四 Leaves、八 UE、七個獨立 NWDAF／PyMTLF runtime foundation，
以及 Phase 3 已驗證的 private collection、manual trigger 與 operator lifecycle 上，跑通 static Hierarchical 的
完整 controlled flow：

```text
operator 啟動四份 Leaf private collection
→ 四個 Leaves 各自蒐集兩個 SUPI 的 observations
→ operator 停止 collection 並保留四份 descriptor
→ operator 對 Root 建立 manual training request
→ Root exact 指派兩個 Branches，每個 Branch exact 指派兩個 Leaves
→ 四個 Leaves 使用各自 retained snapshot 執行兩輪 FedProx fitting
→ 兩個 Branches 各自完成 lower-tier sample-weighted aggregation
→ Root 完成 upper-tier sample-weighted aggregation
→ 四個 Leaves 完成 final validation
→ ADRF storage 與 Root model catalog publication
→ exact upper／lower training-resource cleanup
```

本 Phase 只證明 static Hierarchical 的 collection、two-tier training、aggregation、publication 與 cleanup contract
能在實際三台 VM testbed 上閉合。它不作 performance benchmark、communication measurement 或 Flat／HFL 公平
比較，也不把 FedAvg／FedProx 或 topology 差異解讀為單一因果效果。

本計畫受 [testbed development policy](../../development_policy.md) 約束。Component behavior 以目前 pinned Go
NWDAF、PyMTLF、SMF、UPF 與 ADRF source／tests 為準；本 Phase 不修改 component architecture、API 或 transport
implementation。

## 2. 已完成基線與 core assumptions

### 2.1 Phase 2 static Hierarchical baseline

- Infrastructure common-foundation baseline 為 `fc360fc`，provider safety baseline 為 `531e335`。
- `testbed.static-hierarchical.yaml` 已透過單一 renderer／checker／manifest pipeline 建立一 Root、兩 Branches、
  四 Leaves、八 UE、七個 PyMTLF containers 與 29 個 Guest units。
- Root、Branches 與 Leaves 均為獨立 Go NWDAF NF identity、process、native config、NRF registration、endpoint、
  log、PyMTLF process 與 writable volume；Branch dual role 不代表共用 Root identity 或 state。
- Static Hierarchical topology 已在既有 `core`、`path-a`、`path-b` 三台 VM 驗證 render、validate、activate、
  selected-config identity、NRF registration、status／logs、stop、exact reset 與 deterministic seed restoration。
- Phase 4 不新增或重建 VM。開始 real run 前仍須重新記錄 actual revisions、dirty flags、selected config hash、
  image identity、Host capacity、OS provider process inventory 與 current runtime inventory。

### 2.2 Phase 3 operator-lifecycle baseline

- Infrastructure baseline `e5b1d44` 已建立 manifest-driven `fl-collection-*`／`fl-training-*`、static milestone
  interpretation、bounded rollback、same-ID retry、wrong-config protection 與 exact evidence output。
- Phase 3 已在 real three-VM static Flat run 證明四個 2-SUPI private collections、retained descriptors、manual
  training、two-round aggregation、ADRF publication、training-resource cleanup、ordinary stop、guarded reset 與
  production Flat regression。
- Phase 4 的 canonical existing flow 是 Phase 3 static Flat controlled flow；本 Phase 擴充同一個
  `fl-control.py`、同一組 Make targets、同一個 `ml-status` 與同一 selected manifest pipeline，不建立 HFL-only
  operator script、target namespace、config source、selector、renderer 或 checker。
- Phase 3 verified record 保存既有 source 與 real-run evidence；Phase 4 必須使用新的 `RUN_ID` 與 current-run
  identity，不能把 Phase 3 runtime evidence當作 HFL evidence。

### 2.3 Source-confirmed component contract

目前 pinned source 已具備下列 Phase 4 building blocks：

1. Root 使用 `orchestration.mode: hierarchical`、`participant_source: static`、static topology與
   `complete_required` admission；strategy 為 FedProx、`proximal_mu: 0.01`、`sample_weighted` aggregation。
2. Root manual private trigger沿用
   `POST /internal/v1/federated-learning/training-requests`；GET resource可讀取 `mode=hierarchical`、
   `participantSource=static`、state、round progress、candidate digest與sanitized failure，不使用另一個HFL API。
3. Root是唯一top-level coordinator。它從generated topology exact指派兩個Branches；每個Branch再透過NRF解析並
   指派自己兩個Leaves。`complete_required`不允許缺Branch、缺Leaf或縮小participant set後繼續。
4. Branch同時是upper-tier Client與lower-tier Server，但不擁有training data、不建立private collection，也不
   autonomous啟動另一個Root request。四個Leaves才是data owners，沿用
   `POST／GET／DELETE /internal/v1/training-data-collections/{requestId}`。
5. Collection `DELETE`保留各Leaf的`RETAINED` descriptor並完成remote peer cleanup；Root manual request不替
   Leaves建立collection，也不fallback到Consumer descriptor或未證明的Mongo data。
6. 每個round由四個Leaves執行FedProx local fitting；Branch 1聚合Leaves 1–2，Branch 2聚合Leaves 3–4，Root再依
   effective positive sample count聚合兩份Branch results。第一輪scenario維持兩個fitting rounds。
7. 成功流程建立兩個Root→Branch upper resources與四個Branch→Leaf lower resources；publication前完成四個Leaf
   final validation，之後由Root把validated candidate儲存至ADRF並commit durable family catalog。
8. Static manual request沒有production Model Monitor scopes，因此required cutover scopes為空；publication成功後
   top-level request應進入`COMPLETE`，不執行PyAnLF reprovision或generation cutover。
9. Top-level terminal view是process-local且有TTL；containing NWDAF generation change、Root／Branch／Leaf
   PyMTLF restart、ordinary stop與reset仍須依現有component fencing及cleanup contract判讀。

若implementation characterization證明上述任何core assumption為false，或完成流程需要修改component API、
persistence、public SBI、transport、algorithm或architecture ownership，必須停止、更新計畫並取得使用者決策；
不得在Infrastructure repository增加平行workaround。

## 3. Phase-specific decisions proposed for approval

本計畫提案下列operator contract；使用者核准後才授權implementation：

1. 沿用Phase 3既有命令，不新增HFL-only target：
   - `make fl-collection-start／status／stop TESTBED=... CONFIG_DIR=... RUN_ID=...`；
   - `make fl-training-start／status TESTBED=... CONFIG_DIR=... RUN_ID=... [MODEL_FAMILY_ID=...]`。
2. `fl-control.py`依manifest `deploymentKind`接受`static-flat`與`static-hierarchical`兩種已確認static topology，
   仍拒絕production Flat與任何未知topology。
3. Static Hierarchical collection commands只解析role=`leaf`且具有data-owner mapping的四個services；Root與Branches
   不會收到collection request。四個owner positions、profiles、Internal Groups與八個SUPI仍由manifest／selected
   native config exact compare，不新增清單。
4. Training commands從manifest `coordinatorContainer`解析Root endpoint；preflight要求Root為role=`root`、
   `hierarchical + static`、private trigger enabled、topology exact為2 Branches／4 Leaves，且四份Leaf descriptors
   retained、未過期、cleanup-complete。
5. Operator提供同一個canonical lowercase UUIDv4 `RUN_ID`，在四個Leaf collection namespaces與Root top-level
   request重用；不新增host ledger、current-run pointer或自動產生identity。
6. `fl-training-status`接受並驗證`mode=hierarchical`與`participantSource=static`，但仍以exact selected
   coordinator、family與request identity fail closed；Flat status semantics保持不變。
7. `ml-status`／`experiment-status`擴充HFL milestone interpretation，至少呈現Root request／preparation、四個Leaf
   local results、final validation、publication、2 upper＋4 lower resource lifecycle與terminal result；
   `fl-training-status`另呈現top-level round progress。不再以`milestones=not-evaluated`作為Phase 4最終狀態。
8. HFL evidence必須區分top-level API resource、runtime logs與獨立artifact metadata。Branch／Root aggregate
   identity與sample weighting由run artifact evidence擁有，不要求`ml-status`從沒有對應event的log推測；缺少直接
   cleanup或participant evidence時保持`verification-incomplete`，不只依`COMPLETE`宣稱成功。
9. Hierarchy assignment duplicate GET、transport optimization、component remediation與communication instrumentation由其他
   workstream擁有；本Phase不修改、不驗收，也不把它當作functional-flow blocker。Phase 4只禁止由run結果宣稱
   communication cost或performance結論。

## 4. Authoritative inputs、artifacts與owners

| Item | Owner／source | Phase 4 use |
| --- | --- | --- |
| Deployment topology | `testbed.static-hierarchical.yaml` | 唯一Root／Branch／Leaf placement、endpoint、UE ownership與Host ML inventory source |
| Experiment input | `experiments/examples/fl-closure-smoke/scenario.yaml` | traffic、sampling、2 local epochs、2 rounds與bounded closure budget；不新增Phase 4-only scenario source |
| Generated native config／manifest／Compose／topology | selected `CONFIG_DIR` | processes實際消費的native artifacts與exact lifecycle／hierarchy inventory |
| Run identity | operator-provided `RUN_ID` | 關聯四個Leaf collections與一個Root training request；由operator record保存 |
| Model family | selected Root native config | 預期唯一seed family為`ue-communication-default`；不在Makefile維護第二份truth |
| Collection state | 四個Leaf PyMTLF private ledgers | 每個Leaf自己的create／collect／cleanup／retained descriptor lifecycle |
| Hierarchy assignment | Root generated topology＋NRF current discovery | Root→Branches與Branches→Leaves exact identity、capability與endpoint resolution |
| Upper/lower training resources | Root／Branch／Leaf NWDAF與PyMTLF processes | 2 upper＋4 lower standard ML Model Training resources、round callbacks與DELETE |
| Aggregate artifacts | Branch plan workspaces＋Root FL workspace | four local results、two Branch aggregates、Root aggregate、candidate與validation evidence |
| Published model | ADRF＋Root durable model catalog | final bundle、ADRF reference、model identity、artifact digest與family revision |
| Run evidence | command output、bounded logs、API status、artifact metadata與docs record | identity、collections、assignments、rounds、sample counts、publication、cleanup與limitations |

Generated config、dataset、containers、volumes、runtime ledgers與run logs是artifacts，不納入source commit。Experiment
definition／acceptance說明放在`experiments/`；只有通過required evidence與user review的結果才放入`records/`。

## 5. Canonical operator flow

### 5.1 Prepare and start runtime

1. 由`testbed.static-hierarchical.yaml`與既有`fl-closure-smoke` scenario建立新的`CONFIG_DIR`。
2. 執行config、dataset、capacity、component lock與provider diagnostics；real provider calls只能在approved
   sandbox-outside Host context執行，第一個runtime observation必須是Host OS process inventory。
3. 記錄parent／component revisions、dirty flags、selected config hash、dataset identity、image IDs／revision labels、
   Host capacity、provider process inventory、VM identity與Guest active config。
4. 若前一scenario仍active或retained state會造成ambiguous collection，先保存evidence，再使用既有explicit
   stop與selected guarded reset；不自動切換、覆寫或擴大cleanup scope。
5. 以既有`vm-up`／`experiment-start` lifecycle啟動29個Guest units、八個UE sessions與七個PyMTLF containers；
   static deployment不啟動production Consumer subscriptions。
6. 確認selected／active config identity、七個unique NWDAF NRF profiles、八個UE Registration／PDU Sessions與
   七個PyMTLF readiness一致。

### 5.2 Collect and retain four Leaf partitions

1. Operator產生並保存新的canonical UUIDv4 `RUN_ID`。
2. `fl-collection-start`對manifest中四個exact Leaves各送一個POST；profile依序為`data-owner-1`至
   `data-owner-4`，Root與Branches沒有collection call。
3. 確認每個Leaf只解析自己的Internal Group、兩個SUPI與current SMF／UPF targets，owner之間沒有overlap。
4. 以current-run deterministic stimulus經real SMF／UPF path產生callbacks；support fixture只作focused tests。
5. 等每個owner達到scenario minimum，並證明兩個SUPI均有本run observations。
6. `fl-collection-stop`等待四份resources均為`RETAINED`、descriptor retained、active／pending peers為0且
   `cleanupPending=false`。
7. Training在descriptor TTL與preparation window內開始；過期或ambiguous retained groups使用新`RUN_ID`並依
   fresh-run程序處理，不新增dataset selector或放寬component guard。

### 5.3 Run static Hierarchical training

1. `fl-training-start`讀取selected Root endpoint／family，通過四份Leaf collection、runtime identity與Root health
   preflight後，對既有private endpoint POST同一`RUN_ID`。
2. Root exact載入2-Branch／4-Leaf topology，經NRF確認Branches的dual capability；每個Branch再確認自己兩個
   Leaves的identity、capability與endpoint。任何mismatch使complete-required admission失敗。
3. Root建立兩個upper resources；Branches各建立兩個lower resources。四個Leaves只解析自己的retained
   descriptor，並固定actual collection time window。
4. 完成四個Leaf preparation與兩個Branch preparation-result後，執行兩輪training。每輪四個Leaves以
   FedProx fitting；Branches各依兩個positive sample counts產生lower aggregate；Root依兩個Branch effective
   sample counts產生upper aggregate。
5. Root以round 2 global artifact作candidate，要求四個Leaves完成final validation，保存per-Leaf、per-Branch與
   aggregate validation evidence。Performance gate維持bounded-smoke contract disabled，但記錄
   `gate_would_accept`與rejection reasons。
6. Root把validated candidate儲存至ADRF並commit durable family catalog；新model ID、artifact digest與ADRF
   reference一致，required scopes為0，top-level request進入`COMPLETE`。
7. Exact cleanup證明兩個upper與四個lower resources均terminal DELETE，且沒有cleanup failure、stale current-run
   correlation、plan workspace或unexpected active resource。

### 5.4 Stop, retain and reset

- Successful flow不自動停止processes；operator先取得完整evidence，再執行既有`experiment-stop`。
- Ordinary stop保留collection descriptors、ADRF model、Root catalog、artifacts、containers、volumes、datasets與VM。
- `RUN_ID`不由repository隱式保存；新的collection run使用新的UUIDv4。
- Guarded reset由selected manifest exact清除七個ML volumes、ADRF／catalog／collection／training／hierarchy-plan
  state，並在下一次Root startup驗證canonical seed restoration；其他topology assets不得被清除。

## 6. Phase 3 baseline stage disposition

| Baseline stage | Disposition | Phase 4 contract |
| --- | --- | --- |
| Complete `TESTBED` selection | Adapted | 使用既有`testbed.static-hierarchical.yaml`；不新增selector／overlay／source |
| Scenario／traffic input | Reused without semantic change | 沿用`fl-closure-smoke` timing、traffic、epochs、rounds與gate |
| Config render／validation／native load | Reused without semantic change | 沿用Phase 2單一pipeline；不新增renderer／checker |
| Generated config／manifest／Compose | Adapted | 從5-node Flat inventory切換為既有7-node HFL inventory與generated topology |
| VM definition／provider lifecycle | Reused without semantic change | 沿用三台VM、provider guard與OS process preflight；不新增／重建VM |
| Guest stage／activate／service lifecycle | Reused without semantic change | 沿用manifest-driven lifecycle、NRF identity與wrong-config protection |
| Host ML lifecycle | Adapted | selected Compose從5 containers切換為既有7 containers；health／stop／volume semantics不變 |
| Subscriber／dataset／traffic stimulus | Reused without semantic change | 同樣八UE、四個2-SUPI partitions、real SMF／UPF path且沒有Consumer |
| Collection lifecycle | Adapted | 相同4-owner API與retention contract，owner role從Clients改為Leaves；Root／Branches排除 |
| Manual training trigger | Adapted | 相同private endpoint與`RUN_ID`，coordinator role從Server改為Root，mode驗證改為hierarchical |
| Participant selection | Adapted | Flat exact4 Clients改為Root exact2 Branches及每Branch exact2 Leaves，complete-required |
| Local fitting／aggregation | Explicitly replaced | one-tier FedAvg改為Leaf FedProx＋Branch／Root two-tier sample-weighted aggregation |
| Validation／publication | Adapted | 四Leaf hierarchical validation；沿用ADRF／catalog與zero-cutover completion |
| Status／logs／evidence | Adapted | 擴充two-tier milestones、6-resource lifecycle與hierarchy artifacts；Flat interpretation不變 |
| Stop／reset／seed recovery | Reused with Phase 4 evidence | 沿用exact manifest scope，補驗hierarchy-plan與7-volume cleanup |

## 7. Implementation slices

使用者若在implementation approval時明確授權依序不中斷完成所有slices，則只在遇到decision gate、core assumption
為false或必須改變計畫時暫停。本planning approval本身不授權implementation。

### Slice 1. Current-flow characterization and experiment contract

- 以pinned source／tests固定Root private trigger、static topology、Branch dual role、Leaf collection、FedProx、
  two-tier weighting、validation、publication、cleanup、TTL與restart semantics；
- 建立Phase 4 experiment definition與real-run evidence template，明列functional claim與排除的performance／
  communication claim；
- 建立normative conformance checklist，不修改component behavior。

### Slice 2. Manifest-driven Leaf collection and Root trigger

- 將既有`fl-control.py` contract generalize為兩種confirmed static topologies；Flat path保持characterized behavior；
- HFL只解析四個Leaf owners、Root coordinator、兩個Branches與exact topology；
- 沿用`fl-collection-*`與`fl-training-*` Make targets、same-ID retry、partial rollback、retention、family與
  selected／active identity guards；
- 加入HFL valid contract、wrong role、missing／duplicate owner、Branch-as-owner、bad topology、empty inventory、
  nonhierarchical response及Flat regression tests。

### Slice 3. Hierarchical observability and evidence

- 擴充`ml-status`／`experiment-status`解析Root、Branch、Leaf及two-tier resource lifecycle；
- 呈現Root request／preparation、每輪4 local results、4 validation、publication、`required_scopes=0`與2+4
  cleanup；round progress由`fl-training-status`擁有，Branch／Root aggregate identity由獨立artifact evidence擁有；
- status遇到missing participant、count contradiction、cleanup failure、wrong config或terminal API／log矛盾時
  fail closed為failed或verification-incomplete；
- 保留production Flat與static Flat deterministic parser regressions。

### Slice 4. Controlled successful-flow integration

- 以controlled fixtures／disposable seven-container runtime驗證4 Leaf collections、2-Branch／4-Leaf admission、
  兩輪FedProx、lower／upper sample weighting、final validation、ADRF publication與cleanup；
- exact核對candidate／published artifact、family generation、ADRF record、Root catalog及6-resource lifecycle；
- 確認Branch不蒐集data、沒有autonomous Root、static completion沒有Consumer／PyAnLF／cutover chain；
- container或local evidence不取代Slice 6 real three-VM boundary。

### Slice 5. Failure, restart and cleanup closure

- Collection沿用partial create、callback absence、insufficient data、cleanup pending、Leaf restart與retained recovery；
- Hierarchy涵蓋Branch／Leaf discovery mismatch、incomplete admission、preparation timeout、單一Leaf failure、
  lower／upper round timeout、missing Branch aggregate、validation failure、publication與6-resource cleanup failure；
- Restart涵蓋Root／Branch／Leaf containing-NWDAF generation change及PyMTLF stop／restart，舊plan、correlation、
  resources與terminal view不得被當成新run；
- active-training `experiment-stop`沿用component fencing；cleanup證據不足保持verification-incomplete；
- guarded reset後collection、training、hierarchy-plan、publication state清空且canonical seed恢復。

### Slice 6. Real three-VM acceptance, regression and handoff

- 在approved sandbox-outside Host context使用新`RUN_ID`執行完整§5 flow；
- 取得real SMF／UPF collection、四份Leaf descriptors、2-Branch／4-Leaf admission、兩輪FedProx、two-tier
  aggregation、validation、ADRF publication與6-resource cleanup evidence；
- 執行Phase 3 static Flat operator／status regression與repository full checks；若implementation實際修改shared
  render、start、stop、reset或production lifecycle，另補相應real regression，不以conditional omission掩蓋affected path；
- 完成mandatory initial review、targeted remediation、fresh-read conformance與documentation language pass；
- 保持changes unstaged／uncommitted，停在`Ready for User Review`，之後另提repository-separated commit proposal。

## 8. Failure and recovery contract

| Failure | Required behavior |
| --- | --- |
| Missing／invalid `RUN_ID` | 任何API call前拒絕；不自動產生identity |
| Wrong selected config／active labels | mutating commands與status fail closed，列出selected／actual identity |
| Invalid owner／hierarchy inventory | 不發collection或training request；Root／Branch不能被誤當data owner |
| Partial collection POST | 只rollback本次exact四個Leaf request identity；列出未收斂owners |
| Retained data不足、expired或ambiguous | 禁止training；保存evidence後以fresh-run程序處理，不fallback或猜測descriptor |
| Ambiguous Root POST timeout | 同一request GET／idempotent retry；不得換ID建立第二個active plan |
| Branch／Leaf discovery mismatch | complete-required request terminal failure；不得縮小topology |
| Partial hierarchy preparation | 保存每個Branch／Leaf outcome並cleanup已建立upper／lower resources |
| Leaf fitting／lower aggregation timeout | terminal failure；保留round evidence並exact cleanup本run resources |
| Missing／conflicting Branch aggregate | Root拒絕upper aggregation；不得用單一Branch或stale artifact繼續 |
| Final validation failure | 不publication、不更新catalog；保留per-Leaf／Branch evidence |
| Publication failure | 沿用durable retry；terminal failure不把candidate-ready當published |
| Root／Branch／Leaf generation change | fence舊generation與plan；restart後只接受新run或direct recovery evidence |
| Cleanup evidence不足 | 即使top-level為`COMPLETE`仍保持verification-incomplete |
| Active-training ordinary stop | process stopped不能取代2 upper＋4 lower exact cleanup evidence |

## 9. Verification matrix

| Layer | Verification | Required gate |
| --- | --- | --- |
| Static source | selected TESTBED／scenario／native config／manifest／topology exact compare | 1 Root、2 Branches、4 Leaves、8 UEs、7 PyMTLF、FedProx、2 rounds一致 |
| Operator lifecycle | synthetic HTTP／manifest fixtures | Leaf-only collection、Root trigger、idempotency、rollback、wrong config、invalid inventory及HTTP failure fail closed |
| Component contract | pinned PyMTLF focused native tests | static topology、Branch dual role、FedProx、two-tier weighting、publication與cleanup semantics保持 |
| Status／evidence | deterministic log／API／artifact fixtures | Root request、4 Leaf results、round progress、validation、publication與6-resource cleanup不誤判；aggregate identity不由缺少event的log推測 |
| Repository | focused tests、`make test`、applicable disposable container checks、`git diff --check` | 全部通過；local/container evidence不宣稱real 5GC／ADRF acceptance |
| Real collection | approved Host context、real three-VM SMF／UPF path | 四Leaves各兩SUPI有current-run observations，四descriptors retained且peer cleanup完成 |
| Real admission | Root／2 Branches／4 Leaves＋real NRF | exact identities、capabilities、endpoints與complete-required assignment一致 |
| Real training | seven PyMTLF＋seven NWDAFs | 每輪4 positive Leaf updates、2 lower aggregates、1 sample-weighted Root aggregate完成 |
| Real validation／publication | four Leaf validation＋ADRF＋Root catalog | candidate digest、model ID、ADRF reference、validation summary一致，required scopes為0 |
| Real cleanup | collection／training status、logs與unexpected inventory scan | 2 upper＋4 lower resources無stale active state、correlation、plan或cleanup failure |
| Regression | Phase 3 static Flat operator／status paths及所有shared changed paths | Flat lifecycle interpretation保持；shared runtime change有相應real regression |
| Reset／seed | selected guarded reset＋Root restart | seven-volume HFL state清空、其他topology不被清除、canonical seed identity恢復 |

所有real provider commands，包括status／validate類query，都必須使用approved sandbox-outside execution flow；
sandbox／CI只可執行synthetic provider fixtures，不得啟動真實Vagrant／VirtualBox process。

## 10. Phase completion criteria

只有同時滿足下列條件才完成Phase 4：

1. 沿用authoritative`TESTBED`／scenario／renderer／checker／manifest pipeline，沒有新增config source、selector、
   平行script、VM、service、component API或persistence。
2. Operator可用一個`RUN_ID`操作四個exact Leaf collections與一個Root request，且Flat／HFL由selected manifest
   判斷，不靠另一份人工inventory。
3. 四個Leaves各只解析自己的2-SUPI Internal Group；八個SUPI互斥且real current-run collection evidence完整。
4. 四個collection requests完成peer cleanup並保留未過期descriptor；Branch沒有collection ownership，training不
   fallback到Consumer descriptor或未證明data。
5. Root exact admission兩個Branches，每個Branch exact admission兩個Leaves；identity／capability／endpoint與
   generated topology一致，任何缺失均fail closed。
6. 兩輪均收到四份positive-sample FedProx local updates；兩個Branches各完成lower aggregation，Root依effective
   sample counts完成upper aggregation，artifact lineage可直接核對。
7. 四個Leaves完成final validation；candidate、per-Leaf／Branch metadata與aggregate validation evidence完整。
8. Validated candidate成功儲存至ADRF並commit Root catalog；model ID、artifact digest、ADRF reference一致，
   top-level為`COMPLETE`且required scopes為0。
9. 兩個upper與四個lower training resources全部DELETE；沒有stale current-run correlation、plan workspace、
   unexpected active resource或cleanup failure。
10. Failure、timeout、restart、ordinary stop與guarded reset依§8通過；cleanup不足時不得只依`COMPLETE`宣告成功。
11. HFL status不再顯示`milestones=not-evaluated`，top-level resource、two-tier logs與aggregate summary不矛盾。
12. Phase 3 static Flat regressions、repository full checks與required real three-VM HFL evidence通過。
13. Mandatory initial review、fresh-read conformance、user review與獨立commit gates全部完成。

## 11. Initial normative conformance map

| Normative item | Planned production path／evidence | Status |
| --- | --- | --- |
| Reuse Phase 2 static HFL topology | Existing TESTBED／renderer／checker／manifest／lifecycle | Baseline confirmed；current-run identity pending |
| Reuse Phase 3 operator lifecycle | Generalize existing`fl-control.py` and same Make targets | Implementation pending |
| Leaf-only private collection | Manifest role／data-owner exact mapping＋synthetic and real evidence | Implementation pending |
| Root manual trigger and exact hierarchy | Existing private endpoint＋Root／Branch／Leaf topology checks | Implementation pending |
| Two-round FedProx and two-tier weighting | Pinned component contract＋controlled and real run artifacts | Integration evidence pending |
| HFL status interpretation | Extend existing`ml-status` parser and fixtures | Implementation pending |
| ADRF publication and zero-cutover completion | Existing component path＋real ADRF／catalog evidence | Real evidence pending |
| 2 upper＋4 lower cleanup | Component lifecycle logs/API＋unexpected inventory scan | Real evidence pending |
| Failure／restart／reset closure | Deterministic fixtures、controlled runtime與selected reset | Verification pending |
| Review／conformance／commit gates | Policy workflow | Planning user review pending |

## 12. 明確不包含

- PyMTLF duplicate-GET、transport optimization或其他component remediation；由其他workstream管理。
- Communication byte instrumentation、timing instrumentation或paper metrics。
- Flat FedProx、HFL FedAvg、algorithm transport migration或controlled Flat-vs-HFL comparison。
- Dynamic HFL、`hierarchical + monitor_scopes`、topology hot reload、arbitrary depth或Branch自行補選。
- Production Consumer／Model Provision／Model Monitor participant-selection chain、PyAnLF reprovision與generation
  cutover；static completion只要求required scopes為0。
- 自動collection、auto-trigger、auto-stop、auto-reset或single-button workflow。
- Top-level cancel／DELETE API、host-side run ledger、current-run pointer或request database。
- 新增VM、改變placement、合併NF／PyMTLF identity、共享writable state或降低capacity gate。
- Production benchmark、model-quality superiority、real application throughput或communication-efficiency claim。
