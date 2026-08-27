# Phase 2 Static Scenario Common Foundation Detailed Plan

日期：2026-08-27

狀態：Implementation redesign required；working-tree review／evidence synchronization completed；existing `TESTBED` contract confirmed

最近更新：2026-08-27

## 1. 目的

在不增加 VM 的前提下，讓 `5G_NWDAF_Infrastructure` 能由不同 complete `TESTBED` definitions 與對應
`CONFIG_DIR` 描述並嚴格驗證 static Flat
與 static Hierarchical 的完整 logical runtime inventory。Phase 2 固定 NF identity、八個 UE、四個 data
owners、process placement、endpoint isolation、lifecycle 與 reset contract，但尚不宣稱兩條 static flow 已完成
collection、training 或 publication。

本 Phase 的實作與審查受 [development policy](../../development_policy.md) 約束；本文件保存 phase-specific decisions、baseline
disposition 與 normative conformance map，不另建平行 workflow。

## 2. 已確認決策

1. 沿用 `core`、`path-a`、`path-b` 三台 VM，不新增 VM。
2. production Flat、static Flat 與 static HFL 使用不同 complete `TESTBED` definitions 與 config directories；
   所有 lifecycle commands 必須使用同一 selected `TESTBED`／`CONFIG_DIR` pair。
3. static Flat 的 Client 1–4 與 static HFL 的 Leaf 1–4 具有一一對應的 logical data-owner positions，但由各自
   complete `TESTBED`／generated config 啟動。
4. static scenarios 使用八個 UE；每個 Client／Leaf 擁有兩個互斥 UE／SUPI，兩條 data paths 各有四個 UE。
5. 每個 Server／Root、Client／Leaf、Branch 都是獨立 NWDAF NF，具有獨立 NF Instance ID、Go process、
   config、endpoint、NRF registration、state 與 PyMTLF process；只共用 Go binary 與 component revision。
6. scenario switching 維持 explicit clean-start。使用者負責 stop 與 guarded reset，啟動流程不自動清除或
   覆蓋 active scenario。
7. 沿用既有 `TESTBED` contract：一份 testbed YAML 代表一套完整 topology，不新增 `deployments/`、
   `DEPLOYMENT` selector、內嵌 profiles 或 partial overlay。
8. 保留現有 `testbed.yaml` 作為 production Flat reference；另以相同 schema 建立
   `testbed.static-flat.yaml` 與 `testbed.static-hierarchical.yaml` 兩份完整 definitions。
9. Operator 只需要掌握 selected `TESTBED`、`scenario.yaml`，以及 renderer 產生的完整 `CONFIG_DIR`。
   同一份 topology、identity、placement 或 ownership 資訊不得同時維護在 testbed YAML 與另一個
   高階 deployment YAML。

## 2.1 Configuration source ownership

| Source | Ownership |
| --- | --- |
| selected `TESTBED` | 一份完整 VM／network／NF topology definition；由 `testbed.yaml`、`testbed.static-flat.yaml` 或 `testbed.static-hierarchical.yaml` 三選一 |
| `scenario.yaml` | traffic、sampling、monitoring、training rounds／epochs 與 experiment timing |
| generated `CONFIG_DIR` | processes 實際消費的完整 native configs 與 manifest，不作為另一份手動高階來源 |

NF identity、VM placement、IP、port、Branch／Leaf edges 與 UE ownership 屬於 testbed deployment，不放入
`scenario.yaml`，也不為了 renderer 方便新增另一種需要使用者理解與同步的 deployment schema。三份
testbed definitions 都是既有 schema 的完整文件，不使用繼承或局部覆寫；跨檔共同欄位由 validation 與
tests 防止非預期 drift。

## 3. Target runtime inventory

| Placement | Static Flat | Static HFL | Data ownership |
| --- | --- | --- | --- |
| `core` VM | 1 Server NWDAF | 1 Root NWDAF＋2 Branch NWDAFs | 無 UE training partition |
| `path-a` VM | Client 1、2 NWDAFs | Leaf 1、2 NWDAFs | 四個 UE；每個 data owner 兩個 SUPI |
| `path-b` VM | Client 3、4 NWDAFs | Leaf 3、4 NWDAFs | 四個 UE；每個 data owner 兩個 SUPI |
| host ML runtime | 5 PyMTLF containers | 7 PyMTLF containers | 每個 NWDAF 對應獨立 PyMTLF state |

Branch 同時啟用 downstream aggregation server 與 upstream client engines，但不配置 private collection、UE
partition 或 autonomous Root orchestration。Client／Leaf 是唯一 static training-data owners。

## 4. 已發現的 implementation gaps

目前 production Flat lifecycle 仍以 A／B／C 固定清單為中心，包括：

- Guest service start／stop／status 與 `service-run.sh` 只認得既有 `nwdaf-a`、`nwdaf-b`、`nwdaf-c`；
- Compose 與 ML start／status／reset 只描述 `pymtlf-a`、`pymtlf-b`、`pymtlf-c` 及兩個 PyAnLF containers；
- renderer、config checker、dataset tooling 與 observability assertions 直接引用 A／B／C names；
- reset 的 Guest unit、ML service 與 volume scopes 是 hard-coded，無法安全涵蓋額外 Branch／Leaf instances；
- 現有六個 UE 與兩條 path 的 service inventory 必須擴充為八個 UE，並建立四份互斥 collection profiles；
- 同一 VM 上啟動多個 Go NWDAF processes 需要明確的 config selection、IP／port binding、NF identity、systemd
  instance dispatch、log 與 runtime-directory isolation。

Phase 2 不以複製 binary、source checkout 或整台 VM 作為新增 NF 的方式；應以同一 pinned binary 啟動多個
隔離的 systemd process instances。

## 4.1 目前 working tree 能力與處置

目前 `5G_NWDAF_Infrastructure` working tree 已證明下列 building blocks 可行，但現有組合方式未通過
architecture／lifecycle review，不得直接整理後 commit：

- 5／7 個 NWDAF/PyMTLF、8 UE、4×2 ownership groups 的 static config generation；
- manifest-driven Guest service、Host ML、volume、subscription、status、logs 與 reset inventories；
- generic systemd instance dispatch，同一 Go NWDAF binary 可依 role-named config 啟動獨立 process；
- static Compose definitions、CPU/GPU policy、published ports、per-instance volumes 與 containing-NWDAF endpoints；
- production Flat 仍維持 3 NWDAF、6 UE、5 ML containers 與 Consumer subscription chain；
- strict static checker、PyMTLF native settings/topology loader、deterministic seed bundle identity 與 topology tamper rejection；
- 8-UE PseudoDriver dataset generation；Flat/HFL 使用相同 dataset identity 以固定 stimulus。

其中可保留的是 dynamic inventory、generic systemd dispatch、八 UE fixture 與 topology／native validation
所證明的 component capability；雙 renderer、雙 checker、額外 deployment schema 與混合 Compose catalog
不作為後續實作基礎。

目前 uncommitted implementation 新增了獨立 `deployments/` 目錄。這不是原計畫要求，且增加 operator 需要
理解與同步的設定來源，因此在 review／commit 前必須撤換：

1. 保持現有 `testbed.yaml` 的 production Flat definition 與預設行為。
2. 將 static Flat／HFL contracts 改寫為完整的 `testbed.static-flat.yaml` 與
   `testbed.static-hierarchical.yaml`，沿用既有 testbed schema。
3. 移除 `DEPLOYMENT` option；Renderer、strict checker、dataset tooling 與 tests 只讀 selected
   `TESTBED`，不再讀取 `deployments/*.yaml`。
4. 移除 uncommitted `deployments/` 目錄。
5. 重新 render／validate 三份完整 testbed definitions，確認既定 runtime inventory，並檢查共用 VM／network／
   mobile-network 欄位沒有非預期 drift。

尚未執行的是三台 VM 的 static stack activation、實際 stop/reset/apply/verify，以及 reset 後 coordinator
runtime seed re-import。這些 evidence 完成前，本文件不把 Phase 2 標成 Completed，也不宣稱 Phase 3／4 full flow。

## 4.2 Working-tree review findings

2026-08-27 review 以 Phase 1 baseline `072748b` 對照目前 uncommitted Phase 2 diff。多數高風險問題是本次
實作直接引入；少數是原本只服務單一 production topology 的假設，在擴充後未同步重構而成為 blocker。

### 本次實作直接引入

1. `static-config-render.py` 先呼叫 production renderer 產生 A／B／C，再刪除 NWDAF／ML／Consumer files 後
   重新生成 static topology；這造成雙 renderer、重複 I/O 與難以驗證的中間狀態。
2. `static_config_check.py` 形成第二條 validation path；static config 會提前 return，略過既有 Core、RAN、
   network、endpoint、scenario 與 native common checks。
3. `DEPLOYMENT`、`deployments/*.yaml` 與 testbed definition 同時描述 topology，違反既有 `TESTBED` ownership。
4. `compose.static.yaml` 同時保存 Flat 與 HFL dormant service catalog，且 production lifecycle 也固定載入；
   同一 PyMTLF image target 又按每個 selected service 重複 build。
5. lifecycle 依呼叫時 selected manifest 停止／顯示 processes，卻未驗證它和 Guest active config、container
   labels 相同；使用錯誤 config 可能漏停或隱藏另一 topology 的 active processes，最後仍回報成功。
6. dynamic inventory 透過 shell process substitution 讀取；子程序失敗可能退化成空清單。對 ML lifecycle
   而言，空 service arguments 反而可能讓 Compose 操作完整 catalog；Guest start／stop 也可能零項成功。
7. runtime manifest 目前只驗證欄位形狀，沒有驗證完整 Core／UPF／gNB inventory、container-to-volume 一一
   對應或 exact reset scope；漏列 state 可能保留 stale data，多列 project-owned volume 可能擴大清除範圍。
8. static dataset path 只取第一個 matching Client／Leaf，且使用 hard-coded `min_matched=2`；同一 path 上
   第二個 data owner 的 collection contract 未完整參與診斷。
9. static tests 只涵蓋 render、checker、Compose resolution、8 UE／4×2 ownership 與 topology tamper；未測
   manifest failure、wrong-config lifecycle、unexpected active process、switching、reset exactness 或 capacity。

### 舊假設在本次擴充後成為 blocker

1. production renderer 固定 A／B／C、六 UE；原本只有一種 topology 時是既有 contract，現在應改為由
   selected complete `TESTBED` 直接生成，而不是另加 static workaround。
2. Guest reset safety check 固定 `nwdaf-c`；Host 正常入口目前另有 active-unit guard，但 Guest destructive
   guard 本身沒有涵蓋 Server／Root／Branch／Client／Leaf。
3. capacity gate 只用固定 Host reserve。現有 HFL Compose limits 合計 8192 MiB，已高於 `testbed.yaml`
   的 6144 MiB reserve，且尚未納入 Docker／build overhead 與 GPU memory。
4. project-wide ML rollback、status 與 config activation 原本只面對一種 topology；多 topology 後必須加入
   active identity、unexpected inventory 與 partial activation protection。

Host/native static checks 目前可通過，但不足以否定上述 lifecycle 問題。初次 review 的 `make test` 曾因
執行環境無法寫入 Vagrant home 而停止；後續已在可寫環境完整重跑並通過，包含 static Flat／HFL render、
native／ownership／tamper checks、Compose checks、dataset determinism、production config regressions 與
`vagrant validate`，`git diff --check` 亦通過。這些結果只關閉 repository-local verification，不是 Phase 2
runtime acceptance，也不能取代 missing failure-path tests 或三 VM lifecycle evidence。

### 4.2.1 Follow-up direct runtime／failure-path evidence

2026-08-27 follow-up review 另以 read-only checks 取得下列直接證據；沒有啟動、停止、reset 或修改任何 runtime：

- VirtualBox 已註冊 `core`、`path-a`、`path-b` 三台 VM，當時全部為 poweroff；實際宣告資源分別為
  `4096 MiB／4 CPU`、`3072 MiB／3 CPU`、`3072 MiB／3 CPU`，和目前 testbed definition 相符。因 VM 未啟動，
  Guest active config、process inventory、IP aliases 與 runtime capacity 尚未驗證。
- Host 當時有 24 logical CPUs、約 20.7 GiB available memory、約 212 GiB available disk；RTX 3080 10 GiB
  約有 9988 MiB free，Docker 與 NVIDIA runtime 均存在，目標 ML published ports 未發現占用衝突。Prototype
  Compose limits 加總為 static Flat `7 CPU／6144 MiB`、static HFL `9 CPU／8192 MiB`；這仍未包含 image build／
  Docker overhead，也沒有證明各 GPU participant 的必要 VRAM 或 simultaneous runtime capacity。
- 五個 production ML containers 當時均為 stopped，project volumes 保留。Container config labels 指向舊
  config hash prefix `0c685`，目前 selected `config/default` tree hash prefix 為 `53de4`；實際執行
  `ml-status.sh testbed.yaml config/default` 仍以 exit status 0 顯示舊 hash 與 retained `FL RESULT`，沒有拒絕
  selected／actual identity mismatch。
- 對 inventory reader 注入解析失敗時，subprocess 會輸出 traceback，但呼叫端仍以 status 0 得到
  `record_count=0`；把空 service target 傳給目前 mixed Compose catalog 時會解析成全部 17 個 services。
  這是 fail-open blocker 的 deterministic characterization，不是只依 code inspection 推測。

## 4.3 Production baseline stage disposition

Phase 2 的 canonical existing flow 是現有 production Flat：由 complete `TESTBED` 與 `scenario.yaml` 經既有
`config-create` 產生 native `CONFIG_DIR`，再走同一組 validation、VM／Guest、Host ML、dataset、subscription、
lifecycle 與 reset 入口。各階段處置如下：

| Baseline stage | Disposition | Phase 2 contract |
| --- | --- | --- |
| `TESTBED` selection | Adapted | 維持一份 selected complete definition；增加兩份同 schema 的 static definitions，不增加 selector layer |
| `scenario.yaml`／traffic input | Reused without semantic change | 繼續擁有 traffic、sampling、monitoring、rounds／epochs 與 timing，不接管 topology |
| `config-create`／renderer | Adapted | 擴充既有單一 pipeline，使其直接依 selected `TESTBED` 生成 production／Flat／HFL；移除雙 renderer 與 generate-then-delete |
| strict checker／native loaders | Adapted | 維持單一 common-first checker，再執行 topology-specific checks；所有 native loaders 仍必須通過 |
| generated `CONFIG_DIR`／manifest | Adapted | 依 selected `TESTBED` 生成 exact runtime inventory、native configs 與 selected Compose artifact，且可反向 exact compare |
| Vagrant definition／VM lifecycle | Reused without semantic change | 繼續使用 `core`、`path-a`、`path-b` 三台 VM，不增加 VM |
| Guest stage／activate／network reconciliation | Adapted | 延伸至多 NF configs，加入 selected／active identity 與跨 Guest partial-activation detection／recovery |
| Guest start／status／logs／stop | Adapted | 從 A／B／C 固定清單改為 fail-closed manifest inventory，並呈現 unexpected processes |
| Host ML build／lifecycle | Adapted | 從固定 Compose catalog 改為 selected topology artifact；支援五／七個 PyMTLF instances，避免重複 build |
| subscriber／dataset／traffic stimulus | Adapted | static scenarios 改為八 UE、四個互斥 data owners；production 六 UE baseline 不變 |
| production Consumer subscription chain | Reused without semantic change | production Flat 繼續走既有 Model Provision／Monitor／Consumer chain |
| static training trigger | Not applicable to production chain | static Flat／HFL 不依賴 Consumer subscription chain；Phase 2 只建立明確 static contract，完整 training flow 留在 Phase 3／4 |
| stop／reset／seed recovery | Adapted | 依 declared／actual exact scope guarded reset，保留 canonical seed source 並驗證 deterministic re-import |

任何 stage 若實作時無法依上表處理，而需要新的人工 config source、top-level directory、selector、parallel
renderer／checker 或不同 lifecycle entrypoint，視為 architecture decision，必須先更新本計畫並取得使用者確認。

## 5. State 與 seed model contract

現有 lifecycle 的保留／清理語意應延伸至所有 static instances：

- `experiment-stop` 保留 datasets、databases、ADRF state、ML artifacts、model state、containers、images 與
  volumes；
- `experiment-start` 要求 clean process state；任何既有 experiment Guest unit、ML container 或 consumer active
  時拒絕啟動；
- reset 必須由 selected manifest 列出 exact Guest units、host containers、volumes、ADRF collections 與 model
  storage，先提供 plan，再要求 scenario name confirmation 才執行；
- reset 清除 runtime seed copy、trained models、FL workspaces、publication journals 與 ADRF experiment state，
  但保留 container image 內 canonical seed source；
- reset 後第一次 coordinator 啟動重新匯入 deterministic seed，且 artifact key 必須與 config lock 相符；
- 不新增自動 scenario replacement，也不因選擇另一個 `CONFIG_DIR` 隱式 reset。

## 6. Implementation slices

### Slice 1. Capacity、identity 與 endpoint inventory

- 記錄三台 VM 的 CPU、memory、active IP aliases、可用 ports 與 systemd template contract；
- 記錄 host CPU／memory／GPU capacity、Docker network 與 published-port boundary；
- 由 selected runtime inventory 加總 Compose CPU／memory limits、build overhead 與 GPU participants，不以固定
  Host reserve 代替實際需求；
- 配置五個 base data/coordinator positions 與兩個 HFL-only Branch positions；
- 驗證每個 NF 的 NRF profile、SBI、internal API、callback 與 PyMTLF endpoint 可唯一表示且互相可達；
- 若既有資源不足，停止並回報，不新增 VM 或合併 NF identity。

### Slice 2. Complete testbed definitions、manifest 與 config layout

- 保留 `testbed.yaml`，並以相同 schema 新增 static Flat 與 static HFL 完整 testbed definitions；
- 只保留一個 render pipeline，由 selected `TESTBED` 直接產生對應 config directory，不新增 `DEPLOYMENT`
  selector、平行 deployment schema 或先生成 A／B／C 再刪除的流程；
- 讓 manifest 明確列出 Guest machines、NWDAF NFs、ML processes、UEs、data-owner mapping 與 volumes；
- production Flat default config 保持三 NWDAF／六 UE 原狀；
- 單一 strict checker 先執行 common Core／RAN／network／scenario／native checks，再依 selected topology 執行
  Flat／HFL-specific checks；不使用提前 return 的第二套 checker；
- manifest inventory 必須能由 selected `TESTBED` 完整重建並 exact compare，不只驗證欄位形狀。

### Slice 3. Independent NWDAF NF lifecycle

- 讓同一個 Go binary 能依 systemd instance 選取正確 config；
- 每個 NF 使用獨立 NF Instance ID、bind endpoint、runtime directory 與 log identity；
- start／status／stop 依 manifest inventory 操作，不遺留未宣告或 stale NF process；
- inventory 讀取必須 fail closed；空清單、解析失敗或 declared／actual inventory 不符時不得繼續；
- stop／status／logs 必須比較 selected config hash、Guest active hash 與 container labels，並顯示 unexpected
  active units／containers；wrong-config operation 應拒絕而不是部分執行；
- HFL Branch capability 與 paired PyMTLF server＋client engines 必須一致。

### Slice 4. Dynamic host ML inventory

- 由 selected `TESTBED` 產生只包含該次 topology 的 Compose runtime artifact；不讓 production、Flat、HFL
  共用一份含 dormant services 的 catalog；
- 讓 Compose start、status、stop 與 health checks 支援五個或七個 PyMTLF services；同一 image target 只建置／
  驗證必要次數，不按 instance 重複 build；
- 每個 PyMTLF 使用自己的 config、published endpoint、volume 與 containing-NWDAF internal API root；
- 不為 pure Branch 配置 PyAnLF 或 training-data collection ownership；
- resource limits 與 device policy 必須在現有 host capacity 內通過 validation。

### Slice 5. Eight-UE data-owner mapping

- 將 static scenario 的兩條 paths 各配置四個 UE，共八個唯一 SUPIs；
- 四份 Client／Leaf collection profiles 各包含兩個 SUPIs，彼此無重疊且全集恰為八個；
- Flat Client N 與 HFL Leaf N 使用相同 logical data partition；
- subscriber data、SMF session resolution、UPF stimulus dataset 與 service inventory 保持一致；
- dataset validation 必須涵蓋每個 data owner，不得只取每條 path 的第一個 matching Client／Leaf，也不得以
  hard-coded evidence threshold 取代 selected scenario／native contract。

### Slice 6. Manual reset and seed restoration

- reset plan／apply／verify 全部改用 selected manifest inventory；
- reset 拒絕任何 declared 或 unexpected experiment process 仍 active 的狀態；
- Host 與 Guest reset guards 都依 actual／declared inventory 檢查，不保留 `nwdaf-c`-only safety assumption；
- 驗證新增 NF／ML volumes 與 workspaces 均被納入 exact scope，且 selected manifest 不可加入其他
  project-owned topology 的 volume；
- 驗證 runtime state 清空後，coordinator startup 能重新產生相同 seed artifact key；
- containers、images、volume objects、VMs 與 canonical seed source 必須保留。

### Slice 7. Static validation gate

- render／validate production Flat、static Flat 與 static HFL 三組 configs；
- 使用 pinned NWDAF／PyMTLF native config validation；
- 驗證 NF identities、ports、callbacks、topology edges、UE ownership、volumes 與 cleanup scopes 無衝突；
- 加入 empty／invalid manifest、wrong selected config、unexpected Guest／ML process、partial stop、exact reset scope
  與 resource-capacity regression tests；
- Phase 2 只宣稱 common foundation ready，不以 host-only validation 取代 Phase 3／4 full flow evidence。

## 7. Verification matrix

| Layer | Verification | Gate |
| --- | --- | --- |
| Capacity | three-VM 與 selected Host runtime inventory | 五／七個 logical NFs、ML limits、build overhead 與 GPU participants 可配置；不足時明確阻擋 |
| Config | 三份 complete `TESTBED` definitions、單一 renderer／checker、native loaders | exact mode／role／topology／collection ownership 全部通過，無 generate-then-delete path |
| NF identity | NF Instance ID、SBI／internal endpoints、NRF profiles | 所有同時執行的 NFs 唯一且可達 |
| Data owners | 八個 SUPIs 與四份 profiles | 每個 owner 恰兩個、無交集、無遺漏 |
| Lifecycle | manifest-driven start／status／stop inventory | fail closed；wrong config／unexpected process 被拒絕，不使用 A／B／C hard-coded topology assumptions |
| Reset | exact declared／actual scope、plan／apply／verify、seed re-import check | 不跨 topology 清除、不遺漏 state，canonical seed 可重建為相同 identity |
| Repository | focused tests、`make test`、Compose checks、`git diff --check` | 全部通過 |

## 7.1 Normative conformance map

下表是 2026-08-27 依新版 testbed development policy 對目前 uncommitted working tree 的重新審查結果。
`部分符合` 只代表存在可保留的 building block 或 indirect evidence，不代表 criterion 已關閉。

| Normative item | Current status | Direct evidence | Open work／required evidence |
| --- | --- | --- | --- |
| 三種 topology 使用 complete `TESTBED` definitions | 未符合 | 現有 prototype 能表達三種 inventory | 移除 `DEPLOYMENT`／`deployments/`，新增兩份同 schema complete definitions，驗證共用欄位 drift |
| 單一 authoritative render pipeline | 未符合 | production renderer 與 static renderer 均存在 | 合併為 selected `TESTBED` 驅動的單一 renderer，移除 generate-then-delete |
| 單一 common-first strict checker | 未符合 | production 與 static checks 各自可執行 | 移除提前 return／第二 checker；加入 common 與 topology-specific regression evidence |
| production Flat 3 NWDAF／6 UE behavior 保持不變 | 部分符合 | Host render checks 顯示既有 inventory 仍可生成 | 以同一新 pipeline 執行 production regression，並補實際 lifecycle evidence |
| static Flat 1 Server／4 Clients／8 UE／5 PyMTLF | 部分符合 | prototype render、native checks 與 ownership tests 通過 | 改用 approved source／pipeline 後重跑；補三 VM activation 與 lifecycle evidence |
| static HFL 1 Root／2 Branches／4 Leaves／8 UE／7 PyMTLF | 部分符合 | prototype render、native topology checks 與 tamper rejection 通過 | 改用 approved source／pipeline 後重跑；補三 VM activation 與 lifecycle evidence |
| 每個 NWDAF 為獨立 NF identity／process／state | 部分符合 | generic systemd dispatch 與 per-instance config／volume building blocks 已存在 | 驗證 simultaneous bindings、NRF profiles、logs、runtime directories 與 restart isolation |
| Manifest 可由 trusted `TESTBED` exact rebuild／compare | 未符合 | 目前只驗證部分欄位形狀 | 納入完整 Guest、NF、ML、UE、volume、subscription 與 reset inventory；加入 omission／extra-item tests |
| selected／active runtime identity 一致 | 未符合 | 五個 stopped production ML containers 的 label hash prefix `0c685` 和 selected config hash prefix `53de4` 不同；`ml-status.sh` 仍以 status 0 顯示 stale hash／retained result | 加入 Guest active hash、container labels、unexpected inventory 與 wrong-config rejection；補 active-runtime regression tests |
| Empty／invalid inventory 與 partial operation fail closed | 未符合 | 注入 inventory parser failure 後呼叫端仍 status 0／`record_count=0`；空 Compose target 解析成 mixed catalog 全部 17 services | 將 reader 與 lifecycle 改為 fail closed；加入 rollback、partial activation／stop detection 與 recovery tests |
| Host ML selected artifact 與五／七 instance lifecycle | 部分符合 | static Compose catalog 可解析，per-instance endpoints／volumes 已存在 | 改為只生成 selected topology artifact；補 build deduplication、health、status、logs、stop evidence |
| 八 UE／四個互斥 2-SUPI data owners | 部分符合 | 8-UE fixture 與 4×2 ownership checks 通過 | 修正 per-path first-match／hard-coded threshold，驗證每個 owner 的 subscriber、session、dataset contract |
| 三 VM／Host／Docker／GPU capacity gate | 未符合 | 三台 poweroff VM 的 declared resources 符合 definition；Host 有 24 logical CPUs、約 20.7 GiB available memory、約 212 GiB disk、RTX 3080 約 9988 MiB free，且無目標 ML port conflict；Flat／HFL limits 為 `7 CPU／6144 MiB`、`9 CPU／8192 MiB` | 由 selected inventory 納入 build／Docker overhead、GPU participant VRAM 與 Guest runtime；補 simultaneous activation capacity evidence |
| Exact guarded reset 與 deterministic seed restoration | 未符合 | Prototype 已能產生部分 volume／reset inventory 與 seed identity | 修正 Guest `nwdaf-c`-only guard，驗證 declared／actual exact scope、unexpected state 與 reset 後 re-import |
| Required failure-path regression tests | 未符合 | 現有 static render、native、ownership、tamper tests 通過 | 新增 manifest、wrong-config、unexpected process、switching、capacity、partial activation／stop、reset exactness tests |
| Repository full checks | Local verification passed | 完整 `make test`（含 Compose／production regressions／`vagrant validate`）與 `git diff --check` 通過 | Redesign implementation 完成後必須重跑；目前結果不涵蓋缺少的 failure-path tests 或 real lifecycle acceptance |
| 實際三 VM lifecycle acceptance | 未驗證 | 三台 VM 已註冊但 review 時全部 poweroff；Guest active config／process state 無法觀察 | 完成 stage／activate／start／status／logs／stop／reset／seed recovery evidence |
| User review 與獨立 commit | Pending | 本次已完成 working-tree architecture／lifecycle review | 重作 implementation、mandatory initial review 與 verification 後另行提出 user-review handoff 與 commit proposal |

依此 conformance map，Phase 2 仍為 `Implementation redesign required`。目前 code prototype 只能作為 capability
evidence 與 remediation input，不能標記為 implementation-ready、verification-complete 或 commit-ready。

## 8. Phase completion criteria

只有同時滿足下列條件才完成 Phase 2：

1. production Flat default config 與現有 runtime contract 未被改成 static scenario。
2. Operator 不需要讀取或同步獨立 `deployments/` 目錄；三種 topology 均以既有 `TESTBED` mechanism 選擇一份完整 definition。
3. static Flat 可 render／validate 一 Server、四 Clients、八 UE 與五組獨立 NF／PyMTLF identities。
4. static HFL 可 render／validate 一 Root、兩 Branches、四 Leaves、八 UE 與七組獨立 NF／PyMTLF identities。
5. 三台 VM／host capacity 與所有 endpoints 通過 inventory gate，未新增 VM。
6. 只有一個 renderer 與一個 common-first checker；不存在 generate-then-delete、第二套 config source 或
   static validation shortcut。
7. lifecycle 與 reset scopes 由 selected `TESTBED`／manifest 產生並和 actual runtime identity 比對；invalid／
   empty inventory、wrong config 與 unexpected process 全部 fail closed。
8. reset 不遺漏或跨 topology 清除 state，且 canonical seed restoration identity 驗證通過。
9. focused failure-path tests、repository checks 與實際三 VM lifecycle evidence 通過。
10. Diff 經 user review、verification 通過，並使用獨立 commit。

## 9. 明確不包含

- static Flat full collection／FedAvg／publication runtime acceptance；
- static HFL Leaf fitting／Branch aggregation／Root aggregation runtime acceptance；
- Flat FedProx、controlled performance comparison 或 Branch latency attribution；
- 自動停止、清理或覆蓋 active scenario；
- 新增、重建或刪除 VM；
- production Flat 從六個 UE 改為八個 UE；
- dynamic hierarchy、hot reload 或 arbitrary-depth topology。
