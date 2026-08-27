# Static Flat Controlled Flow Validation Record

執行日期：2026-08-27

User review：2026-08-28 confirmed

狀態：Completed / Verified Record；implementation、real three-VM acceptance、production Flat regression、mandatory
review、user review 與 repository-separated source commits 已完成

## 1. 目的與宣稱邊界

本 record 保存新版 testbed Phase 3 Static Flat controlled flow 的實際執行證據。驗證流程為：

```text
四個 private collections
→ 四份 retained 2-SUPI snapshots
→ 一個 manual Server training request
→ exact 四-Client、兩輪 sample-count-weighted FedAvg
→ final validation
→ ADRF 與 Server catalog publication
→ exact training-resource cleanup
```

本次結果只證明上述 lifecycle、identity、state、publication 與 cleanup contract 在指定 revisions、generated
artifacts 與三台 VM 環境上閉合。不宣稱 model quality benchmark、production throughput、Flat 優於 HFL，或
FedAvg／FedProx 的公平比較。Static HFL external validation 仍由後續 phase 負責。

## 2. Source、config 與 runtime identity

| 項目 | 實際 identity |
| --- | --- |
| Infrastructure branch／completed commit | `feat/r18-hierarchical-federated-learning`／`e5b1d4472809fba906d1c60aefaf038ca276256d` |
| PyMTLF branch／completed commit | `feat/r18-hierarchical-federated-learning`／`747962971b63f0a53031d52a1eb7e047ae776998` |
| Selected testbed | `testbed.static-flat.yaml` |
| Scenario | `experiments/examples/fl-closure-smoke/scenario.yaml` |
| Generated config | `phase3-static-flat-v1:4fdd0da11fe41de263ec349a1a28a688031d735672f0f4f605d29e78f34396bf` |
| Dataset identity | `68b65b9bacc58ba30d53bc8ece46e0374185d8301f993839b4645fde90830da9` |
| PyMTLF image | `sha256:4680b398c6bfeda4b38127fbf2e1c1d6c82990b33f0a0c9fc42b63cc26c1c9b9` |
| OCI revision label | `36166f04320ae70674604659786ba73935371426` |
| Run identity | `61ae473f-85a7-4a67-a0d8-08ddd1c02600` |

Runtime 使用既有 `core`、`path-a`、`path-b` 三台 declared VMs。Provider 操作全部經 approved Host context；
startup 前先比對 Host OS process inventory、declared identities 與 provider state。Selected capacity gate、三台 VM
identity、Guest active config、27 個 Guest units、八個 UE Registration／PDU Sessions、五個 unique NWDAF NRF
registrations與五個 healthy PyMTLF containers均通過。Static deployment沒有啟動production Consumer subscriptions。

## 3. Collection evidence

四個 data owners 各解析自己的 Internal Group 與兩個互斥 SUPI；八個 SUPI 沒有跨 owner overlap。四個
Client-local requests 共保存 4 records／202 observations，且每個 owner 的兩個 SUPI 都具有 current-run real
SMF／UPF callback evidence。

Operator 停止 collection 後，四份 resources 均為：

- `state=RETAINED`；
- `descriptorState=RETAINED`；
- `activePeerResourceCount=0`；
- `pendingCleanupPeerResourceCount=0`；
- `cleanupPending=false`。

實際 run 另確認同一 Client 若同時存在多個 matching retained groups，`DatasetCoordinator` 會依 component
contract fail closed。沒有新增 dataset selector，也沒有放寬 ambiguity guard；保存 failure evidence 後，以
selected guarded reset 回到 fresh state再執行正式 run。

## 4. Training、validation 與 publication evidence

- Top-level training process：`fcd7c795-7e22-45fc-a9c8-1a6466092dd8`；
- request mode／participant source：`flat`／`static`；
- exact participants：Client 1、2、3、4；
- rounds：0、1；四個 Clients 每輪各使用 37 samples；
- aggregation：每輪收齊四份 positive-sample updates並執行sample-count-weighted FedAvg；
- final validation：四個 Clients 全部完成；
- candidate digest：`3cf53e9c731d8a8adacc6c8f77bbd4a63f489cdd3871484fec895a0137ecf513`；
- base WAPE：`2.3936889687029064`；candidate WAPE：`0.19506470978235016`；
- `gate_would_accept=true`，但 bounded-smoke performance gate 依實驗契約保持 disabled；
- published model ID：`1787842701831`；
- top-level terminal state：`COMPLETE`；`required_scopes=0`；
- training cleanup：`created=4 deleted=4 active=0 unknown_deletes=0`，沒有 failure 或 cleanup failure。

Candidate artifact、ADRF publication reference、published model identity 與 Server durable catalog current family
revision一致。Static flow不建立Consumer scope、不執行PyAnLF reprovision或generation cutover。

## 5. Failure、restart、stop 與 reset evidence

Deterministic operator fixtures與controlled local integration直接覆蓋：

- invalid／empty manifest、wrong selected／active config與unexpected container；
- invalid `RUN_ID`、stale profile、wrong family、404／409／422／503與ambiguous timeout；
- partial collection create與bounded exact rollback，包括cleanup未收斂；
- insufficient／expired descriptor與retained-state preflight；
- collection restart cleanup與preparation failure recovery；
- round timeout與containing-NWDAF generation restart；
- Static Flat failure／publication／cleanup milestone interpretation；
- Static HFL status繼續明確顯示未評估，不借用Flat success evidence。

Selected guarded reset 清除 Static Flat 的五個 ML volumes 與 exact ADRF／NRF／model scope，並確認其他 topology
assets 保留。Server restart 後 canonical seed key 恢復為
`a2c796a001e2da2461418f80b01d7d1e33f0e3349c2817d92286f09e67aa6bef`。Successful flow 的 ordinary
`experiment-stop` 停止 selected Guest／ML processes並保留VM與指定 retained assets。

## 6. Production Flat regression

Regression 使用既有 production Flat flow，而不是 Static Flat manual trigger：

```text
Consumer subscriptions
→ degradation trigger
→ two-client training
→ publication／adoption／cutover
→ post-cutover accuracy
→ cleanup／ordinary stop
```

Identity 與結果：

| 項目 | 證據 |
| --- | --- |
| Generated config | `default:4c42dae057fd400f22ac004bd518b6184c133856579454dc07381dd9cfbe5412` |
| Dataset | `23697bf00ae0560c9f07f8ae451ebb91797943092317aea8cafdb37435c2fd59` |
| PyMTLF／PyAnLF image | `d1140540fde3`／`9b75d16725a5` |
| Consumer callbacks | Path A `47`；Path B `47` |
| Training process | `dff9cf15-4c5a-4fb7-866c-49d045936b5f` |
| Published model | `1787844693934` |
| Post-cutover accuracy | `evaluated=true triggered=false` |
| Cleanup | `created=2 deleted=2 active=0` |
| Final result | `FL RESULT outcome=complete` |

Ordinary stop刪除兩個subscriptions，停止五個ML containers與全部23個Guest units；VM與retained assets保留。
這證明新增 Static Flat operator entrypoints、TAI rendering 與 status interpretation 沒有改寫 production
Consumer／degradation／monitor-scopes／cutover 的 operator semantics。

## 7. Verification results

| 驗證 | 結果 |
| --- | --- |
| Infrastructure focused `ml-status`／`fl-control` tests | PASS |
| Infrastructure Ruff | PASS |
| Infrastructure full `make test` | PASS；synthetic provider fixtures，不啟動 real provider process |
| Disposable five-container CPU lifecycle | PASS；五個 containers、health、identity、retention與cleanup皆符合預期 |
| PyMTLF focused tests | PASS；`68 passed` |
| PyMTLF full suite | PASS；`600 passed`、`46 warnings` |
| PyMTLF Ruff | PASS |
| Real three-VM Static Flat flow | PASS |
| Production Flat regression | PASS |
| Repository `git diff --check` | PASS |
| Mandatory initial／targeted review | PASS；所有 admitted findings 已關閉 |
| User review | PASS；2026-08-28 confirmed |

## 8. Limitations 與後續

- 本次是 bounded lifecycle smoke，不是 performance／quality benchmark；
- performance gate disabled，`gate_would_accept` 只保存診斷結果；
- Static HFL Root／Branch／Leaf external execution由Phase 4管理；
- Flat FedProx、HFL FedAvg及algorithm×topology comparison未執行；
- 沒有新增top-level cancel API、host run ledger、current-run pointer、VM、service或persistence；
- PyMTLF `7479629`與Infrastructure `e5b1d44`保存reviewed source；本record隨testbed-docs commit保存最終
  evidence checkpoint。

Phase 3 acceptance與delivery commits均已完成；push仍須另外取得明確核准，本次未執行。
