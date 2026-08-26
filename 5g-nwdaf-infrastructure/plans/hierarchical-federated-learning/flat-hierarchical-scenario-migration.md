# Flat and Hierarchical Testbed Scenario Migration Plan

日期：2026-08-26

狀態：Phase 1 Completed (`072748b`)；Phase 2 Next；Phase 3–4 Pending

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
- 四個 Leaves 是 data owners，各自使用 private collection profile；
- 第一階段沿用既有 HFL FedProx，僅驗證完整流程與 evidence。

## 3. 跨場景固定關係

Static Flat 與 static Hierarchical 使用相同四個 data-owner positions：

| Data partition | Flat role | HFL role | HFL parent |
| --- | --- | --- | --- |
| 1 | Client 1 | Leaf 1 | Branch 1 |
| 2 | Client 2 | Leaf 2 | Branch 1 |
| 3 | Client 3 | Leaf 3 | Branch 2 |
| 4 | Client 4 | Leaf 4 | Branch 2 |

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

## 5. Logical topology 與 physical placement 分離

邏輯 process 數量為：

- static Flat：五個 PyMTLF processes；
- static Hierarchical：七個 PyMTLF processes。

本計畫目前不決定每個 process 是否需要獨立 NWDAF、VM 或 host container，也不固定 IP／port／VM
placement。Physical placement 必須在盤點現有 three-VM network、資源容量、NRF profile、callback reachability
與 lifecycle tooling 後另行確認；不得為了沿用現有三台 VM 而改變上述邏輯 topology。

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
| 2. Static scenario common foundation | Pending | 固定四個 data owners 的 identity 與最大 topology 所需 placement boundary | scenario config layout、ports、callback reachability、collection profile identity、capacity／placement decision | 1 Server＋4 Clients 與 1 Root＋2 Branches＋4 Leaves 均可被 render／validate，尚不宣稱 full flow |
| 3. Static Flat flow | Pending | 跑通一 Server／四 Client controlled flow | static Flat topology、private collection、manual training 與 lifecycle integration | real collection→FedAvg→publication→cleanup evidence |
| 4. Static Hierarchical flow | Pending | 在相同四個 data owners 上加入兩個 Branches | Root／Branch／Leaf topology、Branch lifecycle、two-tier aggregation 與 observability | real collection→Leaf fitting→Branch／Root FedProx→publication→cleanup evidence |

下一個 implementation phase 是 Phase 2；開始前仍需先確認四個 data owners 的 placement／capacity，
不直接新增四個 Clients、Branches 或新 VM。Phase 1 的詳細範圍、結果與 verification gates 由
[Phase 1 Production Flat Config Migration Detailed Plan](phase-1-production-flat-config-migration.md) 管理。

本 roadmap 的完成終點是 Phase 4：既有 production Flat、static Flat 與 static HFL 都完成所需 testbed
flow 與 evidence。Flat FedProx 與跨 topology controlled comparison 不影響 Phase 0–4 的完成判定。

## 8. Acceptance criteria

### 8.1 Config 與 topology

- 三條配置線都通過 parent config checks 與 pinned PyMTLF native loader；
- production Flat 的 participants 仍由 monitor scopes 產生；
- static Flat exact topology 恰為一 Server／四 Clients；
- static Hierarchical exact topology 恰為一 Root／兩 Branches／四 Leaves，且每個 Branch 恰接兩個 Leaves；
- Branch 沒有 autonomous orchestration 或 private collection ownership；
- Client 1–4 與 Leaf 1–4 的 data partition identity 可一一對應。

### 8.2 Flow

- 每個 data owner 只蒐集自己 area／SUPI 範圍的 observations；
- private collection 可 create、observe、DELETE，並產生可匹配的 retained descriptor；
- training request 能使用 actual collected snapshot；
- Flat 完成 Client fitting、Server aggregation、validation 與 publication；
- HFL 完成 Leaf fitting、Branch aggregation、Root aggregation、validation 與 publication；
- failure、timeout、restart 與 cleanup 不遺留錯誤 remote subscription 或 stale active state。

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
