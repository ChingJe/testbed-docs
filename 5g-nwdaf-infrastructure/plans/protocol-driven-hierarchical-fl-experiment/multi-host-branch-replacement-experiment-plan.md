# Protocol-driven Hierarchical FL Multi-host Branch Replacement Experiment Plan

日期：2026-09-08

狀態：Ready for User Review；尚未授權 implementation

索引：

- [Protocol-driven Hierarchical FL Experiment Plans](./README.md)
- [5G NWDAF Infrastructure Development Policy](../../development_policy.md)
- [Testbed Plans](../README.md)

Component 設計與實作輸入位於獨立的 `nwdaf-docs` repository：

- `docs/plans/hierarchical-federated-learning/protocol-extension-implementation/Hierarchical NWDAF FL Protocol Extension Implementation Plan.md`
- `docs/plans/hierarchical-federated-learning/protocol-extension-implementation/slices/Slice 3 Branch Replacement without Retained-result Recovery Detailed Plan.md`
- `docs/plans/hierarchical-federated-learning/protocol-extension-implementation/slices/Slice 7 Experiment Metrics and Event Recording Detailed Plan.md`

---

## 1. 背景與已確認結論

`NWDAF`、`PyMTLF` 與 `nwdaf-resources` 已在本機 real-process harness 跑通
protocol-driven Root–Branch–Leaf hierarchy、單一 Branch training 中途失效、Root degraded
aggregation、replacement preparation、Leaf rebind，以及 replacement 從後續 round 恢復貢獻的流程。Component
workstream 尚缺正式 multi-host testbed evidence。

現有 `5G_NWDAF_Infrastructure` 則以三台 VM、完整 5GC／UERANSIM／UPF、UE traffic dataset、production Flat、
static Flat 與 static Hierarchical 三種 deployment 為主。其 machine、Path A／B、service inventory、scenario、dataset、
status 與 reset 流程仍有大量固定假設，也保存多層 local hash／digest contract。直接在這些分支上疊加新實驗，會讓
舊語意與新 protocol-driven topology 同時存在，增加 renderer、validator、lifecycle 與 evidence 的維護成本。

本計畫固定採用下列方向：

1. 此 feature branch 是新版實驗的 breaking cutover，不維持舊 production Flat、static Flat 或 static
   Hierarchical experiment compatibility。
2. 先清除 obsolete experiment path 與非必要 hash contract，再擴充 topology；不把 cleanup 延到實驗完成後。
3. 新部署使用四台 VM：一台 `core` 與三台獨立 Path VM；每條 Path 擁有自己的 Branch 與 Leaves。
4. Host 執行十一個 PyMTLF containers，和十一個 Guest Go NWDAF processes 一對一對應。
5. 先用短 acceptance run 證明 wiring，再執行有足夠 accepted rounds 與 paired seeds 的正式 baseline／failure
   campaign；單一 round 或聚合一次後結束不構成實驗結果。
6. 實驗觀測以 component 已提供的 node-local structured JSONL 為主；完整一般 log 落盤，只在異常時讀取精確
   service 與時間窗，避免長時間串流大量 log。

本計畫是唯一主計畫。三個 slices 都在第 7 節定義，不建立個別 Slice 1／2／3 文件。

---

## 2. 目標

### 2.1 Infrastructure 目標

- 將 machine、Path、Guest service、NWDAF 與 Host container inventory 改為由單一 selected `TESTBED` 完整描述，
  移除 `core`／`path-a`／`path-b` 與 Path A／B 的程式硬編碼。
- 沿用既有 `TESTBED`、scenario、generated `CONFIG_DIR`、manifest、stage／activate 與 lifecycle entrypoints；不新增
  第二套人工 YAML source、top-level config directory、selector、renderer 或 checker。
- 建立四台較小 VM 的 minimal control-plane deployment，只啟動 protocol-driven HFL 所需的 Root／Branch／Leaf
  NWDAFs、NRF、ADRF 與必要 persistence。
- 保留 provider host-context guard、actual process inventory、selected／active config identity、wrong-config prevention、
  exact reset 與 unexpected runtime detection。

### 2.2 Experiment 目標

- 在 multi-host environment 使用 deterministic MNIST Leaf shards、Root validation set 與獨立 final held-out set。
- 驗證三個 active Branch regions 中 Area A Branch 於 training 中途 fail-stop 後，Root 能以 Area B／C 結果繼續
  accepted degraded rounds，並在 replacement ready 後由新的 Area A Branch 從尚未 dispatch 的 round 恢復貢獻。
- 以相同 seed、data shards、initial model、training effort 與 round target 執行 paired no-failure baseline 和
  Branch-failure condition。
- 產生可直接離線分析的 learning curve、round cohort 與 replacement lifecycle evidence，不從一般文字 log
  猜測實驗結果。

### 2.3 Cleanup 目標

- 對 testbed-owned hash／digest 使用點逐一追蹤 producer、consumer 與保護邊界。
- 由 Slice 1 逐項追蹤 producer、consumer、defining contract 與現有替代保障後，才決定每個 hash／digest 的
  keep／remove disposition；主計畫不預先建立具體 allowlist 或 removal list。
- disposition 必須遵守 development policy 的 hash boundary，並確保 runtime identity、wrong-config prevention、
  transport／supply-chain contract、component-native identity 與 destructive safety 不因簡化而失去必要保障。
- 移除只服務舊 Flat／static experiment 的 definitions、render branches、validators、scripts、Make targets、tests
  與 generated examples；歷史文件與已驗證 records 只作 provenance，不表示 source branch 仍支援舊流程。

---

## 3. Scope 與 repository ownership

| Repository | 本計畫角色 | 預期 disposition |
| --- | --- | --- |
| `5G_NWDAF_Infrastructure/` | topology、config generation、VM／Guest／Host lifecycle、dataset placement、experiment controller、evidence collection、tests | 主要 implementation repository |
| `testbed-docs/` | 本主計畫、後續 confirmed design／operations／experiment definition／verified record | 文件 owner |
| `NWDAF/` | 十一個 Guest processes 的 protocol transport 與 NF lifecycle | 使用目前已完成 revision，預設 read-only |
| `PyMTLF/` | FL execution、Branch replacement 與 node-local JSONL recording | 使用目前已完成 revision，預設 read-only |
| `nwdaf-resources/` | canonical topology／policy／runner behavior 的 component evidence 與可參考 fixture | reference/runtime dependency；不複製其 orchestration owner |
| `nrf/` | exact-ID discovery 與 NF registration | runtime dependency，預設 read-only |
| `adrf/` | Root global-model distribution 與 temporary record storage | runtime dependency，預設 read-only |

若 multi-host integration 證明 component contract 本身有 defect，必須先說明跨 repository 資料流、更新本計畫的
affected repositories，並取得使用者確認；不得在 testbed script 內加入平行 workaround。

不修改歷史 `5G_Infrastructure/`，也不以其中 submodule、README、script 或 runtime state 作為本計畫 baseline。

---

## 4. Authoritative source 與 single pipeline

本計畫不新增 deployment selector。資料 ownership 固定如下：

| 資訊 | Authoritative owner | Generated／runtime consumer |
| --- | --- | --- |
| VM、network、Path、Guest service、NWDAF identity／placement、Host container、resource budget | selected complete `TESTBED` YAML | `Vagrantfile`、renderer、manifest、lifecycle、capacity gate |
| training rounds、local epochs、seeds、fault phase minima、observation interval、run condition | selected scenario YAML | renderer、experiment controller、evidence summary |
| recursive topology、candidate priority、policy、strategy、`reportAfter` | selected `TESTBED` 的 logical NWDAF topology section | generated PyMTLF native topology config，再由 production protocol 傳遞 |
| Leaf train shards、Root validation、final held-out dataset | deterministic dataset preparation owned by testbed scenario/run | read-only mounts／paths in generated PyMTLF configs |
| process-native configs、Compose artifact、systemd inventory、manifest | one existing renderer pipeline output in `CONFIG_DIR` | Guest／Host processes 與 lifecycle scripts |
| actual active deployment identity | Guest active marker、Host labels、actual process/container inventory | status、stop、reset、recovery guards |
| experiment observations | 各 PyMTLF node 與 test controller | evidence collector、offline summary／plot |

同一事實不得同時由 `TESTBED` 和 scenario 人工維護。Scenario 只擁有 run behavior；physical placement、logical
participant identity 與 topology ownership 留在 complete `TESTBED`。Renderer 可產生 native component config，
但 generated artifact 不是新的 authoritative source。

---

## 5. 固定部署架構

### 5.1 Physical topology

| Machine | 主要 placement | 說明 |
| --- | --- | --- |
| `core` | Root NWDAF、NRF、ADRF、必要 persistence | 不承載 Branch／Leaf training participant |
| `path-a` | Area A primary Branch、Area A inactive replacement Branch candidate、Leaf A1、Leaf A2 | failure／replacement 實驗路徑 |
| `path-b` | Area B Branch、Leaf B1、Leaf B2 | degraded window 的 surviving region |
| `path-c` | Area C Branch、Leaf C1、Leaf C2 | degraded window 的 surviving region |

VM names 是 selected deployment data，不得再被 renderer、scripts 或 tests 當成全域固定常數。`core` 可以作為此
deployment 的 machine role；Path inventory 必須可由 YAML 枚舉，因此加入 `path-c` 不需要再新增 A／B 專用 branch。

實作前由 capacity gate 根據 host 實際 RAM、CPU、disk 與 container overhead 決定每台 VM 的精確配額。已確認方向是
移除 UPF／gNB／UE／AMF／SMF 等不屬於本實驗 dependency chain 的負載，使四台 VM 可以小於舊 full-core topology；
不得先在文件中虛構保證可行的固定 MiB／CPU 數值。

### 5.2 Logical process inventory

| Role | Go NWDAF processes | Host PyMTLF containers | Initial state |
| --- | ---: | ---: | --- |
| Root | 1 | 1 | active |
| Area A primary Branch | 1 | 1 | active |
| Area A replacement Branch | 1 | 1 | registered candidate；training 前不在 active cohort |
| Area B／C Branches | 2 | 2 | active |
| Six Leaves | 6 | 6 | active |
| **Total** | **11** | **11** | one-to-one mapping |

每個 NWDAF 必須有獨立 NF Instance ID、SBI endpoint、internal MTLF endpoint、config、systemd unit、journal identity、
NRF registration、writable state 與對應 PyMTLF container／volume。共同 physical VM、binary 或 image 不表示可共用
logical identity 或 writable state。

### 5.3 Area A priority-based replacement contract

Area A 的 primary 與 replacement Branch 必須位於同一個 topology-owned candidate pool，並使用 component 已有的
`selection_method: priority`：

| Candidate | Priority | Selection semantics |
| --- | ---: | --- |
| Area A primary Branch | 100 | 初始最高順位候選；通過 fresh exact-ID resolve 與 preparation 後成為 active Branch |
| Area A replacement Branch | 50 | 初始保留為次順位候選；primary 發生可恢復的 availability failure 後才成為下一個 selection target |

Root 擁有 replacement decision。Primary 失效後，Root 必須先從 active cohort 淘汰失敗 identity，再以同一 candidate
pool、相同 priority ordering 與 fresh NRF exact-ID resolve 選擇下一順位；不得由 test controller、machine placement、
檔名或 hard-coded replacement identity 直接指定替代對象。若未來同組加入更多 candidates，也依其 declared priority
沿用相同 selection semantics。

Test controller 只負責停止 exact primary processes，以及暫時控制 replacement backend availability來建立可觀測的
degraded window。Controller 在 event barrier 後恢復 backend，不向 Root 傳送「選擇此 Branch」的指令，也不修改
candidate priority。Replacement 完成 production preparation 後，只從下一個尚未 dispatch 的 round 加入。

### 5.4 Minimal core dependencies

正式 topology 只納入經 current component trace 證明需要的 service：

- NRF：所有 NWDAFs 與 ADRF 的 registration／discovery，以及 replacement candidate fresh exact-ID resolve。
- ADRF：Root global-model record 與 Branch／Leaf model retrieval。
- MongoDB 或 ADRF／NRF current revision 實際要求的 persistence。
- 十一個 Go NWDAF 與十一個 PyMTLF runtime。

UPF、gNB、UE、UERANSIM、AMF、SMF、NSSF、UDR、UDM、AUSF、PCF、WebConsole、PyAnLF、consumer、subscriber fixture、
PseudoDriver 與 UE communication dataset 預設不納入。若 source trace 發現其中任何 service 是 current protocol
path 的必要 dependency，屬於 architecture contradiction，必須先回報並更新計畫，不得靜默帶回完整 5GC。

---

## 6. Existing-flow baseline disposition

Canonical existing flow 是目前 `TESTBED` + scenario → `config-create` → `CONFIG_DIR` → validate／stage／activate →
Guest services + Host containers → explicit training control → status／logs → stop／reset。Breaking cutover 改變的是
experiment semantics 與 inventory，不另建第二條 pipeline。

| Baseline stage | Disposition | 本計畫要求 |
| --- | --- | --- |
| 1. authoritative deployment selection | adapted | 沿用 `TESTBED` selector與complete YAML；schema改為inventory-driven四機protocol topology |
| 2. experiment／traffic inputs | explicitly replaced | 沿用scenario入口，但以MNIST campaign／seed／fault phases取代UE traffic、monitoring與degradation輸入 |
| 3. config generation／rendering | adapted | 保留單一renderer；移除production-flat／static-flat／static-hierarchical render branches與fallback |
| 4. strict validation／native loaders | adapted | 以new inventory、protocol topology、dataset split、round phases與component-native loader驗證；移除legacy/hash-only checks |
| 5. generated artifacts／manifest | adapted | manifest保存exact runtime/reset/capacity inventory與必要provenance；hash fields依Slice 1 consumer trace決定 |
| 6. infrastructure／machine lifecycle | adapted | Vagrant與provider guard改由selected machine inventory驅動，支援core + three Paths |
| 7. config stage／activate／network | adapted | 保留單一config identity、cross-VM activation與partial activation detection；network只建立實驗需要的management／SBI reachability |
| 8. Guest service lifecycle | explicitly replaced | service inventory改為NRF、ADRF、persistence與十一個NWDAFs；移除full-5GC／RAN／UPF lifecycle |
| 9. Host auxiliary runtime | adapted | Compose改為十一個one-to-one PyMTLF containers；保留exact status／logs／stop與unexpected runtime detection |
| 10. subscriber／dataset／traffic | explicitly replaced | 移除subscriber／UE／PseudoDriver資料鏈；改為deterministic MNIST Leaf train、Root validation與final held-out data |
| 11. trigger | explicitly replaced | 移除collection／consumer／degradation trigger；使用protocol-driven manual training request與controller event barriers |
| 12. reset／restart／evidence | adapted | exact清理十一個node states、ADRF／NRF experiment state、container volumes與records；保留seed重建與failure evidence |

舊文件與 records 可以保留作 provenance；current source、help、README 與 tests 不得再宣稱舊 deployment mode 仍可執行。

---

## 7. Implementation slices

### 7.1 Slice 1 — Legacy experiment 與 hash cleanup

#### Operator-visible outcome

Repository 只呈現新 protocol-driven experiment 的單一 production path。舊 Flat／static topology 選項、UE traffic
pipeline與其命令不再出現在 help、README、examples 或 repository test contract。Config validation 不再因 trusted
local sources 的 nested digest chain 產生無關 mismatch 或重建工作。

#### 工作範圍

1. 建立 testbed-owned hash inventory，對每個 occurrence 記錄 producer、consumer、保護的 failure、是否屬於
   external/component contract，以及 keep／remove disposition。
2. 對每個 occurrence 記錄 proposed disposition 與理由；keep 必須指出 defining contract 與無法由既有機制取代的
   failure，remove 必須列出原本保障、現有重複機制及必要的 semantic／lifecycle replacement。沒有完成 consumer
   trace 的項目保持 open，不得因名稱含有 checksum、hash 或 digest 就直接分類。
3. 盤點時明確區分 external supply-chain／transport、trusted local transfer、selected／active runtime identity、
   Docker-native identity、component-native artifact identity、local provenance、cache key 與純命名用途；分類本身
   不代表 keep 或 remove 結論。
4. 必要替代保障優先使用 YAML／JSON loading、schema、type、shape、path containment、referential integrity、unique
   identity／endpoint、inventory exact comparison、dataset sample/class/shard semantics、atomic activation 與 lifecycle
   state validation。
5. 先完成 read-only hash disposition table；無法由direct evidence明確判斷的項目必須保持open並請使用者決策，
   不能先改source。主計畫不預先鎖定任何具體field、helper或checksum的去留。
6. 移除舊 complete testbed definitions、legacy scenario／traffic examples、renderer topology branches、static FL control、
   subscriber／dataset／PseudoDriver flow、full-core-only service logic、obsolete Make targets、docs與tests。
7. 保留通用 provider guard、VM process inventory、config activation、runtime identity、capacity、status、stop、reset 與
   destructive safety helpers；若 helper 同時服務新舊流程，先拆掉舊 branch，不複製共用邏輯。

#### Focused verification

- Hash inventory 中每個 occurrence 都有 consumer trace 與 disposition，且 production code 不再引用被移除 fields。
- Synthetic config fixtures 驗證 disposition 後仍能偵測 wrong-config、partial activation 與 Host runtime identity
  mismatch；不預設必須由 digest 實作。
- Semantic invalid cases涵蓋missing reference、duplicate identity／endpoint、unknown machine、dataset shape／class／shard
  錯誤與empty runtime inventory。
- Repository搜尋與help／README review確認不再暴露舊 topology selectors或experiment commands。
- Provider guard測試只使用synthetic fixture／mock，不在sandbox內啟動real Vagrant／VirtualBox process。

#### Slice acceptance

- 新 topology implementation 不再依賴任何待移除 hash 或 legacy experiment branch。
- 每個 hash occurrence 都有direct evidence支持的keep／remove disposition；保留項目有defining contract，移除項目的
  必要保障已有semantic或lifecycle替代。
- Common lifecycle safety tests 通過；不要求舊 Flat／static experiment regression 通過。
- 完成 mandatory initial review、user-review handoff與獨立 commit proposal／approval後，才進入 Slice 2
  implementation。

### 7.2 Slice 2 — Inventory-driven four-VM protocol topology

#### Operator-visible outcome

Operator 透過同一組 `config-create`、validate、VM、services、ML、status、logs、stop、reset 入口，可以建立、啟動與
檢查 `core` + Area A／B／C 四台 VM、十一組 NWDAF↔PyMTLF runtime，以及所需 NRF／ADRF dependencies。Topology
切換、unexpected runtime 或 partial activation 都會 fail closed，不會靜默操作其他部署。

#### 工作範圍

1. 將 `Vagrantfile`、host／guest config helpers、network generation、service install、stage／activate、status／logs、
   reset與tests中的固定machine／Path loops改為selected manifest inventory驅動。
2. 在既有complete `TESTBED` schema內定義第5節的physical/logical topology；替換舊definitions，不新增selector。
3. 單一renderer產生：
   - 每個NWDAF獨立native config與systemd unit；
   - 每個PyMTLF獨立config、volume、port與read-only dataset mount；
   - protocol recursive topology、Root／Branch policy、FedProx strategy與`reportAfter`；
   - exact Guest services、Host containers、volumes、ports、reset scope與capacity manifest。
4. 以 `nwdaf-resources` branch-replacement profile的current component contract為語意輸入：一Root、三active regional
   Branches、Area A一inactive replacement、六Leaves；Area A使用同一candidate pool的priority `100` primary與
   priority `50` replacement，Root透過`selection_method: priority`完成初始選擇與故障後重新選擇。Root與Branch皆
   使用configured policy、FedProx `mu`、sample-weighted aggregation，而非testbed自行補hard-coded protocol metadata。
5. 建立 deterministic MNIST partition：六份Leaf training shards、一份Root per-round validation、一份與validation分離的
   final held-out set。Dataset paths只進local config／mount，不進Model Training protocol；相關hash處置依Slice 1
   disposition table，不在本節預判。
6. 加入NTP／chrony與clock-skew preflight，使跨VM JSONL event時間可對齊；clock未同步時不開始正式campaign。
7. 讓capacity gate由selected inventory計算四台VM、十一個containers、build overhead、storage與GPU／CPU policy；
   配額不足必須在`vm-up`或container start前失敗。
8. Lifecycle涵蓋render、validate、stage、activate、start、status、logs、stop、restart與reset；每一階段都核對selected／
   active identity及actual process/container inventory。

#### Failure 與 recovery behavior

- Invalid、missing或empty machine／service／container inventory：fail closed。
- 某Guest config activation失敗：不得宣稱deployment active；status顯示成功與失敗machines，operator可用同一selected
  config重試，或在explicit stop/reset後重新生成。
- Guest service或Host container部分start失敗：停止已由該次operation啟動的in-scope runtimes，保留診斷evidence，
  不擴大到unexpected runtime。
- Wrong selected config、unexpected unit/container/volume、provider/process mismatch：正常stop/reset拒絕執行並顯示
  exact mismatch；事故cleanup需要fresh inventory與額外使用者批准。
- Real provider command一律經approved host-context flow；sandbox內的直接或間接Vagrant／VBoxManage path都不得存在。

#### Focused verification

- 四機render／validate fixture及invalid inventory tests。
- 十一組NWDAF↔PyMTLF identity、port、volume、config與placement one-to-one checks。
- Generated component configs通過current NWDAF／PyMTLF native loader或等價non-starting validation。
- Mock lifecycle涵蓋partial activation、start rollback、unexpected runtime、wrong-config stop/reset、empty targets與capacity
  rejection。
- Approved host context執行real四VM create/start/status，確認OS process inventory與provider state一致。
- Real Guest／Host integration確認NRF registrations、ADRF health、十一個NWDAF與十一個PyMTLF health，以及跨VM／Host
  endpoint reachability。
- Stop／restart／reset後確認exact state、volume、ADRF model records、NRF experiment registrations與seed restoration。

#### Slice acceptance

- 四台VM與十一組runtime可由同一authoritative source完整重建並exact compare。
- Minimal dependency graph在real environment啟動成功；未啟動舊full 5GC／RAN／UPF services。
- Canonical short topology preparation與至少兩個accepted hierarchical rounds跑通，證明dataset、ADRF、protocol與
  JSONL wiring；這只是Slice 2 integration acceptance，不是正式實驗結果。
- 所有required lifecycle evidence完成mandatory review並交付user review；缺real provider evidence時狀態只能是
  `Implementation Complete / Verification Incomplete`。

### 7.3 Slice 3 — Formal paired Branch replacement campaign

#### Operator-visible outcome

Operator可用一個canonical experiment command選擇baseline或failure condition，執行多round HFL、觀察精簡milestones、
停止Area A primary Branch、以event barrier控制replacement release、收集各node JSONL，並輸出可報告的raw evidence、
CSV、plots與run summary。Runner不需要持續把完整journald／Docker logs輸出到terminal。

#### Campaign structure

正式campaign前先跑一個較小epoch／round的acceptance profile，只驗證wiring、fault controller、record collection與
cleanup。Acceptance profile不得納入結果比較。

正式campaign至少執行三組paired seeds；若時間允許可擴充到五組。每個seed都有：

| Condition | 固定內容 | 差異 |
| --- | --- | --- |
| no-failure baseline | 同seed、Leaf shards、initial model、validation/test sets、local epochs、FedProx `mu`、policy與accepted round target | 不停止Branch，三個regions持續參與 |
| Branch-failure／replacement | 與paired baseline完全相同 | 在已完成warm-up後停止Area A primary Branch及其PyMTLF，暫停replacement backend，之後依event barrier放行 |

Failure condition的最小accepted-round結構：

1. 至少5個normal accepted rounds後才注入failure。
2. Controller確認Area A primary Go NWDAF與PyMTLF都已停止後，寫入`BRANCH_PROCESS_STOPPED`。
3. Replacement backend維持不可用，直到Root至少完成5個只含surviving regions的accepted degraded rounds。
4. Controller依observed `ROOT_ROUND_OUTCOME` barrier放行replacement，不使用固定sleep推測進度。
5. `BRANCH_REPLACEMENT_READY`後，replacement只加入下一個尚未dispatch的cohort。
6. Replacement第一次出現在`successfulNfInstanceIds`後，至少再完成10個accepted restored rounds。

因此正式failure run的campaign config必須容納至少20個accepted rounds，並為failure detection、preparation retry與
rejected attempts預留timeout；round number不能只靠wall-clock推定。Baseline使用相同accepted round target。若實際
training時間不允許此最低結構，不得自行降為單round或只剩一個degraded點；必須回報並由使用者決定epochs、sample
size、seed數或campaign scope的調整。

#### Fault 與 production boundary

- Fault injection只由test controller在runtime外停止exact Area A primary Go NWDAF與PyMTLF targets。
- Controller只能控制primary／replacement processes的availability與放行時序；它不得指定replacement identity、改寫
  priority或繞過Root candidate selection。Root必須從Area A同一candidate pool依priority選出replacement。
- 不在NWDAF／PyMTLF production code新增sleep、kill switch、failure endpoint或實驗專用recovery shortcut。
- Root依production timeout／availability分類與configured completion policy接受degraded round；controller不偽造
  component event。
- Replacement沿用同一`mlCorreId`與same Leaf subtree，但建立新的per-edge resource及`notifCorreId`；不使用
  retained-result recovery。
- Old Branch不在同run內恢復或hand back；同時多Branch失效、Leaf replacement與Root restart recovery不是本campaign。

#### Structured observation contract

必收資料：

- Root `MODEL_EVALUATION`：`ROOT_INITIAL`與每個accepted `ROOT_GLOBAL`的validation loss／accuracy。
- Root `ROOT_ROUND_OUTCOME`：每個attempt的selected、successful、failed Branch identities與accepted flag。
- Root `BRANCH_FAILURE_DETECTED`。
- Root `BRANCH_REPLACEMENT_READY`。
- Controller `BRANCH_PROCESS_STOPPED`。
- Replacement首次貢獻：由第一筆將replacement NF Instance ID列入`successfulNfInstanceIds`的accepted outcome推導。
- Training結束後使用獨立held-out set的final evaluation。
- Run metadata：實際repository revisions、dirty flags、selected config identity、container image IDs、VM inventory、
  scenario、seed、condition、timestamps與exit status。

一般文字log是診斷資料，不是loss／accuracy或round cohort的authoritative source。

#### Quiet monitoring 與 token budget

- Journald與Docker完整logs直接保存到run evidence directory，不以follow模式持續送入terminal或agent context。
- Runner只輸出state transitions：prepared、training started、accepted-round counters、fault stopped、failure detected、
  degraded minimum reached、replacement released／ready／first contribution、target rounds completed、cleanup result。
- 長時間執行每30–60秒輸出一行compact status；沒有state change時不重印node-by-node inventory。
- 正常監控只解析新增JSONL records與process exit／health摘要，不反覆讀取整份record或log。
- 發生timeout或failure時，才擷取exact service、bounded time window與bounded tail；若仍不足，再逐步擴大，不一次讀取
  全部十一個nodes的完整logs。
- 最終產出machine-readable summary，讓後續review讀summary與selected raw records即可，不靠重播terminal log。

#### Analysis outputs

每個run至少保存：

```text
<campaign>/<seed>/<condition>/
├── metadata.json
├── controller-events.jsonl
├── nodes/<nfInstanceId>/observations.jsonl
├── logs/<machine-or-container>/...
├── rounds.csv
├── learning-curve.csv
└── summary.json
```

Campaign層級再產生paired summary與plots，呈現：

- Root validation loss／accuracy對accepted global round；
- normal、degraded與restored phase標記；
- stop→failure detected、stop→replacement ready、stop→first replacement contribution的latency；
- 每個accepted round的successful Branch count／identities；
- final held-out accuracy的paired差異。

本計畫只要求描述觀察結果與系統流程，不預先承諾統計顯著性、模型品質改善或failure優於baseline。

#### Slice acceptance

- 短acceptance profile完整跑通，但和正式結果分開保存。
- 至少三組paired seeds都符合minimum phase rounds、相同controlled inputs與完整structured evidence。
- Failure runs在replacement pending期間持續產生accepted degraded aggregates，且replacement只於後續cohort恢復。
- 每個accepted `ROOT_ROUND_OUTCOME`都有對應Root global evaluation；rejected attempt不得偽造evaluation point。
- Final held-out evaluation與per-round validation dataset分離。
- 每個run可由metadata、JSONL與summary重建round／event timeline，不需要解析一般文字log。
- Stop、reset與evidence preservation驗證完成；required evidence通過mandatory review與使用者確認後，才能建立
  verified record並將本計畫標為completed。

---

## 8. Verification matrix

| Requirement | Static／mock evidence | Real-environment evidence | Owner slice |
| --- | --- | --- | --- |
| Hash disposition與consumer trace | inventory review、repository search、focused tests | disposition涉及runtime boundary時執行對應cross-boundary check | 1 |
| Legacy path removal | source/help/docs/test inventory | new branch不啟動legacy services | 1 |
| Inventory-driven四機render | valid／invalid YAML fixtures、manifest exact comparison | four-VM inventory matches OS/provider state | 2 |
| 十一組one-to-one runtime | config／port／volume／identity uniqueness tests | health、NRF registrations、reachability | 2 |
| Partial activation／wrong config | mock stage/activate/start/stop/reset tests | controlled failed activation or equivalent approved host test | 2 |
| Provider safety | synthetic guard／process fixtures only | approved host-context preflight before every real provider command | 1、2 |
| Minimal NRF／ADRF dependency | generated service inventory tests | real registration、discovery、store/retrieve/cleanup | 2 |
| Deterministic MNIST partitions | shape、class、disjoint partition與seed reproducibility tests | mounted paths與native loader acceptance | 2 |
| Short protocol acceptance | runner/parser tests | topology ready與至少2 accepted rounds | 2 |
| Priority-based replacement與fault barrier | candidate ordering、controller-boundary、JSONL stream／timeout／bounded-log tests | exact stop、5 degraded、Root選出下一priority、replacement release／ready／contribution | 3 |
| Paired campaign integrity | metadata／condition diff checker | at least 3 complete paired seeds | 3 |
| Evidence completeness | schema、round/evaluation consistency checks | raw JSONL、summary、CSV、plots、final held-out result | 3 |
| Exact cleanup／seed restoration | mock destructive-scope tests | actual Guest／Host／NRF／ADRF／volume/reset evidence | 2、3 |

Repository tests不能取代required real provider、VM、container、NRF、ADRF與multi-host experiment evidence。任何real
evidence缺失時，對應slice保持open state。

---

## 9. Normative conformance map

Implementation期間必須維護下列mapping；每個item都要列出production path、deterministic test、verification
result與open gap：

| ID | Normative item | 預期 owner |
| --- | --- | --- |
| C1 | 單一 `TESTBED` + scenario + `CONFIG_DIR` pipeline，無新增selector／parallel renderer | config renderer／checker |
| C2 | Breaking cutover移除舊experiment paths，但保留common safety | Slice 1 diff與repository tests |
| C3 | Hash occurrence逐項具有consumer trace、defining contract或semantic replacement，不由主計畫預判去留 | hash disposition table與focused tests |
| C4 | Machine／Path／service loops全部由selected inventory驅動 | configlib、Vagrantfile、host／guest lifecycle |
| C5 | 四VM、十一NWDAF、十一PyMTLF exact one-to-one inventory | generated manifest與runtime status |
| C6 | 每個logical node有獨立identity、endpoint、state、logs與volume | renderer、systemd、Compose、validators |
| C7 | Minimal core只包含current protocol dependencies | service inventory與real startup evidence |
| C8 | Config identity、partial activation、wrong-config、unexpected runtime與reset fail closed | lifecycle guards與tests |
| C9 | Real provider永不在sandbox內啟動，所有入口經共同guard | command-path review、synthetic tests、host evidence |
| C10 | MNIST train／validation／test分離且可由seed重建 | dataset preparation與semantic checks |
| C11 | Root／Branch policy、FedProx與`reportAfter`來自authoritative topology並走production protocol | generated native config、component loader、E2E records |
| C12 | Controller只控制process availability；Root從同組candidate pool依priority完成replacement selection | topology、controller implementation與event evidence |
| C13 | 5 normal + 5 degraded + 10 restored accepted-round minimum | scenario validation與run summary checker |
| C14 | Paired conditions除fault以外的controlled inputs一致，至少3 seeds | campaign metadata diff／summary |
| C15 | JSONL是round、metric與event evidence；一般log只作diagnostic | collector、schema checker與review |
| C16 | Quiet monitor使用incremental records與bounded diagnostic reads | runner tests與actual terminal transcript summary |
| C17 | Stop／reset只處理selected-and-active exact scope，evidence在cleanup前安全收集 | reset plan、actual runtime comparison與record preservation |

---

## 10. 明確非目標

- 修復或保留舊 production Flat、static Flat、static Hierarchical、UE traffic、degradation或WebConsole experiment。
- 執行舊 experiment regression suite；只保留仍適用的common infrastructure safety tests。
- 修改FedProx演算法、Branch aggregation frequency或component protocol schema。
- Retained-result recovery、Leaf replacement、同時多Branch failure、Root restart recovery或old Branch handback。
- 完整5GC user plane、UERANSIM、subscriber、UPF Event Exposure或UE communication analytics。
- `5g-viz`、Grafana、Prometheus、remote write或live dashboard整合。
- FL communication bytes、CPU、GPU、network cost或能源instrumentation。
- 長期results retention service、central metrics service或自動論文統計推論。

這些項目若後續需要，應另行設計，不得在本次三個slices中順手加入。

---

## 11. Decision gates 與 blocker handling

只有下列情況停止並請使用者決策：

- current component trace證明minimal topology仍需要本計畫排除的5GC service或新的external dependency；
- 四VM／十一containers無法通過capacity gate，必須減少participants、合併identity、改變isolation或新增host；
- 既有 `TESTBED`／scenario／renderer無法在不產生雙重source的前提下合理擴充；
- multi-host behavior需要修改NWDAF／PyMTLF／NRF／ADRF contract或production recovery semantics；
- 正式campaign無法滿足5 normal + 5 degraded + 10 restored、至少3 paired seeds或required evidence；
- 必須弱化provider、wrong-config、exact reset、partial activation或unexpected runtime safety；
- 需要清除不在selected exact scope內的existing VM、container、volume或state。

Blocker回報必須列出原假設、直接反證、受影響slice、可行選項、建議與tradeoff。非阻塞finding分類為future
work、legacy cleanup、optional hardening、integration gap或unconfirmed risk，不擴大current slice。

---

## 12. Review、commit 與進度規則

三個slices依序執行，但不另建slice計畫文件：

```text
Slice 1 cleanup
→ focused verification
→ mandatory initial review
→ user review
→ commit proposal／approval
→ Slice 2 topology
→ focused + real-environment verification
→ mandatory initial review
→ user review
→ commit proposal／approval
→ Slice 3 campaign
→ evidence review
→ user review
→ verified record／completion proposal
```

每個slice都保持changes unstaged／uncommitted到user review完成。User review不等於commit approval；commit與push各自
需要獨立批准。不得為了「一次跑完」略過slice boundary的review，但在未遇到decision gate時，可以在同一workstream
連續完成已批准slice的implementation、verification與initial review。

進度只在本節更新，不複製到分類README：

| Slice | 狀態 | Required evidence |
| --- | --- | --- |
| 1. Legacy／hash cleanup | Not Started | inventory、semantic replacements、common safety tests、review |
| 2. Four-VM topology | Not Started | render／lifecycle tests、approved real four-VM integration、review |
| 3. Formal campaign | Not Started | short acceptance、至少3 paired seeds、structured evidence、analysis、review |

本計畫只有在三個slices均完成required evidence、user review，且正式結果已整理到`records/`後，才能標為
`Completed`。若code完成但real testbed evidence未完成，狀態使用
`Implementation Complete / Verification Incomplete`。
