# Full-core Business FL E2E（2026-08-11）

## 結論

`full-core-cat-transition` business scenario 已完成第一次完整閉環。單一 consumer 經
NRF 找到 NWDAF-A／B，兩側由真實 registration、PDU Session、serving-SMF resolution
與 Nupf Event Exposure 建立資料路徑；Path A 的可重現 PseudoDriver degradation 自動
觸發 C 協調 A／B 執行 federated learning。兩輪 FedAvg、final validation、ADRF model
publication、A／B reprovision、兩個 scope adoption 與 monitor generation cutover 全部
成功。

本輪同時驗證 PyAnLF collection/runtime revision race 的修正。Initial model activation
仍會使舊 revision task 失效，但 A／B 現在各只記錄一次 concise INFO cancellation，沒有
`Collection task failed`、traceback、舊 revision retry 或重複 peer resource。

PseudoDriver 僅提供可重現的 traffic stimulus；本結果不代表真實 application traffic
效能或 user-plane throughput benchmark。

## Runtime identity

- Infrastructure：`5886c7f4ecd5b86985be13166d4fe9fcbc73299c`
- NWDAF：`318ac19d8b027373f4468660394da1ec3338268e`
- PyAnLF：`08798f15c3693027e00bc60dd53f74ebaa26c3a1`
- PyMTLF：`7e8ab7f23bf5d6398eb1cd5f053dd8bda9439a87`
- ADRF：`905f0599f68fe389bba14ed56db0ef9abeab5ccd`
- SMF：`128b0ec6157238efe4203e2060415728599ada04`
- go-upf：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- UERANSIM：`2a3ef81f189ca95d5c1996a28ed7af9734f5cfb4`
- gtp5g：`8d723c29fc0de3eeeff3e9a91132838579e8ee1b`
- Scenario：`full-core-cat-transition`（`business-acceptance`）
- Config source SHA-256：`6fad76decad6a7ebfd5a324b86bcff6a6539b0628e706e7dd4c9fc8dad17d55d`
- Effective config set：`default-9ed1c7041b7072ff`
- Dataset set：`c3b428ea763834f34b2ff3a7e7674b5d082a2685e3825595f0b5cc33c356bb49`
- VM OS：Ubuntu 22.04

執行前先以 confirmation-gated experiment reset 清空五個 ML state volume、ADRF data／
model collections、ADRF model artifact 與其 NRF state。Reset verify 確認 scoped state 為空，
containers、images、network 與 named volumes 均保留。

## 啟動與 initial revision race

三台 VM、23 個 Guest units 與五個 Host ML containers 全部 active／healthy。PyMTLF-A／B
以 `cuda:0` 啟動，CDI selector 為 `nvidia.com/gpu=all`，GPU probe 正確辨識 RTX 3080。

Consumer 在 `2026-08-11T15:04:19Z` 建立兩筆 subscriptions：

| Path | NWDAF NF instance | Subscription |
| --- | --- | --- |
| A | `11111111-1111-4111-8111-111111111111` | `8c179907-6243-453d-a42e-04fcb1246d30` |
| B | `22222222-2222-4222-8222-222222222222` | `c06b99ff-b6c9-4c78-835b-b94bb1d64feb` |

Seed model activation 後，PyAnLF-A／B 分別出現一次：

```text
Discarded inactive or superseded collection task: ... revision=1 reason=StaleRuntimeRevisionError
```

五個 ML container 的本輪完整 log 掃描結果：

- `ERROR`：0；
- traceback：0；
- `Collection task failed`：0；
- A／B superseded cancellation：各 1；
- A／B `Provisioned model activated`：各 2（seed 與 completed model）。

因此 authoritative runtime revision fence 仍有作用，但 superseded work 已成為 no-retry
terminal cancellation，符合修正預期。

## Collection、analytics 與 monitor

UPF-A／B 各建立且只建立三筆 EES subscriptions，對應各 Path 的三個 UE；model cutover
沒有新增第二組 resources。兩側持續以 30 秒 sampling 向 PyAnLF callback，HTTP 回應皆為
204。每個 90 秒 monitor window 最多匹配三個 time-slot pairs。

Reference period 中 A 的 deviation 約為 `2.43`，B 約為 `2.20`，兩側前十個 windows 均
`triggered=false`。Path A 進入 degradation 後：

| UTC | A deviation | C decision |
| --- | ---: | --- |
| `15:21:21` | `3.7312` | not triggered |
| `15:22:51` | `14.4757` | not triggered |
| `15:24:21` | `12.4588` | triggered |

Path B 同期仍約為 `2.20`，沒有觸發。觸發發生於 subscription 後約 1,202 秒，早於
scenario 的 1,290 秒 bounded trigger。Consumer 在最終讀取時仍 active，累計 88 筆
analytics notifications，最後通知時間為 `2026-08-11T15:29:21Z`。

## Federated training 與 publication

PyMTLF-C 在 `2026-08-11T15:24:21Z` 建立 federated process
`5cd43ea9-cf96-4760-b0c2-ad04d73edac8`。A／B preparation 分別取得 114／117 筆 records，
兩輪 training 的實際 sample counts 為 A 6、B 7：

| Stage | A | B | C |
| --- | --- | --- | --- |
| Preparation | 114 records | 117 records | two participants ready |
| Round 0 | 6 samples | 7 samples | FedAvg complete |
| Round 1 | 6 samples | 7 samples | FedAvg complete |
| Final validation | 1 sample | 1 sample | evaluated |

Final validation 結果：

- base WAPE：`1.8392504554`；
- candidate WAPE：`0.3226602284`；
- `gate_would_accept=true`；
- deployment policy 的 gate enforcement：`false`。

C 將 completed model `1786461865081` 發布到 ADRF，store transaction 為
`585ef6d2-ee6f-44cc-b782-985ea5a5b0bc`，artifact 大小 362,250 bytes。ADRF record 的
allow list 同時包含 NWDAF-A、B、C。最終資料庫狀態為：

- `adrf.data_store_records`：318；
- `adrf.mlmodel_store_records`：1。

A／B 都啟用 model `1786461865081`；C 隨後收到兩個 required scopes 的 adoption，並於
`15:24:25Z` 記錄 `Federated model cutover complete`。從 trigger 到 cutover 約四秒。

## Post-cutover evidence

Cutover 後 A／B 使用新 model ID 建立新的 accuracy measurement namespace，而不是繼續把
舊 generation 的 evidence 混入新模型。兩側均重新累積到 `matched=3` 並由 C 回覆 204：

- B deviation：`0.3149`、`0.2960`；
- A deviation：`3.2879`；
- C decisions：均 `triggered=false`。

這證明 publication log 之後 production inference、accuracy delivery 與 monitor generation
確實已切換並持續運作。

## 資源與 teardown

驗收末端：

- PyAnLF-A／B RSS 約 280 MiB；
- PyMTLF-A／B RSS 約 702 MiB，PyMTLF-C 約 292 MiB；
- PyMTLF-A／B GPU process 各使用約 392 MiB VRAM；
- Host 約有 24 GiB available RAM；
- swap 仍僅約 8 MiB free，未觸發 RAM hard gate。

結束時對兩個 exact subscription locations 執行 DELETE，停止 consumer、五個 ML
containers、23 個 Guest services，最後 graceful halt 三台 VM。VM 最終均為 poweroff；
containers、images、named volumes 與本輪可供後續查驗的 state 均保留。

Core VM 每次從 poweroff 啟動後，reset plan 曾先發現 MongoDB alias `192.168.57.18` 尚未
恢復並安全停止；重啟既有 `5g-nwdaf-network.service` 後所有 aliases 正常。後續
`services-start` 也依設計重新 reconcile 三台 guest network。此項不影響本輪 E2E 結論，
但後續若要讓 reset 成為開機後第一個操作，可再改善 VM-up/network readiness sequencing。
