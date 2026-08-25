# FL Closure Smoke 與 Persistent Netplan 回歸（2026-08-12）

## 結論

`fl-closure-smoke` 已在 Persistent Netplan migration 後，從三台 VM poweroff 與已清空的
實驗 state 完成一次完整回歸。Core、Path A、Path B cold boot 後不需啟動或重啟
`5g-nwdaf-network.service` 即具備全部 process aliases；reset 可在 services-start 前直接連上
MongoDB alias。後續 23 個 Guest units、五個 Host ML containers、單一 consumer 的兩筆
subscriptions、A-only degradation、兩輪 federated learning、ADRF publication、A／B
reprovision 與 monitor generation cutover 全部成功。

本輪沒有修改 NF／ML source 或 component revision。PseudoDriver 只提供可重現的 traffic
stimulus；本結果證明 bounded FL closure 與 Netplan migration 沒有破壞完整資料路徑，不代表
真實 application traffic benchmark，也不取代 `full-core-cat-transition` 的 business timing
驗收。

## Runtime identity

- Infrastructure：`7699574 fix(network): verify runtime convergence after rollback`
- NWDAF：`318ac19d8b027373f4468660394da1ec3338268e`
- PyAnLF：`08798f15c3693027e00bc60dd53f74ebaa26c3a1`
- PyMTLF：`7e8ab7f23bf5d6398eb1cd5f053dd8bda9439a87`
- ADRF：`905f0599f68fe389bba14ed56db0ef9abeab5ccd`
- SMF：`128b0ec6157238efe4203e2060415728599ada04`
- go-upf：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- UERANSIM：`2a3ef81f189ca95d5c1996a28ed7af9734f5cfb4`
- gtp5g：`8d723c29fc0de3eeeff3e9a91132838579e8ee1b`
- Scenario：`fl-closure-smoke`（`bounded-smoke`）
- Config source SHA-256：`ae5262f62caa92010af0aafcffe7435d61e3640a54c7f1aed1a2520390281630`
- Effective service／Compose config：`fl-closure-smoke:28d59f6111cf`
- Dataset：`2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`
- VM OS／provider：Ubuntu 22.04／VirtualBox

執行前 config checker、20-case negative contract smoke、baseline／CPU Compose checks、dataset
identity 與 timing checks 全部通過。Dataset 每個 Path 各 23,760 rows，historical burst 可形成
100 observations；sampling 為 30 秒、monitor period 為 90 秒、minimum matched samples 為 2。
預估 earliest trigger 為 450 秒、bounded trigger 為 570 秒，並保留 300 秒 closure budget。

## Cold boot、network 與 clean state

三台 VM 由 poweroff 直接 `vagrant up --no-provision`。Cold boot 後：

- Core 14 個、Path A 7 個、Path B 7 個 process aliases 全部通過 verify；
- `5g-nwdaf-network.service` 維持 inactive，boot journal 沒有執行紀錄；
- `services-start` 對三台都只執行 `NETWORK VERIFY`，沒有 `NETWORK RECONCILE`；
- reset plan 在 services-start 前即可經 `192.168.57.18` 讀取 MongoDB。

確認 plan 後，以 scenario 名稱通過 confirmation gate 清除五個 named volume 內容、ADRF data／
model collections、model storage 與 NRF ADRF state；reset verify 全部為空。Containers、images、
network 與 named volumes 均保留。這一段直接回歸 Persistent Netplan migration 原本要解除的
boot-time alias race。

## Startup、subscription 與 stable reference

23 個 Guest units 全部 active，五個 ML containers 全部 healthy。PyMTLF-A／B 都以
`cuda:0`、NVIDIA runtime 與 `nvidia.com/gpu=all` CDI selector 啟動；PyAnLF-A／B 與
PyMTLF-C 使用 CPU。

Consumer 在 `2026-08-12T03:23:47Z` 經 NRF 建立且只建立兩筆 subscriptions：

| Path | NF instance | TAI TAC | Subscription |
| --- | --- | --- | --- |
| A | `11111111-1111-4111-8111-111111111111` | `000001` | `c6a118e5-84e3-416d-a78c-03ee4eccd152` |
| B | `22222222-2222-4222-8222-222222222222` | `000002` | `8cb90a56-b0a1-4b87-8bd2-96cb35df5465` |

Stable windows 中 A deviation 約 `2.43`，B 約 `2.20`，C 均記錄
`evaluated=[True] triggered=[False]`。報告正常具有 `matched=2`，後續多為 3；沒有為 smoke
降低 accuracy sample minimum。

## Degradation、training 與 publication

第一份涵蓋 Path A degradation 的報告於 `03:31:48Z` 形成：

- A：model `1`、`matched=3`、deviation `5.9568`，C 判定 `triggered=true`；
- B：model `1`、`matched=3`、deviation `2.1988`，同輪不再觸發另一個 process；
- trigger 距 subscription 約 481 秒，落在 450–570 秒的預期窗口內。

C 建立 federated process `47cb7bca-e31e-49a4-9e71-4959075273b0`。A／B preparation 分別
取得 45／33 筆 records；兩輪 local training 都各使用 49 個 samples，final validation 各使用
5 個 samples。C 依序完成 round 0、round 1 FedAvg，沒有第二個 federated process。

Final validation 結果：

- base WAPE：`1.8397703958`；
- candidate WAPE：`0.2463687766`；
- `gate_would_accept=true`；deployment policy enforcement 為 `false`。

C 發布 model `1786505512331`，publication ID 為
`a6610c22-c0b8-49a9-9321-d192fcdfadce`。A／B 都從 ADRF 的相同 model record 下載並啟用該
model；C 在兩個 required scopes adoption 後於 `03:31:52Z` 完成 cutover。Trigger 到 cutover
約四秒。

## Post-cutover 與 error evidence

新 generation 沒有沿用 seed model 的 accuracy namespace：

- A 第一份新模型報告為 model `1786505512331`、`matched=2`、deviation `7.0558`，C 正常評估且
  沒有再觸發；
- B 第一份新模型報告只有 `matched=1`、deviation 尚不可用，C 依既有 minimum 規則暫不評估。

因此本輪證明新 model 已實際進入 production inference、accuracy delivery 與 monitor route；
單一短窗口的 A deviation 不能解讀為 deployment 後穩態品質提升，品質比較仍應由較長的主
example 驗收。

五個 ML container 的本輪 log 掃描結果皆為：`ERROR` 0、traceback 0、
`Collection task failed` 0。Initial model activation 仍使 A／B 各有一次預期的 stale revision
terminal cancellation，沒有 retry storm。

## 資源、保留狀態與 teardown

驗收末端：

- PyAnLF-A／B 約 281 MiB；
- PyMTLF-A／B 約 706 MiB，PyMTLF-C 約 293 MiB；
- concurrent training 的兩個 GPU processes 各約 400 MiB VRAM；
- Host 約有 24 GiB available RAM、221 GiB filesystem free；swap 只剩約 8 MiB free，但未觸發
  RAM hard gate。

唯讀 reset plan 顯示本輪保留 108 筆 ADRF data records、1 筆 model record 與 1 份 model
artifact。Consumer 最終共收到 41 次 notifications；兩個 exact subscription locations 均成功
DELETE。之後依序停止五個 ML containers、23 個 Guest units，最後 graceful halt 三台 VM。
最終三台 VM 均為 poweroff、沒有本專案 container 運行，Host available RAM 回升到約 30 GiB。
實驗 state、containers、images、network 與 named volumes 均保留供後續查驗或由下一次
confirmation-gated reset 清理。
