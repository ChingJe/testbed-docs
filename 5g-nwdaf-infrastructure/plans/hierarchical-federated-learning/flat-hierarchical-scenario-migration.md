# Flat and Hierarchical Testbed Scenario Migration Plan

日期：2026-08-26

最近更新：2026-08-27（Phase 2 working-tree review／evidence synchronization）

狀態：Phase 1 Completed (`072748b`)；Phase 2 implementation redesign required；Phase 3–4 Pending

## 1. 目標

將新版 explicit orchestration 與 private collection 能力整合進 `5G_NWDAF_Infrastructure`，先讓既有
production Flat 流程在新版 config schema 下保持原行為，再建立可由 operator 明確控制的 static Flat
與 static Hierarchical testbed scenarios。

第一階段目標是跑通 collection、training、aggregation、publication 與 cleanup 流程，不以現階段結果
比較 Flat 與 HFL 的效能，也不將 FedAvg／FedProx 差異解讀為純 topology effect。

## 2. 已確認的三條配置線

### 2.1 Existing production Flat

既有 default deployment 維持目前 three-NWDAF 行為與角色：

- NWDAF-A／B：FL Clients；
- NWDAF-C：FL Server、Model Provision provider 與 Model Monitor coordinator；
- participant selection：`monitor_scopes`；
- training trigger：accuracy degradation；
- training-data collection：`consumer_subscription`；
- 保留現有 Model Provision／Model Monitor、TAI scope、NRF discovery、IP、port 與資料來源語意。

這條配置只做新版 schema 所需的最小遷移：

- legacy runtime mode 改為 federated runtime；
- C 明確宣告 `flat + monitor_scopes` orchestration；
- C 明確啟用 degradation trigger、停用 private training trigger；
- A／B 明確宣告 `collection_trigger: consumer_subscription`；
- 將 Server-owned training request parameters 從舊 Client-local 欄位遷到正確 owner；
- 不增加 Clients、不改成 static topology，也不移除既有自動訂閱鏈。

### 2.2 New static Flat

邏輯 topology 固定為一個 Server 與四個 Clients：

```text
Server
├── Client 1
├── Client 2
├── Client 3
└── Client 4
```

配置規則：

- orchestration：`flat + static`；
- exact topology：四個明確 Client identities；
- training trigger：operator private API；
- training-data collection：各 data-owning Client 使用自己的 private collection profile；
- 不依賴 analytics consumer、Model Provision／Model Monitor subscription chain 來選擇 participants；
- 使用八個 UE，四個 Clients 各自擁有兩個互斥的 UE／SUPI；
- 第一階段沿用既有 Flat FedAvg，僅驗證完整流程與 evidence。

### 2.3 New static Hierarchical

邏輯 topology 固定為一個 Root、兩個 Branches 與四個 Leaves；每個 Branch 連接兩個 Leaves：

```text
Root
├── Branch 1
│   ├── Leaf 1
│   └── Leaf 2
└── Branch 2
    ├── Leaf 3
    └── Leaf 4
```

配置規則：

- orchestration：`hierarchical + static`；
- Root 是唯一 autonomous top-level coordinator；
- Branch 同時需要 aggregation server 與 upstream client engines，但不是另一個 Root；
- Branch 不擁有額外 training data，也不建立 private collection；
- 四個 Leaves 是 data owners，各自使用 private collection profile，且各自擁有兩個互斥的 UE／SUPI；
- 第一階段沿用既有 HFL FedProx，僅驗證完整流程與 evidence。

Static Flat 與 static HFL 必須由不同 config directories 啟動。Client 1–4 與 Leaf 1–4 只共享 logical
data-owner position 與對應資料分區語意，不代表共用同一個同時執行中的 process 或 config。

沿用既有 `TESTBED` 選擇完整 topology document 的設計：保留 `testbed.yaml` 作為 production Flat reference，
另以相同 schema 建立 `testbed.static-flat.yaml` 與 `testbed.static-hierarchical.yaml`。不新增
`deployments/`、`DEPLOYMENT` selector、內嵌 profiles 或 partial overlay，也不把 deployment topology、
NF identity、placement 或 UE ownership 混入 `scenario.yaml`。Operator 使用 selected `TESTBED` 與
`scenario.yaml` 產生完整 native files 到 selected `CONFIG_DIR`，後續 lifecycle 維持同一組選擇。

## 3. 跨場景固定關係

Static Flat 與 static Hierarchical 使用相同四個 data-owner positions：

| Data partition | Flat role | HFL role | HFL parent | UE／SUPI ownership |
| --- | --- | --- | --- | --- |
| 1 | Client 1 | Leaf 1 | Branch 1 | 兩個互斥 UE／SUPI |
| 2 | Client 2 | Leaf 2 | Branch 1 | 兩個互斥 UE／SUPI |
| 3 | Client 3 | Leaf 3 | Branch 2 | 兩個互斥 UE／SUPI |
| 4 | Client 4 | Leaf 4 | Branch 2 | 兩個互斥 UE／SUPI |

後續實驗應盡量固定：

- initial model 與 model interoperability；
- 四份 data partition、traffic stimulus 與 observation window；
- data-owner collection area 與 SUPI ownership；
- random seed、batch size、learning rate、local fitting effort 與 round count；
- validation、publication、cleanup 與 evidence collection 方式。

Flat 的 topology／training scope 可使用 exact TAI；HFL topology／assignment／training 不加入 TAI。HFL Leaf
仍可在 local collection profile 設定 network area，該 TAI 只供 SMF ingress gating，不參與 hierarchy
participant selection。

## 4. Algorithm 邊界與延後工作

第一階段保留目前已驗證的演算法組合：

- static Flat：FedAvg；
- static Hierarchical：FedProx。

這兩條結果只證明各自流程可執行，不能宣稱 Branch presence、topology 或 aggregation algorithm 的單一
因果效果。目前 roadmap 不排入 Flat FedProx migration 或 controlled comparison；只有在 production Flat、
static Flat 與 static HFL 流程都通過 testbed validation 後，才另立 future plan 決定是否以相同四個 data
owners 與受控參數建立 Flat FedProx vs. Hierarchical FedProx comparison。

## 5. Logical topology、NF identity 與 physical placement

每個 topology node 都是獨立 NWDAF NF，而不是單獨的 PyMTLF worker。每個 NF 必須具有自己的 Go NWDAF
process、NF Instance ID、config、SBI／internal endpoint、NRF registration、runtime state、log 與對應的獨立
PyMTLF process。各 process 可共用同一份 Go NWDAF binary 與相同 component source revision，但不得共用
runtime identity 或可互相覆寫的 state directory。

因此邏輯 runtime 數量為：

- static Flat：五個獨立 Go NWDAF NFs，加上五個對應 PyMTLF processes；
- static Hierarchical：七個獨立 Go NWDAF NFs，加上七個對應 PyMTLF processes。

第一輪 placement 不增加 VM，沿用 `core`、`path-a`、`path-b` 三台 VM：

- Server／Root 放在 `core`；
- Branch 1／2 放在 `core`，但仍是和 Root 分離註冊、分離設定與分離執行的 NFs；
- Client／Leaf 1–2 放在 `path-a`，Client／Leaf 3–4 放在 `path-b`；
- PyMTLF processes 維持 host containers，各自使用獨立 endpoint 與 volume；
- `path-a`、`path-b` 各配置四個 UE，共八個 UE，每個 Client／Leaf 擁有兩個互斥 SUPI。

Phase 2 仍須盤點既有 three-VM CPU／memory、host GPU／memory、IP aliases、ports、NRF profile、callback
reachability 與 lifecycle tooling。若容量不足，應回報 blocker，不得在未重新決策下新增 VM、合併兩個
logical NFs 或讓它們共用 NF identity。

### 5.1 Scenario switching 與 state policy

- operator 必須以同一組 selected `TESTBED`／`CONFIG_DIR` 完成該次 scenario 的 render、validate、start、
  status、stop 與 reset；
- `experiment-start` 維持 clean-start gate；發現任何既有 experiment process／container active 時拒絕啟動，
  不自動停止或覆蓋另一個 scenario；
- scenario state 由使用者明確執行 stop／reset，不在 config 切換時自動清除；
- `experiment-stop` 只停止 processes，保留 databases、artifacts、model state、containers 與 volumes；
- guarded `reset` 清除 runtime model state、training artifacts、ADRF records／model storage 與 ML volume
  contents，但保留 images、containers 與 volumes；
- reset 會清除 runtime 中的 seed artifact copy；下一次 coordinator 啟動時，應從 image 內 canonical
  deterministic seed source 重新匯入並驗證相同 artifact key，使 clean run 回到相同 initial model；
- reset scope 必須由 selected config manifest／runtime inventory 產生。現有只列出 A／B／C 與五個 ML
  containers 的 hard-coded reset 清單必須在 Phase 2 擴充，不能遺漏新增 Client／Leaf／Branch state。
- selected config 必須和 Guest active config hash、container config labels 相符；stop／status／reset 使用錯誤
  config 時應拒絕並列出 actual inventory，不可部分停止後宣稱完成。
- manifest／inventory 解析失敗、空清單或 declared／actual mismatch 必須 fail closed；不得讓空 Compose service
  arguments 退化成操作完整 project catalog。

### 5.2 2026-08-27 working-tree review disposition

目前 uncommitted Phase 2 prototype 已證明 5／7 個 NWDAF/PyMTLF、8 UE、4×2 ownership、generic systemd
dispatch 與 static topology native loading 可行，但整體實作未通過 review，不得直接 commit。主要拒絕原因為：

- 額外 `DEPLOYMENT`／`deployments/*.yaml`、雙 renderer 與 generate-then-delete 流程；
- static checker shortcut 略過既有 common validation；
- production、Flat、HFL 混合 Compose catalog 與重複 image build；
- selected config 和 actual runtime identity 未綁定，wrong-config stop／status 可漏掉或隱藏 processes；
- manifest failure 可退化成空 inventory，reset volume scope 也未 exact 綁定 topology；
- capacity gate 未加總 HFL container limits，dataset validation 只檢查每條 path 的第一個 data owner；
- tests 尚未涵蓋 switching、failure propagation、unexpected runtime、exact reset 與三 VM activation。

Phase 2 應保留已證明可行的 dynamic inventory primitives，撤掉平行 config／renderer／checker／Compose
workaround，回到三份完整 `TESTBED` definitions、單一 renderer、單一 common-first checker，以及 fail-closed
active-identity lifecycle。

後續 read-only review 已確認完整 repository／Compose checks、`vagrant validate` 與 `git diff --check` 通過，
但也直接重現 stale container config identity 未被 `ml-status` 拒絕，以及 inventory parser failure 退化成空清單、
空 Compose target 擴大為全部 17 services。三台 VM 當時均為 poweroff；Host／VM／GPU capacity 只有靜態盤點，
尚未涵蓋 Guest、build／Docker overhead、GPU participant VRAM 或 simultaneous lifecycle。詳細 evidence 與未關閉
項目由 Phase 2 detailed plan 管理，因此本 roadmap 的 Phase 2 狀態仍維持 `Implementation redesign required`。

## 6. 預計修改範圍

`5G_NWDAF_Infrastructure`：

- default production Flat PyMTLF configs；
- static Flat／Hierarchical scenario configs 與 topology files；
- config render、strict validation 與 schema regression tests；
- process／container lifecycle、ports、health checks 與 evidence collection；
- 後續確認後的 VM placement、network aliases 與 service discovery settings。

`testbed-docs`：

- active migration plan；
- 核准後的 experiment matrix；
- 每次實際執行的 revision、config identity、environment、commands、results 與 limitations records。

本計畫不修改 NWDAF／PyMTLF architecture ownership、不新增 public SBI，也不在 testbed repository 複製
component source repository 已擁有的 generic config reference。

## 7. Phased roadmap

每個 phase 都使用獨立 scope、verification 與 commit gate。前一 phase 的 acceptance 未滿足前，不把下一
phase 的 topology、process placement 或 algorithm change 混入同一 implementation diff。

| Phase | 狀態 | 目標 | 主要變更 | 完成閘門 |
| --- | --- | --- | --- | --- |
| 0. Component baseline freeze | Completed (`8311883`) | 固定已驗證的 source revisions | parent gitlinks、lock 與 Compose build identity | parent baseline review／commit，repository 與 component tests 通過 |
| 1. Existing production Flat migration | Completed (`072748b`) | 讓既有 A／B／C flow 使用新版 config schema 且保持原行為 | default PyMTLF configs、renderer、strict validation 與 regression tests | native config load、repository／Compose checks、既有 Flat runtime regression、user review／獨立 commit |
| 2. Static scenario common foundation | Implementation redesign required；working-tree review completed | 在既有三台 VM 固定八個 UE、四個 data owners、獨立 NF identity 與最大 topology placement | 三份完整 testbed definitions、單一 renderer／checker、fail-closed dynamic lifecycle、ports、collection profiles、exact reset／seed restoration 與 actual capacity gate | 不新增 deployment schema／selector；wrong-config／invalid inventory 被拒絕；兩種 static topology 可 render／validate／activate，尚不宣稱 full flow |
| 3. Static Flat flow | Pending | 跑通一 Server／四 Client controlled flow | static Flat topology、private collection、manual training 與 lifecycle integration | real collection→FedAvg→publication→cleanup evidence |
| 4. Static Hierarchical flow | Pending | 在相同四個 data owners 上加入兩個 Branches | Root／Branch／Leaf topology、Branch lifecycle、two-tier aggregation 與 observability | real collection→Leaf fitting→Branch／Root FedProx→publication→cleanup evidence |

下一個 implementation phase 是 Phase 2；其 decision baseline 與詳細 slices 由
[Phase 2 Static Scenario Common Foundation Detailed Plan](phase-2-static-scenario-common-foundation.md) 管理。
開始時先完成既有三台 VM 與 host 的 capacity／endpoint inventory，不直接新增 VM。Phase 1 的詳細範圍、
結果與 verification gates 由
[Phase 1 Production Flat Config Migration Detailed Plan](phase-1-production-flat-config-migration.md) 管理。

本 roadmap 的完成終點是 Phase 4：既有 production Flat、static Flat 與 static HFL 都完成所需 testbed
flow 與 evidence。Flat FedProx 與跨 topology controlled comparison 不影響 Phase 0–4 的完成判定。

## 8. Acceptance criteria

### 8.1 Config 與 topology

- 三條配置線都通過 parent config checks 與 pinned PyMTLF native loader；
- production Flat 的 participants 仍由 monitor scopes 產生；
- static Flat exact topology 恰為一 Server／四 Clients；
- static Hierarchical exact topology 恰為一 Root／兩 Branches／四 Leaves，且每個 Branch 恰接兩個 Leaves；
- 每個 topology node 都映射至獨立 NWDAF NF identity 與獨立 PyMTLF state；
- Branch 沒有 autonomous orchestration 或 private collection ownership；
- Client 1–4 與 Leaf 1–4 的 data partition identity 可一一對應；
- 八個 UE／SUPI 在單一 scenario 內恰由四個 data owners 互斥擁有，每個 owner 恰有兩個。

### 8.2 Flow

- 每個 data owner 只蒐集自己 area／SUPI 範圍的 observations；
- private collection 可 create、observe、DELETE，並產生可匹配的 retained descriptor；
- training request 能使用 actual collected snapshot；
- Flat 完成 Client fitting、Server aggregation、validation 與 publication；
- HFL 完成 Leaf fitting、Branch aggregation、Root aggregation、validation 與 publication；
- failure、timeout、restart 與 cleanup 不遺留錯誤 remote subscription 或 stale active state；
- active scenario 存在時，以另一個 `CONFIG_DIR` 啟動必須明確失敗；只有 operator 明確 stop／reset 後才可
  切換；
- reset 後 runtime state 為空，下一次啟動重新匯入 canonical seed 並得到預期 artifact identity。

### 8.3 Evidence 與宣稱限制

- 每次 run 記錄 parent／component revisions、dirty flags、rendered config hash、artifact identity 與 dataset
  evidence；
- support callback replay 與 real SMF／UPF runtime evidence 分開標示；
- FedAvg Flat 與 FedProx HFL 的第一階段結果不作公平效能比較；
- VM／full-core 未執行前，狀態維持 Testbed Validation Pending。

## 9. 明確延後

- Flat FedProx implementation、algorithm transport 與 controlled comparison；只有在本 roadmap 完成後才
  另立 future plan；
- dynamic HFL／`hierarchical + monitor_scopes`；
- arbitrary-depth hierarchy 或 topology hot reload；
- Root／Server 代替 data owners fan out collection；
- 未經 placement inventory 即新增、重建或刪除 VM；
- production benchmark 或 real application traffic 宣稱。
