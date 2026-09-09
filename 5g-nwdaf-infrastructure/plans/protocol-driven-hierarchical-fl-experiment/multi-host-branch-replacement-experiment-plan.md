# Protocol-driven Hierarchical FL Multi-host Branch Replacement Experiment Plan

日期：2026-09-08

最近更新：2026-09-09

狀態：Slice 1 Completed；Slice 2 Not Started

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

1. 此 feature branch 以新版 protocol-driven experiment 作為唯一會持續維護與正式驗證的 canonical target；既有
   production Flat、static Flat／Hierarchical source與操作資產保留為legacy／unverified，不保證仍可執行，也不投入
   專屬維護或驗證。
2. 先把legacy assets和canonical runtime明確分離，移除會造成隱式選取的default／fallback及非必要hash，再擴充
   topology。Canonical不使用某項舊資產本身不是刪除理由；只有具體阻擋common path、繞過安全邊界、或已證明為
   無consumer重複產物的項目才可移除。
3. 新部署使用四台 VM：一台 `core` 與三台獨立 Path VM；每條 Path 擁有自己的 Branch 與 Leaves。
4. Host 執行十一個 PyMTLF containers，和十一個 Guest Go NWDAF processes 一對一對應。
5. MNIST與CIFAR-10先各用短normal acceptance run證明dataset與topology wiring，再各執行一次至少包含2個normal、
   2個degraded與2個restored accepted rounds的Branch replacement flow；本計畫不執行paired baseline、multi-seed或
   統計性model-quality campaign，單一round或聚合一次後結束也不構成流程驗證。
6. 實驗觀測以 component 已提供的 node-local structured JSONL 為主；完整一般 log 落盤，只在異常時讀取精確
   service 與時間窗，避免長時間串流大量 log。

本文件是跨 Slice 的唯一主計畫，保存共同架構、依賴、驗收與進度。詳細計畫採漸進式建立：只在某個 Slice 即將
進入實作前完成該 Slice 的盤點與獨立 review；目前只建立 Slice 1 詳細計畫，不預先建立 Slice 2、3 文件。

---

## 2. 目標

### 2.1 Infrastructure 目標

- 將 canonical／common reachable path 的 machine、Path、Guest service、NWDAF 與 Host container inventory 改為由
  單一 selected `TESTBED` 完整描述，移除該路徑對 `core`／`path-a`／`path-b` 與 Path A／B 的程式硬編碼；隔離的
  legacy-only implementation可保留既有固定值，不要求repository-global歸零。
- 沿用既有 `TESTBED`、scenario、generated `CONFIG_DIR`、manifest、stage／activate 與 lifecycle entrypoints；不新增
  第二套人工 YAML source、top-level config directory、selector、renderer 或 checker。
- 建立四台較小 VM 的 minimal control-plane deployment，只啟動 protocol-driven HFL 所需的 Root／Branch／Leaf
  NWDAFs、NRF、ADRF 與必要 persistence。
- 保留 provider host-context guard、actual process inventory、selected／active config identity、wrong-config prevention、
  exact reset 與 unexpected runtime detection。

### 2.2 Experiment 目標

- 在multi-host environment分別使用deterministic MNIST與CIFAR-10 Leaf shards、Root validation set及獨立final
  held-out set，並確認兩種dataset都能通過native loader與實際training flow。
- 驗證三個 active Branch regions 中 Area A Branch 於 training 中途 fail-stop 後，Root 能以 Area B／C 結果繼續
  accepted degraded rounds，並在 replacement ready 後由新的 Area A Branch 從尚未 dispatch 的 round 恢復貢獻。
- MNIST與CIFAR-10各執行一個Branch-failure／replacement run；每個run至少包含2個normal、2個degraded與2個restored
  accepted rounds，並可縮小sample count與local epochs以控制時間。
- 產生可直接離線檢查的learning curve、round cohort與replacement lifecycle evidence，不從一般文字log猜測流程結果，
  也不把兩個短run解讀成模型品質、跨dataset優劣或統計結論。

因此本計畫最低共有四個real training runs：Slice 2的MNIST與CIFAR-10 normal wiring各一個，以及Slice 3的MNIST與
CIFAR-10 replacement flow各一個。Slice 3所稱「6」是每個replacement run內的accepted round數，不是run數量。

### 2.3 Cleanup 目標

- 對 testbed-owned hash／digest 使用點逐一追蹤 producer、consumer 與保護邊界。
- 具體 producer、consumer、defining contract、替代保障及 keep／remove 理由由
  [Slice 1 詳細計畫](./slices/slice-1-legacy-and-hash-cleanup-detailed-plan.md)保存；主計畫只保留跨 Slice 的
  cleanup 原則與驗收責任。
- disposition 必須遵守 development policy 的 hash boundary，並確保 runtime identity、wrong-config prevention、
  transport／supply-chain contract、component-native identity 與 destructive safety 不因簡化而失去必要保障。
- 保留既有production Flat、static Flat／Hierarchical definitions、component／provisioning assets、操作script與重建
  其意圖所需的設定輸入，並明確標示為legacy／unverified；它們不構成canonical runtime、regression baseline或
  real-environment acceptance，也不保證仍可執行。
- Legacy／unverified支援狀態只由`5G_NWDAF_Infrastructure/README.md`的legacy asset inventory說明；不為此新增
  `TESTBED`／scenario／manifest欄位、特殊檔名規則、opt-in flag或另一條lifecycle status branch。
- 不為legacy profiles新增第二套renderer／lifecycle或投入相容性修復。現有validators、scripts、Make targets、tests、
  submodules與provisioning branches原則上保留；canonical path必須不選取、不部署、不驗收它們。只有能指出確切
  canonical／common reachability衝突、安全風險或無consumer重複產物時，才可在Slice 1提出逐項移除。
- 既有WebConsole維持獨立optional feature，保留其source、config、build／lifecycle targets與既有tests；canonical
  protocol-driven profile將它關閉，本計畫不部署或驗收，但不能只因新實驗未使用就刪除或改列legacy。
- Slice 2建立inventory-driven schema後，可用同一pipeline低成本遷移legacy profiles。只有通過共通parse、schema、
  reference、inventory與destructive-safety validation的profile才可由`TESTBED`選取；其餘仍保留為不可啟動的參考資產。

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
| dataset kind、training rounds、local epochs、seed、fault phase minima、observation interval | selected scenario YAML | renderer、experiment controller、evidence summary |
| recursive topology、candidate priority、policy、strategy、`reportAfter` | selected `TESTBED` 的 logical NWDAF topology section | generated PyMTLF native topology config，再由 production protocol 傳遞 |
| MNIST／CIFAR-10 Leaf train shards、Root validation、final held-out dataset | deterministic dataset preparation owned by testbed scenario/run | read-only mounts／paths in generated PyMTLF configs |
| process-native configs、Compose artifact、systemd inventory、manifest | one existing renderer pipeline output in `CONFIG_DIR` | Guest／Host processes 與 lifecycle scripts |
| actual active deployment identity | Guest active marker、Host labels、actual process/container inventory | status、stop、reset、recovery guards |
| experiment observations | 各 PyMTLF node 與 test controller | evidence collector、offline summary／plot |

同一事實不得同時由 `TESTBED` 和 scenario 人工維護。Scenario 只擁有 run behavior；physical placement、logical
participant identity 與 topology ownership 留在 complete `TESTBED`。Renderer 可產生 native component config，
但 generated artifact 不是新的 authoritative source。

Slice 2建立canonical definition後，Make的default `TESTBED`指向該definition，未提供`CONFIG_DIR`時由其
`config.directory`解析同一份generated runtime。Operator不需在每個lifecycle command重打長路徑；明確
`CONFIG_DIR`只作override。Canonical output不存在、selection不完整或identity不一致時fail closed，不回退至
`config/default`。

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
不部署 UPF／gNB／UE／AMF／SMF 等不屬於本實驗 dependency chain 的負載，使四台 VM 可以小於舊 full-core topology；
相關source、config與legacy lifecycle仍可留在repository。不得先在文件中虛構保證可行的固定 MiB／CPU 數值。

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
其中WebConsole仍保留為repository既有的optional feature；「不納入」只表示canonical experiment不啟用，不表示移除。

---

## 6. Existing-flow baseline disposition

Canonical existing flow 是目前 `TESTBED` + scenario → `config-create` → `CONFIG_DIR` → validate／stage／activate →
Guest services + Host containers → explicit training control → status／logs → stop／reset。Breaking cutover 改變的是
experiment semantics 與 inventory，不另建第二條 pipeline。

| Baseline stage | Disposition | 本計畫要求 |
| --- | --- | --- |
| 1. authoritative deployment selection | adapted | 沿用 `TESTBED` selector與complete YAML；新canonical deployment改為inventory-driven四機protocol topology。既有production／static definitions保留為legacy／unverified，但不得成為implicit default或fallback |
| 2. experiment／traffic inputs | adapted and replaced for canonical flow | 新canonical scenario以MNIST／CIFAR-10 dataset selection、seed與fault phases取代UE traffic；既有scenario、traffic與dataset inputs原樣保留於legacy flow，不作canonical run source |
| 3. config generation／rendering | adapted | Canonical topology擴充既有renderer與common pipeline；移除隱式default／fallback，不因未使用而刪除dormant legacy branches。Legacy profile若遷移，只能使用同一inventory-driven pipeline |
| 4. strict validation／native loaders | adapted | Canonical path以new inventory、protocol topology、dataset split、round phases與component-native loader驗證；legacy-only validation可留在隔離路徑，common path不再依賴其hash-only checks |
| 5. generated artifacts／manifest | adapted | manifest保存exact runtime/reset/capacity inventory；canonical runtime identity與其hash boundary依Slice 1詳細計畫，source revision／dirty flag由run evidence記錄 |
| 6. infrastructure／machine lifecycle | adapted | Vagrant與provider guard改由selected machine inventory驅動，支援core + three Paths |
| 7. config stage／activate／network | adapted | 保留單一config identity、cross-VM activation與partial activation detection；network只建立實驗需要的management／SBI reachability |
| 8. Guest service lifecycle | canonical inventory replaced | Canonical selected inventory只啟動NRF、ADRF、persistence與十一個NWDAFs；既有full-5GC／RAN／UPF implementation保留於legacy flow，不得被canonical start／status／stop／reset隱式納入 |
| 9. Host auxiliary runtime | adapted | Canonical generated Compose包含十一個one-to-one PyMTLF containers；既有Compose sources／examples保留，exact status／logs／stop與unexpected runtime detection沿用selected inventory |
| 10. subscriber／dataset／traffic | canonical input replaced | Canonical path使用deterministic MNIST或CIFAR-10 Leaf train、Root validation與final held-out data；既有subscriber／UE／PseudoDriver資料鏈保留為legacy assets，但canonical renderer與lifecycle不引用 |
| 11. trigger | canonical trigger replaced | Canonical path使用protocol-driven manual training request與controller event barriers；既有collection／consumer／degradation control保留為legacy／unverified commands，不納入新實驗驗收 |
| 12. reset／restart／evidence | adapted | exact清理十一個node states、ADRF／NRF experiment state、container volumes與records；保留seed重建與failure evidence |

舊definitions、文件、commands、tests與records可以保留作legacy／provenance；只有
`5G_NWDAF_Infrastructure/README.md`的legacy asset inventory負責說明其`legacy`／`unverified`支援狀態，不要求在help、
test名稱或source config重複標記。它們的存在或既有test結果不構成maintained／verified聲明。Legacy profile未通過
共通靜態安全檢查時，lifecycle必須在任何provider或destructive operation前fail closed。

---

## 7. Implementation slices

### 7.1 Slice 1 — Legacy experiment 與 hash cleanup

詳細盤點與 implementation-ready requirements：

- [Slice 1 Legacy Experiment And Hash Cleanup Detailed Plan](./slices/slice-1-legacy-and-hash-cleanup-detailed-plan.md)

#### Operator-visible outcome

Repository將canonical protocol-driven target與既有legacy experiment assets清楚分開，不再宣稱舊production／static
topology、UE traffic或static FL control是本次受支援路徑，也不因canonical path上的trusted local nested digest chain
產生無關mismatch或重建工作。舊source與commands仍可留作參考或best-effort使用，但不保證執行結果。這是cleanup
checkpoint，不交付可執行的新topology；新的正式deployment由Slice 2建立。

#### 工作範圍

- 依Slice 1詳細計畫的L1–L11保留production Flat、static Flat／Hierarchical及其既有runtime assets，在
  `5G_NWDAF_Infrastructure/README.md`建立legacy／unverified inventory；canonical path只解除對舊flow的隱式選取與
  依賴。任何source刪除都必須有逐項consumer trace與具體理由，不得以「新實驗未使用」概括處理。
- 保留既有optional WebConsole subsystem及其operator contract；新canonical deployment固定不啟用，本Slice不為它新增
  adaptation或real-environment acceptance。
- 依 H1–H17 逐項處理 testbed-owned hash：canonical／common path預設移除trusted-local provenance與重複digest，只保留
  有明確external、runtime-identity、Docker-native、component-native或既有build-artifact reuse contract的boundary；
  isolated legacy flow內有consumer且不影響canonical runtime的hash可原樣保留。
- 在Slice 2 topology尚未存在時，缺少明確`TESTBED`／effective `CONFIG_DIR`的deployment入口必須fail closed，不得
  回退至舊default。Operator明確選取legacy profile時仍可經既有common guards作best-effort操作，但本計畫不保證或
  修復其結果。

#### Focused verification

- 逐項關閉詳細計畫中的legacy／hash inventory，確認所有legacy occurrence均標明保留、隔離、具體移除理由或Slice 2
  handoff，且無未分類producer、consumer、field、helper或current docs。
- 以 synthetic fixtures 驗證 common lifecycle safety與transitional fail-closed behavior；本 Slice 不要求 real provider、
  VM、container或experiment evidence。

#### Slice acceptance

- 詳細計畫L1–L10與H1–H17都完成implementation、focused verification與review；既有legacy assets未被無理由刪除，
  L11及legacy profile migration形成Slice 2的明確handoff。
- Common lifecycle safety及transitional fail-closed tests通過；不要求舊 Flat／static experiment regression或real
  environment驗證通過。
- 完成 mandatory initial review、user-review handoff與獨立 commit proposal／approval後，才進入 Slice 2
  implementation。

### 7.2 Slice 2 — Inventory-driven four-VM protocol topology

#### Operator-visible outcome

Operator 透過同一組 `config-create`、validate、VM、services、ML、status、logs、stop、reset 入口，可以建立、啟動與
檢查 `core` + Area A／B／C 四台 VM、十一組 NWDAF↔PyMTLF runtime，以及所需 NRF／ADRF dependencies。Topology
切換、unexpected runtime 或 partial activation 都會 fail closed，不會靜默操作其他部署。

#### 工作範圍

1. 將`Vagrantfile`與canonical／common reachable host／guest config helpers、network generation、service install、
   stage／activate、status／logs、reset及tests中的固定machine／Path loops改為selected manifest inventory驅動；
   dormant legacy-only occurrence依Slice 1 allowlist保留或低成本遷移。
2. 在既有complete `TESTBED` schema內定義第5節的physical/logical topology；它是唯一maintained canonical profile，
   不新增selector。Legacy static profiles依同一schema處理，不建立compatibility schema；既有WebConsole optional
   contract保留，但canonical profile選擇disabled。Make default切至canonical definition，並由其`config.directory`
   提供後續commands的effective `CONFIG_DIR`；不新增current-selection state file。
3. 單一renderer產生：
   - 每個NWDAF獨立native config與systemd unit；
   - 每個PyMTLF獨立config、volume、port與read-only dataset mount；
   - protocol recursive topology、Root／Branch policy、FedProx strategy與`reportAfter`；
   - exact Guest services、Host containers、volumes、ports、reset scope與capacity manifest。
4. 以 `nwdaf-resources` branch-replacement profile的current component contract為語意輸入：一Root、三active regional
   Branches、Area A一inactive replacement、六Leaves；Area A使用同一candidate pool的priority `100` primary與
   priority `50` replacement，Root透過`selection_method: priority`完成初始選擇與故障後重新選擇。Root與Branch皆
   使用configured policy、FedProx `mu`、sample-weighted aggregation，而非testbed自行補hard-coded protocol metadata。
5. 為MNIST與CIFAR-10分別建立deterministic partition：六份Leaf training shards、一份Root per-round validation、
   一份與validation分離的final held-out set。Dataset kind與bounded training parameters由scenario選取；paths只進local
   config／mount，不進Model Training protocol。兩種dataset的shape、class、model／loader compatibility與split語意都必須
   通過component-native validation；相關hash處置依Slice 1 detailed plan，不在本節重新盤點。
6. 加入NTP／chrony與clock-skew preflight，使跨VM JSONL event時間可對齊；clock未同步時不開始real acceptance run。
7. 讓capacity gate由selected inventory計算四台VM、十一個containers、build overhead、storage與GPU／CPU policy；
   配額不足必須在`vm-up`或container start前失敗。
8. Lifecycle涵蓋render、validate、stage、activate、start、status、logs、stop、restart與reset；每一階段都核對selected／
   active identity及actual process/container inventory。
9. Canonical profile完成後，對保留的static Flat／Hierarchical definitions做bounded best-effort migration：能以
   mechanical mapping轉成新schema且通過共通靜態安全檢查者可保留`TESTBED`可選性；否則保存原始意圖與migration gap，
   保持不可啟動。Migration disposition只更新README legacy asset inventory，不加入profile status metadata。不得為此
   恢復legacy-only service、parallel renderer或延後canonical acceptance。

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
- MNIST與CIFAR-10都通過deterministic partition、train／validation／test disjointness、shape／class／count、read-only mount
  與component-native model／loader validation。
- 已遷移的legacy static profiles只執行共通parse、schema、reference、inventory與destructive-safety checks；不新增
  profile-specific regression、real VM或training test，未通過者確認不能進入lifecycle。
- Mock lifecycle涵蓋partial activation、start rollback、unexpected runtime、wrong-config stop/reset、empty targets與capacity
  rejection。
- Approved host context執行real四VM create/start/status，確認OS process inventory與provider state一致。
- Real Guest／Host integration確認NRF registrations、ADRF health、十一個NWDAF與十一個PyMTLF health，以及跨VM／Host
  endpoint reachability。
- Stop／restart／reset後確認exact state、volume、ADRF model records、NRF experiment registrations與seed restoration。

#### Slice acceptance

- 四台VM與十一組runtime可由同一authoritative source完整重建並exact compare。
- Minimal dependency graph在real environment啟動成功；未啟動舊full 5GC／RAN／UPF services。
- MNIST與CIFAR-10各完成一個short normal-topology run，且每個run至少有兩個accepted hierarchical rounds，證明兩種
  dataset的mount、native loader、ADRF、protocol與JSONL wiring；這只是Slice 2 integration acceptance，不是replacement
  evidence或模型品質結果。
- `5G_NWDAF_Infrastructure/README.md`的legacy asset inventory完整記錄static profiles的migration disposition與
  `unverified`狀態：通過共通靜態安全檢查者僅表示可選取，不表示能跑；未通過者仍保留為不可啟動資產。兩者都不要求
  real-run修復，也不影響canonical profile acceptance。
- 所有required lifecycle evidence完成mandatory review並交付user review；缺real provider evidence時狀態只能是
  `Implementation Complete / Verification Incomplete`。

### 7.3 Slice 3 — Dual-dataset Branch replacement flow acceptance

#### Operator-visible outcome

Operator可用同一個canonical experiment command分別選取MNIST或CIFAR-10，執行一個短Branch-failure／replacement run、
觀察精簡milestones、停止Area A primary Branch、以event barrier控制replacement release、收集各node JSONL，並輸出
可review的raw evidence、CSV、簡單plots與run summary。Runner不需要持續把完整journald／Docker logs輸出到terminal。

#### Flow structure

本Slice只執行兩個real flow runs：MNIST一個、CIFAR-10一個。兩個run都使用bounded sample count與較小local epochs來
縮短時間，但不能省略任何normal、degraded或restored phase。它們不是paired baseline，也不重複多個seeds。

每個run的最小accepted-round結構：

1. 至少完成2個normal accepted rounds後才注入failure。
2. Controller確認Area A primary Go NWDAF與PyMTLF都已停止後，寫入`BRANCH_PROCESS_STOPPED`。
3. Replacement backend維持不可用，直到Root至少完成2個只含surviving regions的accepted degraded rounds。
4. Controller依observed `ROOT_ROUND_OUTCOME` barrier放行replacement，不使用固定sleep推測進度。
5. `BRANCH_REPLACEMENT_READY`後，replacement只加入下一個尚未dispatch的cohort。
6. Replacement第一次出現在`successfulNfInstanceIds`後，至少再完成2個accepted restored rounds。

因此每個run至少有6個accepted rounds，並為failure detection、preparation retry與rejected attempts預留timeout；round
number不能只靠wall-clock推定。若實際training時間不允許此最低結構，不得自行降為單round、移除degraded phase或
只證明replacement process啟動；必須回報並由使用者決定sample count、local epochs或timeout的調整。

#### Fault 與 production boundary

- Fault injection只由test controller在runtime外停止exact Area A primary Go NWDAF與PyMTLF targets。
- Controller只能控制primary／replacement processes的availability與放行時序；它不得指定replacement identity、改寫
  priority或繞過Root candidate selection。Root必須從Area A同一candidate pool依priority選出replacement。
- 不在NWDAF／PyMTLF production code新增sleep、kill switch、failure endpoint或實驗專用recovery shortcut。
- Root依production timeout／availability分類與configured completion policy接受degraded round；controller不偽造
  component event。
- Replacement沿用同一`mlCorreId`與same Leaf subtree，但建立新的per-edge resource及`notifCorreId`；不使用
  retained-result recovery。
- Old Branch不在同run內恢復或hand back；同時多Branch失效、Leaf replacement與Root restart recovery不在本flow scope。

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
  scenario、dataset kind、seed、sample count、local epochs、timestamps與exit status。

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
<acceptance>/<dataset>/
├── metadata.json
├── controller-events.jsonl
├── nodes/<nfInstanceId>/observations.jsonl
├── logs/<machine-or-container>/...
├── rounds.csv
├── learning-curve.csv
└── summary.json
```

每個dataset run產生summary與簡單plots，呈現：

- Root validation loss／accuracy對accepted global round；
- normal、degraded與restored phase標記；
- stop→failure detected、stop→replacement ready、stop→first replacement contribution的latency；
- 每個accepted round的successful Branch count／identities；
- final held-out evaluation，僅用來確認evaluation path與輸出完整性。

本計畫只要求證明兩種dataset的系統流程能完成，不比較MNIST與CIFAR-10的模型品質，也不承諾統計顯著性、accuracy
門檻、收斂品質或failure相對於baseline的差異。

#### Slice acceptance

- MNIST與CIFAR-10各有一個完整flow run；每個run都符合2 normal + 2 degraded + 2 restored accepted-round minimum。
- 兩種dataset都由其scenario明確選取並通過對應native model／loader、partition與mount validation。
- 兩個failure runs在replacement pending期間持續產生accepted degraded aggregates，且replacement只於後續cohort恢復。
- 每個accepted `ROOT_ROUND_OUTCOME`都有對應Root global evaluation；rejected attempt不得偽造evaluation point。
- Final held-out evaluation與per-round validation dataset分離。
- 每個run可由metadata、JSONL與summary重建round／event timeline，不需要解析一般文字log。
- Stop、reset與evidence preservation驗證完成；required evidence通過mandatory review與使用者確認後，才能建立
  flow-acceptance record並將本計畫標為completed。這個record不得被描述為controlled comparison或模型效果實驗結果。

---

## 8. Verification matrix

| Requirement | Static／mock evidence | Real-environment evidence | Owner slice |
| --- | --- | --- | --- |
| Hash disposition與consumer trace | inventory review、repository search、focused tests | disposition涉及runtime boundary時執行對應cross-boundary check | 1 |
| Legacy support separation | README legacy asset inventory、canonical reachability與transitional fail-closed tests、deleted-file justification | 不要求legacy real run；Slice 2 canonical startup確認未隱式啟動legacy services | 1、2 |
| Optional WebConsole retention | source／config／targets／existing-test inventory與canonical disabled check | 本計畫不要求WebConsole real run | 1、2 |
| Inventory-driven四機render | valid／invalid YAML fixtures、manifest exact comparison | four-VM inventory matches OS/provider state | 2 |
| 十一組one-to-one runtime | config／port／volume／identity uniqueness tests | health、NRF registrations、reachability | 2 |
| Partial activation／wrong config | mock stage/activate/start/stop/reset tests | controlled failed activation or equivalent approved host test | 2 |
| Provider safety | synthetic guard／process fixtures only | approved host-context preflight before every real provider command | 1、2 |
| Minimal NRF／ADRF dependency | generated service inventory tests | real registration、discovery、store/retrieve/cleanup | 2 |
| Deterministic MNIST／CIFAR-10 partitions | per-dataset shape、class、disjoint partition與seed reproducibility tests | mounted paths與native model／loader acceptance | 2 |
| Short protocol acceptance | runner／parser與per-dataset config tests | MNIST與CIFAR-10各完成至少2個normal accepted rounds | 2 |
| Priority-based replacement與fault barrier | candidate ordering、controller-boundary、JSONL stream／timeout／bounded-log tests | 兩種dataset各完成exact stop、2 degraded、Root選出下一priority、replacement release／ready及2 restored rounds | 3 |
| Dual-dataset flow integrity | metadata dataset coverage與phase-count checker | one complete 2+2+2 replacement run per dataset | 3 |
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
| C2 | Canonical target與既有production／static legacy assets分離；README記錄legacy／unverified disposition，canonical path不隱式選取舊flow，且刪除項目都有具體理由 | Slice 1 README inventory、reachability／deletion review、Slice 2 migration disposition與repository tests |
| C3 | Canonical／common path上的hash occurrence逐項具有consumer trace、defining contract或semantic replacement；isolated legacy內部hash可原樣保留 | Slice 1詳細計畫與focused tests |
| C4 | Canonical／common reachable Machine／Path／service loops全部由selected inventory驅動；legacy-only固定值限於reviewed allowlist | configlib、Vagrantfile、host／guest lifecycle |
| C5 | 四VM、十一NWDAF、十一PyMTLF exact one-to-one inventory | generated manifest與runtime status |
| C6 | 每個logical node有獨立identity、endpoint、state、logs與volume | renderer、systemd、Compose、validators |
| C7 | Minimal core只包含current protocol dependencies | service inventory與real startup evidence |
| C8 | Config identity、partial activation、wrong-config、unexpected runtime與reset fail closed | lifecycle guards與tests |
| C9 | Real provider永不在sandbox內啟動，所有入口經共同guard | command-path review、synthetic tests、host evidence |
| C10 | MNIST與CIFAR-10各自的train／validation／test分離且可由seed重建，並通過native model／loader | dataset preparation與semantic checks |
| C11 | Root／Branch policy、FedProx與`reportAfter`來自authoritative topology並走production protocol | generated native config、component loader、E2E records |
| C12 | Controller只控制process availability；Root從同組candidate pool依priority完成replacement selection | topology、controller implementation與event evidence |
| C13 | MNIST與CIFAR-10各完成一個2 normal + 2 degraded + 2 restored accepted-round flow | scenario validation與run summary checker |
| C14 | 每個dataset只要求一個bounded flow run；不把不同dataset或未配對run解讀為controlled model comparison | metadata coverage與summary wording review |
| C15 | JSONL是round、metric與event evidence；一般log只作diagnostic | collector、schema checker與review |
| C16 | Quiet monitor使用incremental records與bounded diagnostic reads | runner tests與actual terminal transcript summary |
| C17 | Stop／reset只處理selected-and-active exact scope，evidence在cleanup前安全收集 | reset plan、actual runtime comparison與record preservation |
| C18 | WebConsole optional subsystem與既有operator contract保留，但canonical protocol-driven profile不啟用 | Slice 1 source inventory、Slice 2 rendered canonical config與existing tests |

---

## 10. 明確非目標

- 保證、專門維護、修復或執行舊production Flat、static Flat／Hierarchical、UE traffic或degradation experiment；
  既有source／tools保留為legacy／unverified assets並接受bounded migration，但保留不代表可執行。
- 為optional WebConsole新增功能、配合新topology改造或執行real-environment驗收；既有subsystem保留，但canonical
  protocol-driven profile不部署它。
- 執行或修復舊experiment regression suite；既有tests可保留，但只有仍適用的common infrastructure safety tests列入
  本計畫required verification。
- 修改FedProx演算法、Branch aggregation frequency或component protocol schema。
- Retained-result recovery、Leaf replacement、同時多Branch failure、Root restart recovery或old Branch handback。
- 完整5GC user plane、UERANSIM、subscriber、UPF Event Exposure或UE communication analytics。
- `5g-viz`、Grafana、Prometheus、remote write或live dashboard整合。
- FL communication bytes、CPU、GPU、network cost或能源instrumentation。
- Paired no-failure baseline、multi-seed repetition、統計推論、模型品質比較或預先設定accuracy／convergence門檻。
- 長期results retention service、central metrics service或自動論文統計推論。

這些項目若後續需要，應另行設計，不得在本次三個slices中順手加入。

---

## 11. Decision gates 與 blocker handling

只有下列情況停止並請使用者決策：

- current component trace證明minimal topology仍需要本計畫排除的5GC service或新的external dependency；
- 四VM／十一containers無法通過capacity gate，必須減少participants、合併identity、改變isolation或新增host；
- 既有 `TESTBED`／scenario／renderer無法在不產生雙重source的前提下合理擴充；
- multi-host behavior需要修改NWDAF／PyMTLF／NRF／ADRF contract或production recovery semantics；
- PyMTLF current native model／loader無法支援MNIST或CIFAR-10，且必須改變component contract或新增dependency；
- 任一dataset無法完成2 normal + 2 degraded + 2 restored accepted rounds或required evidence；
- 必須弱化provider、wrong-config、exact reset、partial activation或unexpected runtime safety；
- 需要清除不在selected exact scope內的existing VM、container、volume或state。

Blocker回報必須列出原假設、直接反證、受影響slice、可行選項、建議與tradeoff。非阻塞finding分類為future
work、legacy cleanup、optional hardening、integration gap或unconfirmed risk，不擴大current slice。

---

## 12. Review、commit 與進度規則

三個slices依序執行。每次只替即將開始的current Slice建立詳細計畫並完成review；不預先建立後續Slice文件：

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
→ Slice 3 dual-dataset flow acceptance
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
| 1. Legacy／hash cleanup | Completed | implementation、focused／repository synthetic verification、mandatory initial review、使用者review與commit approval已完成 |
| 2. Four-VM topology | Not Started | render／lifecycle tests、approved real four-VM integration、review |
| 3. Dual-dataset replacement flow | Not Started | MNIST與CIFAR-10各一個2+2+2 run、structured evidence、cleanup、review |

本計畫只有在三個slices均完成required evidence、user review，且雙資料集flow-acceptance結果已整理到`records/`後，才能標為
`Completed`。若code完成但real testbed evidence未完成，狀態使用
`Implementation Complete / Verification Incomplete`。
