# NWDAF replay × testbed alignment plan

## Purpose

這份文件定義 replay 與 testbed 對齊工作的兩個階段：

1. **actual traffic aggregation 對齊**
2. **prediction / accuracy / trigger pipeline 對齊**

目前狀態是：

- 第 1 階段已完成
- 第 2 階段才是下一個主要目標

第一個驗證基準仍使用：

- `5G_Infrastructure/.agent/tmp/0513-13`

這份設計目前主要處理：

1. replay 的流量輸入與 slotting 語意
2. replay 內對 group-level actual traffic 的形成方式
3. replay prediction path 與 testbed NWDAF prediction path 的行為對齊
4. replay accuracy monitor / policy 與 testbed NWDAF 的行為對齊

這份設計**不處理**：

- 模型架構或 scaler 設計
- testbed 自身 `go-upf` 行為修改
- Daisy 訓練流程本身的行為修改

## Background

### What `0513-13` already proved

`0513-13` 的 pseudo-only trace 驗證已經確認：

- pseudo-driver 的 final notify bytes 和 raw `pre_data` **完全一致**
- 驗證方式不是 group-level `slot_raw = floor(ts/30)`，而是：
  - **per-subscription / per-UE rebasing**
  - `wIdx = floor((ts - minTS_for_that_UE) / 30)`
- 六個 IP 都是：
  - `180/180 exact match`

對應報告：

- [testbed-0513-13-pseudo-only-trace-alignment-validation.md](/home/chingje/testbed/docs/5g-infra/reports/testbed-0513-13-pseudo-only-trace-alignment-validation.md)

### What the old replay originally did

`retrain_replay.py` 目前的 `load_group_slots()` 使用的是：

- **group-level rebasing**
- 對整個 group parquet 只取一個 `min_ts`
- 再算：
  - `global_ts = timestamp - group_min_ts`
  - `slot_index = floor(global_ts / report_period_sec)`

而且它直接產生：

- 一個 **group-level aggregated notificationDoc**
- `ueIpv4Addr = "0.0.0.0"`
- `group_id = group1 / group2`

也就是：

- replay 目前不是先產生 per-UE notify，再讓 NWDAF 聚 group
- replay 目前是**直接跳到 group total**

### Why that was a problem

這造成 replay 與 testbed 在資料語意上已經不是同一條路徑：

#### testbed (`UPF EES -> NWDAF`)

1. 每個 UE / subscription 自己找 `minTS`
2. 各自 rebasing
3. 各自切 30s window
4. 以 per-UE notify 送到 NWDAF
5. NWDAF 再按 group 與 `startTime` 聚合

#### replay (`retrain_replay.py`)

1. 整個 group 共用一個 `min_ts`
2. 直接切 group-level slot
3. 直接把 group total 當成輸入

這兩條路徑在：

- `.1 = 0s`
- `.2 = 15s`
- `.3 = 30s`

這種 staggered dataset 上，數學上不等價。

實際比較也已經證明：

- `exp46` 的 `slots.parquet`
- 和 `0513-13` 實際 `NWDAF UPF VOLUME` 在 phase2 的 group total 差異很大

因此如果 replay 要拿來模擬 testbed 的資料路徑，就必須先把這個差異補齊。

## Current status

### Stage 1 is now complete

在目前工作樹中，`retrain_replay.py` 已經完成以下對齊：

1. 不再使用 group-level `min_ts`
2. 先對每個 UE 各自做 rebasing
3. 先產生 per-UE window
4. 再依 `wIdx` 聚成 group-level `SlotObservation`
5. retrain upload docs 也改成沿用 per-UE notify semantics

### What was verified

已用 `0513-13` 做兩層驗證：

1. `UPF EES trace -> raw pre_data`
   - 已由獨立驗證工具確認逐 UE exact
2. `NWDAF latest aggregated slot -> replay slots.parquet`
   - phase2 上：
     - `group1`: `150/150` exact for `UL/DL/Total/Pkts`
     - `group2`: `144/144` exact for `UL/DL/Total/Pkts`
   - 剩餘差異只在 throughput 欄位的 `±1` rounding

這代表：

- **actual aggregation 已對齊 testbed NWDAF**
- testbed 與 replay 的剩餘差異已經不在 actual traffic path

### What still diverges

`exp48` 已經證明：

- actual aggregation 對齊後
- replay 還是會在 `00:33:00 group1` 觸發 retrain
- 而 `0513-13` 的 testbed 不會 retrain

因此目前真正不一致的是：

- `predictedTrafficScale`
- `current/mean/std/zscore`
- `degradationHits`
- 最終 trigger/retrain 行為

進一步比對 `0513-13` 後，現在可以更精確地說：

- **前幾筆 `ML inference` 本身已經幾乎一致**
- **第一個真正的分岔點是第一個 accuracy / policy round**

也就是：

- aggregation 不是主因
- initial model/scaler 不是主因
- early prediction values 也不是主因
- 分歧發生在 prediction 被 monitor 消費、配對 actual、形成 baseline/policy input 時

### Current replay status after monitor alignment

目前工作樹中的 replay 已再補上兩個關鍵修正：

1. `dataset.subscription_to_first_urr_delay_sec: 30`
   - 用來吸收 testbed 中：
     - first URR
     - phase1 history flush
     - first usable monitor window
     的整體有效 delay
2. **same-slot-end batch processing**
   - 同一個 `slot_end` 的所有 group slots
   - 必須先全部 `record_slot()` / `predict_next()`
   - 再跑 due monitor rounds

第二點是這輪才確認的 root cause：

- replay 原先逐筆處理 global slot list
- 遇到 `group1` 當前 slot 時就可能先跑 monitor
- 但同一個 `slot_end` 的 `group2` current actual slot 還沒被記進 state
- 導致 `group2` 早期 policy round 會把某些 prediction 配回前一格 actual

修正後：

- `exp48` 仍維持 `retrain_jobs=0`
- `group1` 前 4 個 policy rounds 幾乎與 testbed 一致
- `group2` 前 4 個 policy rounds 也已幾乎與 testbed 一致

最終定位到的 `group2` root cause 是：

- 不是 delay 本身
- 而是 replay 原先逐筆處理 global slot list
- 在同一個 `slot_end` 上，`group1` slot 先被處理時就可能先觸發 monitor
- 但 `group2` 當前那格 actual slot 還沒 `record_slot()`
- 導致 `group2` 早期 policy round 會把某些 prediction 配回前一格 actual

因此最後的結構修正是：

- **同一個 `slot_end` 的所有 group slots 先全部 record/predict**
- **再執行 due monitor rounds**

這個 same-slot-end batch ordering 修正後，
`group2` 的 early policy rows 才真正收斂到 testbed。

也就是：

- `actual aggregation` 已對齊
- `startup delay` 已有可調 feature
- `early policy sample pairing` 的主要結構性 bug 已修正

目前 `exp48` 已可視為：

- 在 `0513-13` 這個驗證基準下
- replay 與 testbed 的 **policy/no-retrain 行為已實質對齊**

## New problem statement

在 model artifact、scaler、actual aggregation 都已一致的前提下，
testbed 與 replay 的 retrain 行為仍不一致，原因已收斂到：

- **prediction target 定義**
- **historical input 建立方式**
- **ground-truth matching**
- **accuracy monitor warmup / maturity / sample consumption**

換句話說，下一階段的目標是：

- 讓 replay 的 **prediction / accuracy / trigger pipeline**
- 嚴格依照 testbed NWDAF 的現行 Go 實作行為運作

## Strict testbed behavior to mirror

本節只列目前已確認、且 replay 必須嚴格複製的行為。

### 1. Historical input comes from `alignAndZipInMemory()`

testbed NWDAF 在：

- `internal/anlf/analytics.go`
- `alignAndZipInMemory(...)`

會先對每個 corrId/session 做：

1. 以該 session 第一筆 report 作為 anchor
2. 依 anchor-relative grid 做 round
3. 在 session 內 dedup
4. 再用 `snappedNow` 做 global round
5. 聚成最近的 `inputWindow` 個 slots

重要的是：

- 這不是單純「取最後 30 個 group slots」
- 也不是「把 phase1 preload 直接塞進 deque 就算完成」

這是一個：

- **以 absolute snappedNow 為目標**
- **每次推論時重新從 in-memory ring buffer 對齊**
- **再取最近 input window**

的流程

### 2. Prediction target uses `snappedNow`, not `slot_end`

testbed 在：

- `predictionTargetTime(snappedNow, samplingInterval, step)`

定義：

- `step=0` 的 prediction target = `snappedNow`
- 不是 `slot_end`
- 不是 `slot_start`

而 replay 目前在 `predict_next()` 用的是：

- `predicted_at = slot.slot_end`
- `target_time = slot.slot_end`

這和 testbed 不同，是目前最可疑的分岔點之一。

另外，已確認 testbed 第一批有效 inference 的 input 終點 index
應視為：

- `wIdx = 29`

也就是：

- 用 `wIdx 0..29` 這 30 個 slots 當 input
- 去預測下一個 step

這一點先前 replay 曾比 testbed 晚一格，已修正；但修正後仍有分歧，
表示 monitor / policy pipeline 還存在其他差異。

### 3. Ground truth matching is nearest-within-window, not exact timestamp join

testbed 在：

- `monitor.go`
- `lookupGroundTruth(...)`

會對每個 corrId 在：

- `TargetTime ± samplingInterval`

的範圍內找 **nearest** ground truth，再把多個 corrId 的值加總。

這和 replay 目前：

- `(group_id, target_sim_time)` exact lookup

不同。

所以即使 actual aggregation 一致，
只要 target 定義與 matching 規則不同，
`predictedTrafficScale` 與 matched pairs 仍然會分岔。

### 4. Accuracy monitor has startup warmup and discards warmup predictions

testbed 在：

- `internal/anlf/monitor.go`

的行為是：

1. 啟動 accuracy monitor
2. 等待 warmup
3. warmup 完成後：
   - `ConsumeMaturePredictions(0)` 把 warmup 期間 prediction 丟掉
   - `GetAndResetInferenceNum()` 把 warmup inference counter 清掉

也就是：

- warmup 前產生的 prediction 不參與正式 baseline

replay 目前雖然有 `maturity_lag_sec`，但沒有完整複製這套 warmup discard 行為。

### 5. Accuracy monitor only evaluates mature predictions every check interval

testbed 行為是：

1. prediction 持續產生
2. accuracy monitor 每 `checkInterval` 才跑一次
3. 每次呼叫：
   - `ConsumeMaturePredictions(2 * samplingInterval)`
4. 只用已成熟 prediction 組 matched pairs
5. 若 sample 不足，不進 policy

replay 現在的整體節奏比較接近：

- live slot 前進
- 到 monitor 時點就把 exact-target matched predictions 收進來

這兩者的 sample composition 不一定相同。

### 6. First policy round is the immediate validation target

目前最有價值的驗證點，不是直接看最後有沒有 retrain，
而是先看：

- 第一個 `Accuracy policy` round

因為這已經被確認是目前 replay 與 testbed 的第一個主要分岔點。

若這一輪仍對不上，後續：

- `mean`
- `std`
- `zscore`
- `degradationHits`

都會被連鎖帶偏，後面的 trigger 行為就不適合直接解讀。

### 7. Subscription creation does not immediately produce usable data

`0513-13` 的 testbed log 另外揭露了一個重要行為：

- subscription 建立後
- pseudo-driver 不會立刻讓 NWDAF 看到 phase1/phase2 的可用資料
- 而是要等 first URR 到達、timeline anchor 完成、phase1 history flush 後，
  NWDAF 才開始看到第一批 usable aggregated slots / inference

從 `nwdaf.log`、`UPF-EES/upf.log` 對時後，可量到三段 delay：

- `subscription created -> first URR`: 約 `4s`
- `subscription created -> phase1 history notify success`: 約 `37s`
- `subscription created -> first usable aggregated slot / first ML inference`: 約 `60s`

這表示 testbed 的第一個有效 monitor/policy window，不是從 subscription 建立當下開始算。

因此 replay 需要額外支援一個「subscription 建立到第一批可用資料」的啟動偏移，
否則第一個 monitor round 很容易比 testbed 早一拍，提早消耗第一批成熟 prediction。

目前 replay 已新增可調參數：

- `dataset.subscription_to_first_urr_delay_sec`
- 相容 alias：`dataset.first_urr_delay_sec`

它的作用是：

- 不改 actual slot 與 prediction 數值本身
- 只調整 replay monitor timeline 的起點
- 讓 warmup、mature prediction consume、第一個 policy round 的時機
  更接近 testbed

目前快速驗證結果：

- `delay=4`
  - 幾乎無法修正第一個 policy round 的偏移
- `delay=30`
  - 可讓 `group1` 前三個 policy rounds 幾乎和 testbed 一致

因此目前 `0513-13` 對齊的推薦起始值是：

- `dataset.subscription_to_first_urr_delay_sec: 30`

補充：目前已另外實作 per-group delay 支援（`dataset.subscription_to_first_urr_delay_sec_by_group`），並以 `group1=30`、`group2=30..35` 做快速掃描。結果顯示：

- 它會改變各 group `policy` row 的 `timestamp`
- 但在目前 replay monitor/pairing 邏輯下，不會改變 `group2` 前幾輪的 `actualTrafficScale` / `predictedTrafficScale` 數值
- 因此 `group2` 目前殘留差異，暫時不像是單純由「group2 比 group1 晚幾秒開始」造成

這代表 per-group delay 支援仍然值得保留，因為它符合 testbed startup timing 的結構；但從 `0513-13` 現況看，它不是修正 `group2` early policy 殘差的主要槓桿。

後續進一步排查後已確認：

- `group2` early policy 殘差的主因不是 delay 本身
- 而是 replay 在同一個 `slot_end` 上的 monitor 觸發順序
- 修正為 same-slot-end batch 後，`group2` 也已能貼近 testbed

因此：

- **全域 `30s` delay** 仍是目前推薦的對齊基準
- **per-group delay** 先保留成 feature，但目前不是必要前提

這個值不應被理解成「first URR 本身就在 30 秒後才出現」，
而應理解成一個吸收以下因素的 **effective startup delay**：

- first URR
- phase1 history flush
- first usable monitor window

## Target behavior

下一版 replay 在資料與 prediction 路徑上應儘量符合下面語意：

1. **actual aggregation 保持目前已完成的對齊方式**
2. prediction 前的 historical input 應改成：
   - 依照 testbed `alignAndZipInMemory()` 語意重建
3. prediction target 應改成：
   - `snappedNow` 對齊
   - 而不是 `slot_end`
4. matched pairs 應改成：
   - `TargetTime ± samplingInterval` 內 nearest ground truth
5. accuracy monitor 應補齊：
   - startup warmup
   - warmup prediction discard
   - mature prediction consumption
6. policy 評估仍沿用現有公式
   - 但輸入報告的形成方式要貼齊 testbed

補充：

- 這一輪的成功標準，不再是「retrain 時間接近」
- 而是先讓第一個 policy round 的：
  - `actualTrafficScale`
  - `predictedTrafficScale`
  - `current`
  - `mean/std`
  - `zscore`
  - `degradationHits`
 盡量對齊 testbed

換句話說，下一步目標不是再修 actual slots，
而是讓：

- **prediction rows**
- **monitor rows**
- **policy rows**

也盡量收斂到 testbed 同樣的行為。

## Design principles

### 1. Separate “UE notify generation” from “group actual slot formation”

目前 `SlotObservation` 直接代表 group total。  
新的設計應拆成兩層：

1. **UE-level notification rows**
2. **group-level aggregated slots**

這樣 replay 才能明確表達：

- 哪些資料本來是 per-UE
- 哪些資料是 NWDAF 聚出來的 group total

### 2. Mirror behavior first, optimize structure second

這一輪不應先追求 replay 內部架構簡潔，
而應先追求：

- 行為嚴格貼近 testbed

所以若需要在 replay 內顯式加入：

- `snappedNow`
- warmup discard
- nearest ground truth lookup
- mature prediction queue

應優先做，之後再考慮整理結構。

### 3. Make phase1 / warmup behavior explicit

testbed 實際上：

- phase1 `900s` 主要是 pseudo-driver warm-start history
- 並不是正式 prediction / policy 評估期間

因此 replay 裡也應明確區分：

- `phase1 preload history`
- `phase2 live actual slots`
- `accuracy monitor startup warmup`

不要再把：

- phase1 preload
- prediction warmup discard
- 正式 live evaluation

混在同一個簡化流程裡解讀。

## Proposed replay architecture changes

### A. Replace `load_group_slots()` with a two-stage builder

目前：

- `load_group_slots(group_dir, ...) -> GroupReplayData(slots=[group-level SlotObservation...])`

建議改成兩階段：

#### Stage A1. Build UE-level rebased windows

新增一層結構，例如：

- `UeWindowObservation`

至少包含：

- `group_id`
- `ue_ip`
- `wIdx`
- `slot_start`
- `slot_end`
- `ul_vol`
- `dl_vol`
- `ul_pkts`
- `dl_pkts`
- `notification_doc`

這一層的 slotting 規則要和 pseudo-driver 一致：

- 對每個 `ue_ip` 各自找 `min_ts`
- `wIdx = floor((ts - min_ts) / report_period_sec)`

#### Stage A2. Aggregate UE windows into group slots

再新增一個 group aggregation stage：

- 把同一 `group_id`、同一 `wIdx` 的三個 `UeWindowObservation`
- 聚成一個 group-level `SlotObservation`

這一層才是 replay 後續 prediction / monitor 實際會吃的 actual slot。

這樣就能明確保留：

- per-UE staggered 起點
- testbed 式「先 per-UE，再 group aggregate」語意

### B. Generate per-UE notification docs instead of synthetic `0.0.0.0`

目前 `build_notification_doc()` 會寫：

- `ueIpv4Addr = "0.0.0.0"`

這是 group-level synthetic doc，和 testbed 不同。

建議：

- 新增 `build_ue_notification_doc(...)`
- doc 內使用真實 `ue_ip`

而 group-level `SlotObservation` 不再直接對應一份 synthetic UPF doc。  
它應該是 replay 內部聚合後的結果，不應假裝它是 testbed 原生 UPF notify。

### C. Preserve phase1 / phase2 split at the UE level first

breaking time / aligned breaking time 的切分應該先作用在：

- `UeWindowObservation`

再聚成 group-level：

- `preload_ue_windows`
- `live_ue_windows`

之後：

- phase1 的 `preload_ue_windows`
  - 先聚成 `preload_group_slots`
  - 用於 preload history
- phase2 的 `live_ue_windows`
  - 聚成 `live_group_slots`
  - 用於正式 replay run

這樣可避免先 group aggregate 再切 phase，導致 phase boundary 的語意再次和 testbed 不一致。

### D. Add optional diagnostics output for alignment

為了方便直接和 `0513-13` 對帳，建議 replay 額外輸出：

1. `ue_windows.parquet`
   - per-UE rebased windows
2. `group_slots_rebased.parquet`
   - 由 per-UE 聚合出的 group slots
3. `alignment_meta.json`
   - 記錄：
     - 每個 UE 的 `min_ts`
     - 每個 group 的 `breaking_time`
     - `report_period_sec`

這樣之後比對 testbed 時就不需要每次重新從 code 推導語意。

## Detailed change scope

### File 1. `retrain_replay.py`

預計主要改動區：

- `build_notification_doc()`
- `load_group_slots()`
- `run_command()` 中 preload/live 資料準備流程
- `ReplayEngine.preload_history()`
- `ReplayEngine.run()`

下一輪重點已不只是 actual slots，
而是要調整以下幾層：

1. prediction input window materialization
2. prediction target time
3. mature prediction storage / consumption
4. ground-truth lookup
5. warmup discard behavior

### File 2. Possibly new helper structures

建議在同檔內先新增 dataclass：

- `UeWindowObservation`
- `PendingPrediction` 或等價 helper
- `GroundTruthMatcher` 或等價 helper

actual aggregation 已經就定位，
這一輪更需要的是幫 replay 顯式表達：

- snapped prediction step
- mature prediction queue
- nearest actual matching

### File 3. Optional report / validation helper

視需要可新增一支專用驗證腳本，用來比：

- replay `ue_windows`
- testbed `UPF VOLUME`

但這不一定要和 replay 主程式同一輪一起做。

## Validation plan

### Validation target

第一輪對齊驗證以：

- `5G_Infrastructure/.agent/tmp/0513-13`

作為唯一基準。

原因：

- 它已經證明 pseudo-only path 本身數值正確
- kernel bytes = 0
- 比較乾淨，容易把問題收斂到 slotting / aggregation

### Validation steps

#### Step 1. UE-level parity

用 replay 產出的 `ue_windows` 去比 `0513-13` 的 per-IP `UPF VOLUME`：

- 應先做到：
  - 每個 UE / `wIdx`
  - `ul/dl/total`
  - 完全一致

這一步若過不了，說明 replay 的 per-UE rebasing 仍未對齊 pseudo-driver。

#### Step 2. Group-level parity

再把 replay 內聚出的 `group_slots_rebased`
去比 testbed 真正的 group aggregate：

- 也就是把 `0513-13` 的三個 IP 按同 group、同 `wIdx` 相加後的結果

目標應該是：

- `group1`
- `group2`

在 phase2 `wIdx >= 30` 上逐 slot exact match 或極接近。

#### Step 3. Prediction parity

固定同一個 group、同一批 actual slots，
直接比：

- testbed `predictedTrafficScale`
- replay `predictedTrafficScale`

並找出第一個分叉點。

至少要檢查：

- first monitor round
- baseline ready 前幾輪
- first `degradationSignal=true` 的位置

目前已知：

- first monitor round 前的 inference 幾乎一致
- first monitor round 本身就是第一個主要分岔點

因此這一步的優先順序應高於直接看後面 retrain 是否發生。

#### Step 4. Monitor / policy parity

在 prediction path 對齊後，再比：

- `current`
- `mean`
- `std`
- `zscore`
- `degradationHits`
- `baselineReady`

目標不是一開始就要求所有 rounding 完全一致，
但至少要讓：

- signal 出現時機
- hit accumulation
- retrain trigger

往 testbed 收斂。

#### Step 5. phase1 and warmup behavior

確認 replay 內 phase1 `900s` 的資料只用於 preload history：

- 不進正式 actual monitor round
- warmup prediction 會被丟棄
- 不影響正式 baseline 建立

#### Step 6. exp46 regression baseline

完成新設計後，再回頭和當前 `exp46` 對照：

- 比較舊版 group-level slotting
- 與新版 per-UE -> group aggregation

在：

- `slots.parquet`
- `monitor_rounds.parquet`
- `policy.parquet`

上的差異

這一步不是要保證和 `exp46` 完全一致，而是要明確知道：

- 哪些行為改變是因為資料語意終於對齊 testbed

## Non-goals for the first implementation

第一輪不建議一起做：

1. 不改 Daisy / retrain 參數
2. 不改 prediction model 本身
3. 不改 trigger policy 公式
4. 不改 NWDAF Go runtime
5. 不把 replay 改成真的「先 per-UE HTTP notify，再由 Python collector 重跑一遍」

因為目前最需要先修的是：

- **prediction / monitor 行為對齊**

不是再去調模型或 policy 本身。

## Risks

### Risk 1. Replay outputs will no longer match old experiments

一旦 slotting 從 group-level 改成 per-UE -> group aggregate：

- `exp35` ~ `exp47` 的 actual traffic 底層就不再可直接和新版重跑比

這是預期中的破壞性差異，不應當成 regression bug。

### Risk 2. Prediction target parity may require replaying snapped clock explicitly

testbed 的 prediction 不是「每來一個 slot 就預測這個 slot」，
而是：

- 以 `snappedNow` 為時間原點
- 把最近 input window 對齊到 absolute timeline
- 再生成 step-based prediction

replay 若只在 live slot arrival 時做 predict，
即使 actual slots 一致，
prediction state 也可能永遠無法完全重現。

### Risk 3. Group names differ from testbed

replay 現在常用：

- `group1`
- `group2`

testbed log 常見：

- `group-test-001`
- `group-test-002`

這不是資料語意問題，但驗證工具層要明確做 mapping，不然容易混淆。

## Implementation order

建議分三步：

1. 保持現有 actual aggregation，不再重寫這一層
2. 先補 replay 版 `snappedNow + mature prediction + nearest GT` 基礎設施
3. 再對齊 warmup discard 與 monitor cadence
4. 先驗第一個 `policy` round 是否對齊
5. 只有在第一個 `policy` round 對齊後，才重看後續 `policy.parquet` / retrain trigger 是否收斂

不要一開始就連 replay engine 核心也一起大改。

## Expected outcome

這輪完成後，replay 應該能做到：

1. actual aggregation 已與 testbed 一致
2. replay prediction / monitor / policy 行為會更接近 `0513-13`
3. 之後比較 replay 與 testbed 的 retrain 行為時，
   不再先被 prediction target / GT matching / warmup 差異污染

最終目標不是讓 replay 只是「看起來差不多」，而是讓它至少在：

- per-UE rebasing
- group aggregation
- snapped prediction target
- mature prediction consumption
- nearest ground-truth matching
- accuracy warmup discard

這些關鍵點上，和 testbed 使用同一種邏輯。
