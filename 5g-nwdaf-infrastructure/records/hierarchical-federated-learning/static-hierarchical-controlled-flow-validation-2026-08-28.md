# Static Hierarchical Controlled Flow Validation Record

執行日期：2026-08-28

User review：2026-08-28 confirmed

狀態：Completed / Verified Record；implementation、real three-VM acceptance、Static Flat regression、
mandatory review、user review 與 Infrastructure source commit 已完成

## 1. 目的與宣稱邊界

本 record 保存新版 testbed Phase 4 Static Hierarchical controlled flow 的實際執行證據。驗證流程為：

```text
四個 Leaf private collections
→ 四份 retained 2-SUPI snapshots
→ 一個 operator-triggered Root request
→ exact 2 Branches × 2 Leaves
→ 兩輪 Leaf FedProx fitting
→ Branch lower-tier sample-count-weighted aggregation
→ Root upper-tier sample-count-weighted aggregation
→ four-Leaf final validation
→ ADRF 與 Root catalog publication
→ exact 2 upper + 4 lower resource cleanup
```

本次結果只證明上述 lifecycle、identity、two-tier artifact lineage、publication 與 cleanup contract
在指定 revisions、generated artifacts 與三台 VM 環境上閉合。不宣稱 model-quality benchmark、communication
cost、production throughput、HFL 優於 Flat，或 FedAvg／FedProx 與 topology 差異的單一因果效果。

實驗契約由 [Static Hierarchical controlled-flow definition](../../experiments/static-hierarchical-controlled-flow.md)
與 [Phase 4 detailed plan](../../plans/hierarchical-federated-learning/phase-4-static-hierarchical-flow.md) 管理。

## 2. Source、config 與 runtime identity

| 項目 | 實際 identity |
| --- | --- |
| Infrastructure branch／source commit | `feat/r18-hierarchical-federated-learning`／`9a7bc914f40f3ab5e4f13b19a538c0a2a3c4d754` |
| Go NWDAF | `6aed268d6528f8be6c729cbd45b59d067e5e80dc` |
| ADRF | `905f0599f68fe389bba14ed56db0ef9abeab5ccd` |
| PyMTLF | `747962971b63f0a53031d52a1eb7e047ae776998` |
| Selected testbed | `testbed.static-hierarchical.yaml` |
| Scenario | `experiments/examples/fl-closure-smoke/scenario.yaml` |
| Generated config | `phase4-static-hfl-v1:406b7fcd0fe7e5fccd74c41c7f8ed121a55e33b2c2da37a0a5d10c208237b51e` |
| Dataset identity | `68b65b9bacc58ba30d53bc8ece46e0374185d8301f993839b4645fde90830da9` |
| PyMTLF image | `sha256:6f7cb98ccf796ae75bc32ff744eccc03ff428884a5610ee658c0d2473eb4ce3c` |
| OCI revision label | `747962971b63f0a53031d52a1eb7e047ae776998` |
| Run identity | `84708477-5dca-499f-a8a0-640b25a30c8c` |

Runtime 使用既有 `core`、`path-a`、`path-b` 三台 declared VMs。Provider 操作全部經 approved Host
context；startup 前第一個 runtime observation 是 Host OS process inventory，並與 declared identities 及
provider state exact compare。三台 VM 各只有一個 matching process，沒有 duplicate 或 orphan process。

Capacity gate 記錄 24 CPUs、約 16.5 GiB `MemAvailable`、149 GiB free disk，以及 9988 MiB free GPU
memory，高於 8192 MiB requirement。Host 沒有 swap，作為 non-blocking warning 記錄；實驗期間沒有
capacity rejection。

Selected／active identity、29 個 Guest units、八個 UE Registration／PDU Sessions、七個 unique NWDAF NRF
profiles 與七個 healthy PyMTLF containers 一致。Static deployment 沒有啟動 production Consumer
subscriptions。

## 3. Leaf collection evidence

四個 Leaves 各只解析自己的 Internal Group 與兩個互斥 SUPI；八個 SUPI 沒有跨 owner overlap。
Root 與兩個 Branches 沒有收到 collection request。每個 Leaf 均保存 2 records 與 200 observations，
且兩個 SUPI 都具有 current-run real SMF／UPF callback evidence。

Operator 停止 collection 後，四份 resources 均為：

- `state=RETAINED`；
- `descriptorState=RETAINED`；
- `activePeerResourceCount=0`；
- `pendingCleanupPeerResourceCount=0`；
- `cleanupPending=false`。

Training 在 preparation window 內使用同一個 `RUN_ID`；沒有新增 dataset selector，也沒有 fallback
到 Consumer descriptor 或未證明的 Mongo data。

## 4. Hierarchy admission 與 two-tier training evidence

| 項目 | Identity／result |
| --- | --- |
| Root request | `84708477-5dca-499f-a8a0-640b25a30c8c` |
| Root plan | `9b53e445-f08f-4eff-a734-c0211fcfd1ad` |
| Root training process | `c69d4f16-7c47-4afc-b3e2-b9edad3e168b` |
| Branch 1 NF／lower process | `66666666-6666-4666-8666-666666666666`／`24e0edaf-8c5d-4492-b6e1-5a13d5986c6f` |
| Branch 2 NF／lower process | `77777777-7777-4777-8777-777777777777`／`b4cd31f0-0d12-4e04-99b3-f4857c0e0ee2` |
| Branch 1 Leaves | `11111111-1111-4111-8111-111111111111`、`44444444-4444-4444-8444-444444444444` |
| Branch 2 Leaves | `22222222-2222-4222-8222-222222222222`、`55555555-5555-4555-8555-555555555555` |
| API contract | `mode=hierarchical`、`participantSource=static`、`triggerSource=private_api` |

Root exact admission 兩個 Branches，每個 Branch exact admission 兩個 Leaves。四個 Leaves 在 rounds 0 與 1
各用 36 positive samples 完成 FedProx fitting；因此每個 Branch aggregate 的 effective sample count 為
72。

| Round | Branch 1 lower digest | Branch 2 lower digest | Root output-lineage digest |
| --- | --- | --- | --- |
| 0 | `d7ce43b2ce7b58ead87bdc00d978fbd0eb89a8bd6aa91a5786f5b9dfe488224d` | `648651db99283d2a7c08f036b50df65c07a404c3c449256cb1bc1cee9c35d6ff` | `eaffbf82bebdea7066b93388fab3b9a3f0c8c2d36f08b29f0c36921ab8fdab53` |
| 1 | `bfc185cd400d3075659d25e6df14c22688acb11f9a6af05773c3e2c58414e6bf` | `ebdfe6a733a38fa797b7b985c1f8129b27362ae860e3c073e4d00bbac796d349` | `48d3d1089df6edcafb38b9511d69c42dddfe6972710d22c6b14f8eafcacda750` |

Leaf local artifact URLs／digests、Branch lower artifacts、Root round inputs 與 Root global artifacts 已在執行時
exact compare，上表的 parent／round lineage 一致。這是 artifact evidence，不是由缺少 aggregate
event 的 status log 間接猜測。Round 0 Root output 在下一輪以 `ROUND_INPUT` identity 被兩個 Branches
取得；Round 1 的 final Root output 為 `ROUND_GLOBAL` candidate。

## 5. Validation、publication 與 cleanup evidence

- 四個 Leaves 均完成 round 2 final validation，每個使用 4 validation samples；
- base WAPE：`2.3936889687029064`；candidate WAPE：`0.2922181181149421`；
- `gate_would_accept=true`，但 bounded-smoke performance gate 依實驗契約保持 disabled；
- candidate digest：`48d3d1089df6edcafb38b9511d69c42dddfe6972710d22c6b14f8eafcacda750`；
- final bundle digest：`43485c824b1fdfdfd51fc6aa81788e2bcee5ffc899cf7a80f726dc69c6be59d8`；
- publication ID：`2997c4f6-2e69-43cf-8eee-71c92f593d76`；
- published model ID：`1787852091404`；
- top-level terminal state：`COMPLETE`；`required_scopes=0`。

ADRF record GET、stored artifact SHA、published model identity 與 Root durable catalog current family revision exact
compare 通過。Durable catalog 記錄 generation 2，兩個 Branch participants 各有 `sampleCount=72`。Static
flow 不建立 Consumer scope，不執行 PyAnLF reprovision 或 generation cutover。

Root 建立兩個 upper resources，兩個 Branches 各建立兩個 lower resources。六個 create 均有對應
terminal `DELETE status=204`；之後六個 resource GET 均為 404。七個 FL workspaces 沒有 active
plan content，且沒有 cleanup failure、stale current-run correlation 或 unexpected active resource。

`ml-status` 完成 HFL milestones 解析，但刻意保持 `verification-incomplete phase=top-level-status`；
terminal top-level API state 由同一 `RUN_ID` 的 `fl-training-status` 獨立證明，兩個 evidence sources
沒有矛盾。

## 6. Stop、reset 與 seed recovery evidence

Successful HFL flow 的 ordinary `experiment-stop` 停止 selected Guest／ML processes，保留 VM 與指定
retained assets。

Selected guarded reset 的 plan exact 列出七個 HFL volumes，並排除其他 topology containers／volumes。
執行後：

- 七個 selected volumes 全部為空；
- ADRF 清除 296 個 data records、2 個 model records 與 2 個 model files；
- collection、training、hierarchy-plan、publication 與 catalog state 無 stale content；
- Root restart 後 catalog 為 generation 1、model ID `1`、origin `SEED`；
- canonical seed artifact key 恢復為
  `a2c796a001e2da2461418f80b01d7d1e33f0e3349c2817d92286f09e67aa6bef`。

Reset／restart 驗證完成後再次執行 ordinary stop。VMs 保留，沒有執行 destroy。

## 7. Static Flat regression

Phase 4 對 shared operator、renderer、status 與 container lifecycle 的變更使用 fresh Static Flat run
驗證，不借用 Phase 3 舊 evidence。

| 項目 | 實際 identity／result |
| --- | --- |
| Generated config | `phase4-static-flat-regression-v1:cb2612945ce67e31a0336afa8a2d9714fc140e7b50b0c1571c79e1e82513836e` |
| Dataset | `68b65b9bacc58ba30d53bc8ece46e0374185d8301f993839b4645fde90830da9` |
| PyMTLF image | `sha256:2b96d307bf7285103bb590a29f6740f96e3c9e1aa3774169ba18501c62ce3f6f` |
| OCI revision | `747962971b63f0a53031d52a1eb7e047ae776998` |
| Run identity | `795f25c7-95bc-4321-b7ba-fe5f27c3a011` |
| Training process | `61d80be2-beeb-41d8-8084-40f67bf63d2b` |
| Candidate digest | `a9a7d9b2bf23e8c4e4a79aab4e7d1f0d171e06740ab38d9082e806fdec5bdaba` |
| Published model | `1787853429655` |
| Cleanup | `created=4 deleted=4 active=0` |
| Final result | `COMPLETE`；`required_scopes=0` |

四個 Flat owners 各解析兩個互斥 SUPI，各有約 198–200 current-run observations。每輪四個 Clients
各使用 36 samples 完成 FedAvg，四個 final validations、ADRF publication、4／4 cleanup 與 ordinary
stop 均通過。Base WAPE 為 `0.19506470978235016`，candidate WAPE 為 `0.34657703484375657`，
`gate_would_accept=false`；performance gate 在 bounded smoke 中 disabled，因此此數值不影響 functional
regression，也不用來作 model-quality claim。

## 8. Verification results

| 驗證 | 結果 |
| --- | --- |
| Infrastructure focused `fl-control`／`ml-status`／`runtime-inventory` tests | PASS |
| Infrastructure full `make test` | PASS；synthetic provider fixtures，不啟動 real provider process |
| Disposable production-Flat／HFL CPU container lifecycle | PASS；5／7 containers、identity、health、retention 與 cleanup 通過 |
| PyMTLF focused native suite | PASS；`257 passed`、`37 warnings` |
| Ruff correctness rules | PASS；`E4,E7,E9,F` |
| Bash syntax／`git diff --check` | PASS |
| Real three-VM Static HFL flow | PASS |
| Real three-VM Static Flat regression | PASS |
| Selected guarded reset／seed restart | PASS |
| Mandatory initial／targeted review | PASS；所有 admitted findings 已關閉 |
| User review | PASS；2026-08-28 confirmed |

## 9. Limitations 與後續

- 本次是 bounded lifecycle smoke，不是 performance／quality benchmark；
- HFL 的 `gate_would_accept=true` 與 Flat regression 的 `gate_would_accept=false` 都只是未啟用 gate 的
  診斷值；
- hierarchy assignment duplicate GET、transport optimization、communication instrumentation 與 component
  remediation 屬其他 workstream，本次沒有修改或驗收；
- 沒有新增 HFL-only API、script、config source、VM、service 或 persistence；
- guarded reset 依契約清除 mutable run artifacts；精確 identities、API results、artifact lineage 與
  cleanup evidence 已在 reset 前取得，stopped container logs 保留可重新核對的 direct evidence；
- Infrastructure source 已提交為 `9a7bc914f40f3ab5e4f13b19a538c0a2a3c4d754`；本 verified record 隨
  testbed-docs 的 approved Phase 4 completion commit 保存。
