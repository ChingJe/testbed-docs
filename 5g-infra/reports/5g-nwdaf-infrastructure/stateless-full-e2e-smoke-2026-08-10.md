# Stateless Full E2E Smoke（2026-08-10）

## 結論

新版 stateless backend lifecycle 已完成第一次 bounded full-stack E2E。三台 VM、23 個
Guest services、五個 Host ML containers、六個 UE、兩筆 consumer subscriptions 與雙 Path
PseudoDriver 都能共同運作；A/B 兩邊完成 historical warm-start 及完整快速場景 replay，
consumer 共收到 56 筆 analytics notifications。

這次也直接證明 PyAnLF 的資料實際寫入 ADRF，而不是 MongoDB fallback。不過 accuracy
pipeline 在每個 PyAnLF instance 各產生一筆 measurement 後沒有繼續，因此沒有觸發
degradation、federated training、FedAvg、publication、reprovision 或 generation cutover。
本輪不能宣稱 FL closure 成功。

## Runtime identity

- Infrastructure：`d89ea5393f114d05138ca1ab68eae16c9ef2e2f0`
- NWDAF：`318ac19d8b027373f4468660394da1ec3338268e`
- PyAnLF：`5c305c7b69a50e9356bcfca8229f1a3cffd11a9a`
- PyMTLF：`49b1ef474472559a487b4cf36d312265c45b0c9a`
- ADRF：`905f0599f68fe389bba14ed56db0ef9abeab5ccd`
- go-upf：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- 場景：`fl-closure-smoke`
- VM OS：Ubuntu 22.04
- NWDAF runtime binary：三台 SHA-256 均為
  `79cb52bf340e9e6a8b5a98a322a0ddf89ea6aa189f924c6d2205eb908d5e2fe0`

五個 ML containers 均通過 readiness，且具有不同 `processInstanceId`。PyMTLF-A/B 以
`cuda:0` 啟動並能看到 RTX 3080；本輪沒有進入 training，因此沒有可代表 FL peak 的 VRAM
數字。三台 Guest 對五個 Host endpoints 的 HTTP readiness 全部成功，五個 backend 也都能
呼叫所屬 NWDAF 的 `/internal/v1/nwdaf-context`。這驗證的是 pull-based thin context contract，
流程中沒有使用已移除的 `/internal/v1/sync`。

## 5GC、subscription 與 replay 結果

23 個 Guest units 全部 active，六個 UE 完成 registration 與 PDU Session：

| Path | PDU addresses |
| --- | --- |
| A | `10.60.0.1`–`10.60.0.3` |
| B | `10.61.0.1`–`10.61.0.3` |

單一 consumer 經 NRF 找到兩個不同 NWDAF，建立且只建立兩筆 subscription。兩個 UPF 各建立
三筆 EES subscription，分別對應所屬 Path 的三個 UE。每個 Path 都完成 18,006 packets 的
Phase 1 warm-start，接著完成快速場景全部 26 個 30 秒 Phase 2 windows，最後一個 index 為
125。consumer 最終累計 56 筆 analytics notifications。

## ADRF 與 MongoDB 實際資料路徑

Core MongoDB 的正確 ADRF database 是 `adrf`，不是 free5GC NFs 使用的 `free5gc`。replay
完成後：

- `adrf.data_store_records`：420 筆；
- `adrf.mlmodel_store_records`：0 筆，符合尚未完成 model publication；
- `free5gc.nwdaf_raw_notifications_a`：0 筆；
- `free5gc.nwdaf_raw_notifications_b`：0 筆。

因此本輪即使 PyAnLF config 當時仍允許 MongoDB fallback，實際 observation path 走的是
ADRF。ML containers 啟動初期曾在 Core VM 尚未啟動時留下 MongoDB `No route to host`，之後
沒有持續發生，不能把該 startup warning 解讀為本輪資料來源。

ADRF 對 NRF 的 periodic heartbeat 持續收到 HTTP 400。初始 registration、discovery 與 420
筆資料寫入仍成功，所以 heartbeat 是獨立的 lifecycle 問題，但長時間 E2E 前仍需診斷。

## Accuracy 與 FL 卡點

PyAnLF-A/B 各只記錄一次 `Accuracy measurement ready`；兩筆皆為 `matched=1`、
`inferences=1`、`deviation=None`。兩筆 monitor subscription 於 22:12:04 建立，A/B 的第一筆
measurement 到 22:13:34 才 ready。快速場景的 report period 是 30 秒，而 PyMTLF-C 沿用
`missed_report_threshold=2`、`watchdog_grace_seconds=30`，因此首個 watchdog deadline 是
`30 × 2 + 30 = 90` 秒，恰好也是第一筆 measurement ready 的時間。

A 的首次 Model Monitor notification 對本地 NWDAF internal endpoint 在 5 秒後 timeout；B 的
notification 約 5 秒後到達 PyMTLF-C，但 C 只得到 `evaluated=[False]`、
`triggered=[False]`。C 隨即於 22:13:39 將 A/B 兩筆 monitor subscription 都判定為 missing
periodic reports 並刪除。這解釋了後續不再有可送出的 measurement：monitor resource 已被 C
的 watchdog 移除，不能只歸因為 PyAnLF accuracy worker 自行停止。

所以目前直接 blocker 是「第一筆有效 accuracy report 的 startup latency 與 C watchdog
deadline 相撞」，另需診斷通知經 Path NWDAF 為何接近或超過 5 秒。PDU、EES、ADRF observation
與 consumer analytics transport 不是這個 blocker。A/B 在 seed model activation 附近也各出現
一次 `StaleRuntimeRevisionError`；它可能只是舊 collection task 在 cutover 時被淘汰，仍需
後續核對，但現有證據不足以把它認定為 watchdog expiry 的根因。

依 workspace 的變更邊界，本輪沒有修改 NWDAF、PyAnLF、PyMTLF、ADRF 或 go-upf source。
下一步若確認需要 PyAnLF 修正，應先說明根因與預計修改範圍，再取得使用者同意。

## 資源與 teardown

完整 replay 結束時，五個 ML containers 的 RSS 合計約 1.5 GiB：PyAnLF-A/B 約 320/304
MiB，PyMTLF-A/B/C 約 310/298/284 MiB。三台 VM 的可用 RAM 約為 Core 3.25 GiB、Path A
2.56 GiB、Path B 2.56 GiB。Host 當時約有 19 GiB available RAM；swap 幾乎滿載。

結束時 exact DELETE 兩筆 subscriptions，依序停止 consumer、Guest services、ML containers
與三台 VM。三台 VM 均已 poweroff，五個本專案 containers 保留為 exited，原先八個共用
containers 持續運行。teardown 後 Host available RAM 約 30 GiB，workspace 約 227 GiB free；
swap 仍只有約 8 MiB free。

## 後續 config policy

本輪結束後，Infrastructure default 的 PyAnLF-A/B 已改為 `mongodb.enabled: false`，renderer
產生的完整與快速場景亦繼承此值；config checker 會拒絕重新啟用 fallback。Core MongoDB 不會
移除，因為 free5GC NFs 與 ADRF 本身仍需要它。這項 ADRF-only policy 不涉及 PyMTLF。

default、`full-core-cat-transition-check` 與 `fl-closure-smoke` 均通過 config 與 Compose
checker；PyAnLF loader 確認六份 A/B config 都能解析且 fallback 為 false，PyAnLF
`tests/test_config.py` 為 24 passed。

為先排除首報恰好撞上 deadline 的問題，使用者另行核准 PyMTLF-C deployment config 明確設定
`watchdog_grace_seconds: 300`，沒有修改 PyMTLF source。當時 30 秒快速場景的 deadline 因而
從 90 秒延長為 `30 × 2 + 300 = 360` 秒；完整場景為 `90 × 2 + 300 = 480` 秒。三組 config
均通過 PyMTLF loader，PyMTLF `tests/test_config.py` 為 48 passed。

這項調整只提供足夠的 startup margin，不代表 5 秒 notification latency 已解決；若後續仍無
週期報告，watchdog 仍會正常清除 subscription。

後續短 rerun 中，A/B subscriptions 成功越過舊 deadline，連續兩輪 accuracy notifications
均立即由 C 回 204，沒有再出現 5 秒 timeout。新 blocker 是 30 秒 window 每輪只包含一個
group-level time-slot pair；三個 UE 已聚合為該 pair 的 contributors，因此 report 固定為
`matched=1`、`deviation=None`。

最終決策是不降低 `min_matched_predictions=2`，而是把 `fl-closure-smoke` report period 恢復為
90 秒。這可讓每輪累積約三個 time-slot pairs，並使 smoke 與完整場景的 watchdog deadline
同為 480 秒；快速性仍由 reference/hit policy、stable lead-in 與 local epochs 控制。

### 90 秒 rerun 結果

恢復 90 秒後，A/B 第一份可評估 measurement 都是 `matched=2`，後續 report 為
`matched=3`，且 deviation 均非空。C 依序處理 baseline reports，並在
`2026-08-09T23:44:51Z` 由 Path A degradation 自動啟動 federated process
`9564de02-8464-41a3-ba29-cb66ac605234`。

A/B 各自從 ADRF 取得 27 筆 records，實際 local training 各使用 44 個 samples；兩個 GPU
clients 都產生 round 0 與 round 1 local artifacts。C 在 round 0 成功完成 FedAvg，證明
90 秒 monitor、automatic trigger、ADRF retrieval、雙 client preparation 與 GPU local training
路徑都已跑通。

流程在 C 準備 round 1 FedAvg 時停止。C 會重新下載自己發布於 `http://192.168.57.1:9292`
的 round 0 global artifact，但 rendered `pymtlf-c.yaml` 的
`federated_learning.artifact_download.allowed_origins` 只有 A `:9092` 與 B `:9091`，因此既有
安全檢查回報 `FL artifact origin is not allowed`。這是 infrastructure config 漏掉 C self-origin，
不是 PyMTLF source failure。經使用者確認後，C self-origin 已加入 default、renderer 與 checker；
三組 config check、兩種 Compose check 與 PyMTLF 48 個 config tests 均通過。

### Self-origin 修正後 rerun

更新後 C 能正常下載自己的 round 0 global artifact；A/B 都完成 round 0、round 1 GPU local
training，C 也完成兩輪 FedAvg，原本的 allowlist blocker 已排除。流程下一步進入 client final
validation，但 A/B 都在相同呼叫點失敗：

```text
TypeError: LocalTrainer._predict() missing 1 required positional argument: 'device'
```

PyMTLF GPU 支援 commit `e9c5b08` 將 `_predict` 改為明確接收 device，但
`fl_client.py` 的 base/candidate final-validation 呼叫仍使用舊簽名。現有 test 只確認 validation
task 被排程，沒有實際執行該路徑，因此 48 個 config tests 不會涵蓋這個 runtime error。A/B
最終回報 `NOT_AVAILABLE_ML_TRAIN`，C 在 degradation 持續存在時會於後續 report 再次觸發 FL；
本輪因此主動停止。

## Runtime 後的配置契約修正

使用者核准後，PyMTLF client 已 resolve configured training device，並把同一 device 傳入
base／candidate final-validation prediction；新增 regression test 實際執行 validation 並確認
兩個 call sites。Targeted PyMTLF FL client、trainer、server、dataset、scope、workspace、
artifact、wire 與 config tests 通過；完整 suite 仍受既有 FastAPI/TestClient shutdown hang
阻擋，因此不列為 full-suite pass。

Infrastructure audit 同時確認舊 smoke 的 180 秒 stable lead 只是
`2 references × 90 seconds`，沒有保留第一個不可評估 window 或 sampling phase。修正後：

- sampling 30 秒、report 90 秒、report capacity 3、minimum matched 2；
- historical 3000 秒、stable lead 360 秒、Path A boundary `t=3360s`；
- minimum stable lead 300 秒，bounded trigger 570 秒，絕對 dataset time 3570 秒；
- degraded tail 600 秒，其中明確保留 300 秒 closure budget，minimum tail 510 秒；
- dataset 於 `t=3960s` 結束，新的 set identity 是
  `2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`。

Checker 新增 endpoint/callback、C self-origin、GTP ifname、model shape/interoperability、MongoDB、
FL deadline 與 generated manifest hash/file inventory 對照。Negative contract smoke 分別破壞
report capacity、self-origin、GTP interface、FL public URL、Consumer callback、baseline hash
與 config generator hash，七個 case 均被拒絕。這一輪只產生 audit dataset，未啟動 VM 或
containers。

## 下一個 gate

1. 重跑 `fl-closure-smoke`，確認 final validation、ADRF model publication、A/B reprovision 與
   monitor generation cutover。
2. 核對 `StaleRuntimeRevisionError` 是否只屬於正常 cutover race。
3. 分開診斷 ADRF heartbeat HTTP 400，不把它和已驗證成功的 ADRF data path 混為同一問題。
4. 若再需要修改 PyAnLF、PyMTLF 或其他 NF/ML source，重新停下回報並取得核准。
5. 快速場景成功後，再執行較長的 `full-core-cat-transition` business example。
