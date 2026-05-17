# NWDAF retrain_replay testbed parity refactor plan

## Purpose

這份文件定義 `5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py`
下一輪重構計畫，目標是讓 replay 在 **prediction / monitor / policy / first-trigger**
這條線上，近乎完全貼齊 testbed 目前實際使用的 NWDAF 行為。

這份計畫的直接背景是：

- `0514-22` 對齊工作已證明
  - actual aggregation 已基本對齊
  - inference target slot 也已高度接近
- 但 `policy.parquet` 仍與 testbed 有系統性差異
- 進一步對 git 與主程式比對後，已確認 testbed 在最近 commit 中改變了：
  - target-slot 定義
  - pred/actual pairing
  - pending prediction mature / retry / discard 生命周期

這份文件處理：

1. `retrain_replay.py` 與 testbed 現行 Go 行為的差異盤點
2. replay 重構的目標語義
3. 建議的實作拆分順序
4. 驗證與完成標準

這份文件不處理：

- Daisy 訓練流程本身的精細 wall-clock 模擬
- testbed Go 程式碼修改
- 模型架構、權重、scaler 設計

---

## Progress tracker

| Area | Status | Notes |
| --- | --- | --- |
| Aggregation alignment baseline | done | `0514-22` / `exp64` 已確認 `slots.parquet` 基本對齊 |
| Inference target-slot baseline | done | 早期 target slot 已大致收斂，主問題已不在這裡 |
| Testbed current behavior audit | done | 已確認 `e8c8978` 後的 target/pairing/pending lifecycle 語義 |
| Replay mismatch inventory | done | 已整理 nearest match、warmup discard、pending lifecycle、startup timing 等差異 |
| Startup timing parity design | done | 已補 `subscription -> first URR -> anchor -> first inference` 設計 |
| Workstream A. Prediction timing refactor | done | 已完成目前 `0514-22` pre-trigger parity 需要的切片；剩餘 prediction residual 改列已知潛在問題 |
| Workstream B. Pending store lifecycle refactor | done | pending snapshot / resolve / retry / discard 已落地，足以支撐目前 monitor / policy 對齊目標 |
| Workstream C. Slot-equality pairing refactor | done | replay 已改為 slot-based pairing，nearest lookup 已退出主路徑 |
| Workstream D. Warmup / first-monitor parity | done | warmup discard、first-monitor sample composition、monitor trace 已收斂到目前目標 |
| Workstream E. Startup timing profile refactor | done | 已完成目前 scope 需要的 startup anchoring / visibility / wall-clock 切片 |
| Exp64 replay re-validation after refactor | done | 已完成第一批實作後的 `exp64` 重跑，輸出可穩定供後續 compare 使用 |
| Compare report review | done | compare 已強化 `Accuracy scope` / `Ground truth slot match`，並完成一次新版報表檢視 |
| Current `0514-22` pre-trigger parity target | done | 以 `exp64`、最新 compare 與 shadow probe 判斷，已達到目前可接受的收斂狀態 |

狀態約定：

- `done`: 已完成分析或設計，且已寫入文件
- `pending`: 尚未開始或尚未進入實作
- `in_progress`: 仍在推進，尚未收斂到目前目標

以下各批「實作驗證」段落保留的是歷史推進軌跡，
其中若出現當時的 `in_progress` 判讀，應以前述 `Progress tracker`
與 `Remaining issues by severity` 的現況描述為準。

最新第一批實作驗證：

- compare report:
  - [compare_exp64_after_pending_slot_parity.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_exp64_after_pending_slot_parity.md)
- 目前結果：
  - `slots.parquet` 仍維持 `shift +0`
  - inference surface 仍維持 `shift +0`
  - compare 腳本現已額外 parse testbed `Accuracy scope` 與 `Ground truth slot match`
    - 可直接檢查每輪 monitor 的 `samples / WAPE / MAE`
    - 也可檢查 testbed 實際 matched 的 target slots
    - 並可用 `subId + targetSlotTime -> ML inference` 回推出該輪實際使用的 prediction totals
  - pending lifecycle / warmup discard / slot pairing trace 已可觀測
  - `Monitor Surface` 已顯示 early monitor 才是目前的主要偏差源
    - `group:group1` 最佳 shift 為 `-2`
    - `group:group2` 最佳 shift 為 `-1`
  - first trigger 仍落在 `group:group2`
  - 目前最值得優先處理的殘差已收斂到：
    - startup timing / first effective monitor round
    - prediction target-time / prediction selection 語義

最新第二批實作驗證：

- compare report:
  - [compare_exp64_after_startup_timing_slice.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_exp64_after_startup_timing_slice.md)
- 目前結果：
  - 已補入 `StartupTimingProfile`
  - replay 現在會在 batch 前後按時間順序觸發 warmup discard 與 monitor rounds
    - 不再讓 timer events 吃到未來 slot / prediction
    - warmup 也不再延後到第一批資料到達時才丟棄剛生成的 prediction
  - `Monitor Surface` 的 early shift 已明顯收斂：
    - `group:group1` 最佳 shift 由 `-2` 收斂到 `+0`
    - `group:group2` 最佳 shift 由 `-1` 收斂到 `+0`
  - 第一輪 monitor sample count 與 early sample composition 已較接近 testbed
  - 但 replay 目前仍未觸發 retrain，表示主殘差已逐步轉移到：
    - prediction target-time / prediction selection 語義
    - 以及其後續對 policy surface 的影響

最新第三批實作驗證：

- compare report:
  - [compare_exp64_after_workstream_a_slice.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_exp64_after_workstream_a_slice.md)
- 目前結果：
  - `Workstream A` 已落最小切片：
    - replay prediction target 已改成預設對齊 testbed
      `baseTargetTimeFromHistorical(last.Ts) + predictionTargetTime(step)` 語義
    - `slot.slot_end` 現在作為 default `base_target_sim_time`
    - 舊 `prediction_target_offset_sec` 已降為 legacy mode
  - compare 現在已顯示：
    - first trigger 回到 `group:group1`
    - `Monitor Surface` 仍維持 `shift +0`
    - `Policy Surface` 也已收斂到 `shift +0`
  - 但殘差仍存在：
    - 第一輪 monitor sample count 仍是 replay `1` vs testbed `2`
    - 部分 `rp_targetSlots / rp_predTotals` 仍與 testbed 同輪 batch 不同
    - first trigger timing 仍早於 testbed
  - 因此 `Workstream A` 仍屬 `in_progress`
    - 下一步應繼續追 prediction batch selection / first-round composition

最新 trace 發現：

- 第一輪 `replay sampleCount=1` vs `testbed sampleCount=2` 的直接原因已收斂
- 問題不在 slot-equality pairing 本身，而在：
  - replay 目前的 `prediction visible time`
  - 與 `actual slot available time`
  - 兩者相對順序仍和 testbed 不一致
- 在 replay 第一輪 monitor 前：
  - 兩筆 prediction 已經存在
  - 但只有第一筆 target slot 的 actual 已經進到 `actual_slots_by_group_time`
  - 第二筆 actual 要等下一個 batch flush 才會可見
  - 因此形成 `2 pending -> 1 matched + 1 missed`
- 在 testbed 第一輪 monitor 前：
  - 兩筆 prediction 與兩筆 actual 都已經到位
  - 所以第一輪可直接形成 `samples=2`
- 這表示目前 `Workstream A` 尚未對齊的重點，已明確縮到：
  - `predicted_at_sim_time` / prediction visibility lag 的建模
  - 而不是 target slot identity 或 pairing rule 本身

最新第四批實作驗證：

- compare report:
  - [compare_exp64_after_prediction_visibility_slice.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_exp64_after_prediction_visibility_slice.md)
- 目前結果：
  - replay 已新增 delayed prediction activation
    - prediction 不再在 slot flush 當下立刻進 `prediction_store`
    - 而是先 schedule，等 `predictedAtSimTime` 到達後才進入 pending store
  - `StartupTimingProfile` 現在已明確帶出：
    - `firstVisibleInferenceEpoch`
    - `predictionVisibilityLagSec`
  - 第一輪 monitor 已回到：
    - `pending 2 -> matched 2, missed 0`
    - `sampleCount=2`
  - `Monitor Surface` 已高度貼近 testbed：
    - `group:group1` best shift `+0`
    - `group:group2` best shift `+0`
    - early rows 的 `WAPE / MAE / targetSlots / predTotals / actualSlots` 已高度接近
  - `Policy Surface` 也已明顯收斂：
    - `actualTrafficScale` 對照已大致一致
    - `group:group1` / `group:group2` 的 early `current / zscore / hits` 已接近 testbed
  - first trigger 方向維持正確：
    - replay first trigger 仍為 `group:group1`
  - 目前剩餘問題已從 monitor/pairing 主問題，進一步收斂成：
    - first trigger 絕對時刻仍早於 testbed
    - 以及少量 pre-trigger surface residuals

最新第五批實作驗證：

- compare report:
  - [compare_exp64_after_startup_epoch_slice_v2.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_exp64_after_startup_epoch_slice_v2.md)
- 目前結果：
  - replay 已把 startup anchoring 從 `firstDataSlotEpoch` 改成明確的 startup wall-clock slice
    - `firstTargetSlotEpoch`
    - `firstUrrSignalEpoch`
    - `anchorEstablishedEpoch`
    - `firstVisibleInferenceEpoch`
  - `exp64.yaml` 也已切到 `startup_timing.mode=explicit_offsets`
    - `subscription_to_first_urr_signal_sec=3`
    - `first_urr_signal_to_anchor_sec=1`
    - `anchor_to_first_visible_inference_sec=56`
  - replay batch 內事件已改成真正依時間點前進：
    - `prediction_emitted`
    - `warmup discard`
    - `monitor round`
    - 不再因為 `batch_end` 而把未來 prediction 提前暴露給較早的 monitor round
  - 第一輪 monitor 仍維持正確：
    - `pending 2 -> matched 2`
    - `sampleCount=2`
  - `Monitor Surface` 與 `Policy Surface` 仍維持：
    - `group:group1` best shift `+0`
    - `group:group2` best shift `+0`
  - 第一輪 monitor timestamp 已從 `00:16:20` 推進到 `00:16:47`
    - 與 testbed `14:27:16` 相對於第一批 target slots 的距離已對齊
  - replay first trigger 也已從 `00:32:50` 推進到 `00:33:17`
    - 與 testbed first trigger 相對於第一輪 policy 的距離仍維持一致
  - 因此原本 `High` 所指的 startup epoch / absolute monitor timeline anchoring 問題，
    - 在 pre-trigger parity 範圍內可視為已基本解決
  - 目前新冒出的後續問題是：
    - replay 在 hot-swap 後又出現第二次 retrain trigger
    - 這屬於 post-trigger / post-swap 行為，尚未對照 testbed 收斂

最新收斂確認：

- shadow inference probe:
  - [shadow_inference_window_probe_pretrigger.md](/home/chingje/testbed/5G_Infrastructure/.agent/shadow_inference_window_probe_pretrigger.md)
- 目前結果：
  - 使用 testbed `nwdaf.log` 中可見的 `latest aggregated slot` 視窗，
    套用與 testbed 相同的 initial bundle 重跑 inference，
    會穩定重現 replay，而不是完全重現 testbed `ML inference`
  - 這表示目前剩餘的 prediction residual，
    並不主要來自 bundle / scaler / ML service inference 實作不一致
  - 更可能來自：
    - testbed 當下實際送入 ML service 的 historical window
    - 與可見 finalized-slot window 不完全相同
  - 但在 `0514-22` pre-trigger parity window 內，
    這類 residual 對 replay / testbed 的影響量級，
    已不足以改變目前的主要 monitor / policy / first-trigger 結論
  - 因此目前可將此問題降級為：
    - 已知潛在問題
    - future refinement
    - 非當前重構 blocker

### Remaining issues by severity

#### High

- 目前在 `pre-trigger parity` 範圍內，沒有新的 confirmed high-severity blocker
  - 原本的 startup epoch / absolute monitor timeline anchoring 問題
    - 已透過 `firstTargetSlotEpoch` anchoring 與 batch 內事件時間序處理，基本收斂
  - 後續若要追更細的 wall-clock parity，
    - 應再確認 `from_logs` mode 與 group-specific startup offsets
    - 但目前不再是主要 blocker

#### Medium

- testbed 實際 inference window 可能不等於可見 finalized-slot window
  - shadow inference probe 顯示：
    - 用可見 `latest aggregated slot` 重建出的 30-slot historical window，
      會幾乎精準重現 replay，而不是 testbed `ML inference`
  - 這表示剩餘 residual 更可能位於：
    - testbed 端真正送入 ML service 的 historical window 組成
    - 而不是 replay bundle / scaler / predictor 實作
  - 目前仍未直接證明其唯一機制就是 raw-report overwrite / mutable reconstruction
  - 但在 `0514-22` pre-trigger 範圍內，
    它目前屬於可接受的已知潛在問題，而非必修修正項

- replay 在 hot-swap 後又出現第二次 retrain trigger
  - 這屬於 post-trigger / post-swap lifecycle 行為
  - 不在目前 `0514-22` parity window 的主要範圍內
  - 目前計畫仍聚焦：
    - `CAT1-only policy window`
    - `early CAT2 pre-trigger window`
    - 也就是 first trigger 之前的 monitor / policy 對齊
  - 因此這一項目前應視為 future work，而不是當前 parity blocker

- `Policy Surface` 的 early rows 仍有小幅數值偏差
  - 主要集中在 `current / predictedTrafficScale / zscore`
  - 雖然整體已 `shift +0`，但還沒到逐列幾乎重合
  - 現階段判斷這屬於可接受 residual，
    不足以破壞目前 first-trigger 前的主要對齊結論

- prediction visibility 目前仍是固定 lag approximation
  - 這次已足夠把第一輪 monitor 修正回來
  - 但仍不是完整重現 NWDAF notifier / analytics 真實節奏

#### Low

- `Aggregated Slot Surface` 在最新版 compare 出現 `shift -2`
  - 目前更像 compare 視角或 startup wall-clock 與 logical slot 軸分離後的表面現象
  - 暫時不像 monitor / policy 那樣影響主結論

- 尚未補 replay-side automated tests
  - 目前主要仍靠 `exp64 + compare report` 做回歸
  - 這是工程完整度問題，但不是現階段 parity 的主要 blocker

---

## Reference baseline

### Main alignment target

- testbed run:
  - `5G_Infrastructure/.agent/tmp/0514-22`
- validation experiment config:
  - [exp64.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp64.yaml)
- baseline report:
  - [nwdaf-replay-testbed-0514-22-offline-comparison-prep.md](/home/chingje/testbed/docs/5g-infra/reports/nwdaf-replay-testbed-0514-22-offline-comparison-prep.md)
- current comparison:
  - [compare_0514_22_vs_exp64.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_0514_22_vs_exp64.md)
- current inference window match example:
  - [match_group1_seq7.md](/home/chingje/testbed/5G_Infrastructure/.agent/match_group1_seq7.md)

### Main code references

- testbed entry:
  - [cmd/main.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/cmd/main.go:20)
- service bootstrap:
  - [pkg/service/init.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/pkg/service/init.go:37)
- UPF callback ingestion:
  - [internal/sbi/processor/upf_notify.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/sbi/processor/upf_notify.go:118)
- inference path:
  - [internal/anlf/analytics.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/analytics.go:104)
- monitor path:
  - [internal/anlf/monitor.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/monitor.go:16)
- pending prediction store:
  - [internal/context/accuracy_store.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/context/accuracy_store.go:10)
- policy path:
  - [internal/mtlf/trigger.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/trigger.go:44)
- replay tool:
  - [tools/retrain_replay/retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py:1)
- comparison tool:
  - [compare_nwdaf_replay_testbed.py](/home/chingje/testbed/5G_Infrastructure/.agent/compare_nwdaf_replay_testbed.py)

### Key commits

- testbed behavior change:
  - `e8c8978` `fix(anlf): align ue communication target slots`
- replay-side timing follow-up:
  - `dc36ecc` `Align replay monitor timing with testbed`
- older policy baseline fix already mirrored on both sides:
  - `ecd6ce4` `fix(mtlf): fix catch-22 in degradation reference baseline building`

---

## Current understanding

### What is already considered aligned

截至目前，這條線可以先視為已收斂的部分：

1. group-level aggregated actual slots
2. early inference target-slot time axis
3. initial bundle / scaler baseline assumption
4. `policy` 公式本身的大框架

也就是：

- 問題不再主要是 actual aggregation
- 問題也不再主要是 replay 有沒有吃到同一段 history
- 目前最值得投入的面，是 replay 對 testbed 現行 `prediction -> monitor -> policy`
  生命周期的模擬仍不完整

### What is now believed to be the main mismatch source

根據 `e8c8978` 與目前 replay 的對照，主要差異已收斂到以下幾項：

1. prediction target time 的語義不同
2. warmup 後 prediction 清理規則不同
3. pred/actual pairing 規則不同
4. pending prediction 的 retry / discard 生命周期不同
5. monitor round 對 sampleCount / inferenceNum / windowStart / windowEnd 的形成方式不同

這些差異會直接改變：

- 每輪被 monitor 消費到的 pair 集合
- `actualTrafficScale`
- `predictedTrafficScale`
- `current`
- `mean`
- `std`
- `zscore`
- `degradationHits`
- first trigger group / timing

所以即使 replay 的 `evaluate_policy()` 數學公式看起來與 testbed 接近，
只要前面 pair 集合不同，`policy.parquet` 仍會持續偏掉。

---

## Confirmed behavior in testbed

本節只記錄已由 git 與目前工作樹確認的 testbed 行為，不再混用舊版假設。

### 1. Prediction target is derived from the last historical slot

在 [internal/anlf/analytics.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/analytics.go:137)：

- testbed 先取 `historicalData[len-1]`
- 再用 `baseTargetTimeFromHistorical(last.Ts, samplingInterval)`
- `step=0` target = `lastHistoricalSlot + samplingInterval`

這表示：

- target 不是單純 `predicted_at`
- target 也不是直接取 `snappedNow`
- replay 若仍以 `slot_end + offset` 近似，而沒有明確表達
  「最後 historical slot 的下一格」，就可能在細節上與 testbed 不同

### 2. Prediction records now carry both semantic target and slot target

testbed 的 `PredictionRecord` 現在有：

- `TargetTime`
- `TargetSlotTime`
- `MissCount`
- `ID`

位置在 [internal/context/accuracy_store.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/context/accuracy_store.go:12)。

這代表：

- replay 不能再把 prediction 視為「生成後到時就消費掉」的短生命物件
- prediction 現在是具備 pending 狀態與跨 monitor round 重試能力的狀態物件

### 3. Warmup 結束後，testbed 直接丟棄所有 pending predictions

在 [internal/anlf/monitor.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/monitor.go:83)：

- warmup 結束後執行 `DiscardAllPredictions()`
- 同時 `GetAndResetInferenceNum()`

這和較早版本的：

- `ConsumeMaturePredictions(0)`

已不同。

影響是：

- warmup 期間產生但尚未 mature 的 prediction
  在 testbed 也不會被保留進正式 baseline
- replay 如果只丟掉 warmup end 前已 matured 的 prediction，
  就會比 testbed 多保留一批 pre-baseline pending predictions

### 4. Monitor rounds now scan all pending predictions and retry misses

在 [internal/anlf/monitor.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/monitor.go:127)：

- 每輪 monitor 先 `SnapshotPredictions()`
- 每筆 prediction 嘗試 lookup actual
- matched 的 prediction 會被移除
- missed 的 prediction 會 `MissCount += 1`
- `MissCount` 超過 `predictionMaxMissCount()` 才被丟棄

這和舊版：

- 只吃 `ConsumeMaturePredictions(2*samplingInterval)`
- 這輪沒配到就自然消失

完全不同。

影響是：

- testbed policy round 所看到的 matched pair 集合
  可能包含前一兩輪尚未成功配對、但這一輪才補配上的 prediction
- replay 若沒有這個機制，`sampleCount`、`windowStart/windowEnd`、
  `actualTrafficScale`、`predictedTrafficScale` 都可能偏掉

### 5. Ground-truth matching changed from nearest-match to slot-equality pairing

在 [internal/anlf/monitor.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/monitor.go:255)：

- 先以 `TargetSlotTime` 決定 slot identity
- Mongo/in-memory actual 都要先映射到 slot key
- 只有 slot key 完全相同才算 match

這已不是舊版的：

- `TargetTime ± samplingInterval` 內 nearest

因此 replay 若仍以 nearest match 模擬，就不是「現行 testbed」行為，
而是「`e8c8978` 前的舊版」行為。

### 6. Startup timing in testbed is event-driven, not a single fixed delay

目前已確認 testbed 的 startup 不是：

- 訂閱建立
- 等固定 `N` 秒
- monitor / inference / policy 一起開始

而是至少有三條彼此鬆耦合的時間線：

1. subscription / model-init timeline
2. UPF pseudo-driver anchor timeline
3. first-usable-data / first-visible-inference timeline

在 [internal/notifier/notifier.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/notifier/notifier.go:101)：

- notification scheduler 在 subscription 建立後立刻啟動
- 第一筆 notification 也是立即送出

在 [internal/anlf/monitor.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/anlf/monitor.go:83)：

- accuracy monitor 在 model initialize 後立刻進入 warmup
- warmup 完成後直接開始固定 `checkInterval` monitor round

但在 [internal/ees/pseudodriver.go](/home/chingje/testbed/5G_Infrastructure/go-upf-ess/go-upf/internal/ees/pseudodriver.go:238)：

- pseudo-driver 會先等待第一個 kernel URR
- 再用該時間建立 `anchorTime` / `GridAnchor`
- 之後才執行 Phase 1 warmstart 與 Phase 2 future simulation

而 [internal/ees/aggregator.go](/home/chingje/testbed/5G_Infrastructure/go-upf-ess/go-upf/internal/ees/aggregator.go:480)
目前是用 `time.Now()` 去 signal 第一個 URR 到達。

因此 testbed 的真實語義是：

- monitor timeline 比資料更早起跑
- UPF data timeline 會晚於 subscription start
- first visible inference 會再晚於 first URR / anchor established

這一點會直接影響：

- warmup 完成時 store 內有沒有 prediction
- 第一輪 monitor round 看到多少 pending predictions
- delayed match 何時開始出現
- first policy row 的 `sampleCount / inferenceNum / trafficScale`

### 7. Policy path itself is not the newly changed part

在 [internal/mtlf/trigger.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/trigger.go:48)：

- `current/mean/std/zscore`
- `degradationBaselineReady`
- `recentBaselineReady`
- `degradationHits`
- `predictedTrafficScale`
- `degradation reference baseline` 更新規則

目前看起來仍沿用既有框架。

因此這輪重構的重點不是先重寫 `evaluate_policy()`，
而是先讓 replay 送進 policy 的 `AccuracyReport` 更接近 testbed 真正產出的 report。

---

## Current replay behavior that still diverges

以下差異是目前已確認、且足以解釋 `policy.parquet` 持續偏掉的地方。

### 1. Replay still uses nearest ground-truth lookup

目前 [retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py:1033)
是 `lookup_ground_truth_nearest()`。

這與 testbed 現在的 slot-equality pairing 不同。

### 2. Replay still consumes only mature-and-already-matched predictions

目前 [retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py:1083)
的 `consume_mature_predictions()`：

- 只看 `matured_at_sim_time <= round_time`
- 只要這輪當下找得到 actual 才消費
- 找不到 actual 的 prediction 不會有完整 `MissCount` 生命周期

這與 testbed 現在的 pending snapshot / resolve / retry 模型不同。

### 3. Replay warmup discard is still narrower than testbed

目前 [retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py:1053)
只丟棄：

- warmup end 前已 matured 的 prediction

但 testbed 現在是丟掉：

- warmup 結束時 store 裡的全部 pending predictions

### 4. Replay target-time semantics are still expressed as `slot_end + offset`

目前 [retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py:933)
是：

- `predicted_at = slot.slot_end`
- `target_time = slot.slot_end + prediction_target_offset_sec`

這種表達雖然在某些 case 上可能數值接近，
但它沒有明確反映 testbed 的真正語義：

- target = `last historical slot + samplingInterval`

這會讓 replay 後續難以和 testbed 一樣處理：

- hidden history
- preloaded phase1
- monitor pairing trace

### 5. Replay monitor state object still lacks pending-record lifecycle parity

目前 replay 有：

- `PredictionRecord.prediction_id`
- `consumed`

但沒有對應到 testbed 的：

- `ID`
- `MissCount`
- `ResolvePredictions(matchedIDs, missedIDs, maxMissCount)`

結果是 replay 難以精準輸出：

- 本輪 matched 幾筆
- 本輪 missed 幾筆
- 本輪 discarded 幾筆
- 哪些 pair 是 delayed match

這些資訊目前對 debug `monitor_rounds.parquet` 很重要。

### 6. Replay still treats startup timing as a single scalar offset

目前 [retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py:1543)
對每個 group 的 schedule 是：

- `monitor_start = first_slot.slot_start - group_delay`
- `warmup_end = monitor_start + startup_warmup`
- `next_monitor = warmup_end + check_interval`

而 `group_delay` 主要來自：

- `dataset.subscription_to_first_urr_delay_sec`
- `dataset.subscription_to_first_urr_delay_sec_by_group`

這種做法只是在：

- 用一個固定秒數回推 monitor 起點

它沒有明確模擬：

1. subscription 建立時間
2. first URR signal 時間
3. anchor established 時間
4. phase1 warmstart 完成時間
5. first usable slot / first visible inference 時間

因此這個參數目前的實際角色，比較接近：

- `subscription_to_startup_data_gap_approximation`

而不是字面上的：

- `subscription_to_first_urr_delay`

---

## Refactor objective

### Main goal

把 replay 的 monitor pipeline 重構成：

- 在語義上盡量複製 testbed `e8c8978` 後的現行 NWDAF
- 不是只做「看起來相近」的近似

### Success criteria

這一輪的完成標準依優先序定義如下：

1. `slots.parquet`
   - 持續維持與 testbed 對齊
2. `predictions.parquet`
   - target slot 仍與 testbed 在同一條 time axis
3. `monitor_rounds.parquet`
   - `sampleCount`
   - `inferenceNum`
   - `windowStart`
   - `windowEnd`
   在 `CAT1-only` 與 `early-CAT2 pre-trigger` 視窗中顯著收斂
4. `policy.parquet`
   - `current`
   - `mean`
   - `std`
   - `zscore`
   - `actualTrafficScale`
   - `predictedTrafficScale`
   - `degradationHits`
   在同一視窗內顯著收斂
5. first trigger
   - group
   - timing
   - hit progression
   與 testbed 一致或只剩極小殘差

### Non-goals for this refactor

以下不作為這輪的必要完成標準：

- Daisy pending 期間 wall-clock 完全一致
- hot-swap 之後每一輪 policy 全部 bitwise 相同
- 所有模型預測值 bitwise identical

---

## Startup timing parity design

本節專門處理 `subscription_to_first_urr_delay_sec` 目前承擔過多語義的問題。

### Design goal

把 replay 的 startup 行為從：

- 單一 fixed offset 的 schedule approximation

改成：

- 接近 testbed 的 event-driven startup timeline

### Why this needs to be separate

`0514-22` 已顯示這幾個時間點不是同一件事：

- NWDAF subscription created:
  - `2026-05-14T14:25:26.091Z`
- UPF first URR signal broadcasted:
  - `2026-05-14T14:25:29.053Z`
- pseudo-driver anchor established:
  - `2026-05-14T14:25:33.539Z`
- NWDAF accuracy monitor warmup complete:
  - `2026-05-14T14:25:46.804Z`
- first visible aggregated slot / ML inference:
  - `2026-05-14T14:26:26.096Z`
  - `2026-05-14T14:26:26.133Z`
- first monitor round:
  - `2026-05-14T14:27:16.813Z`

這組時間線意味著：

- 真實 `subscription -> first URR` 大約是 `3s`
- 真實 `subscription -> anchor established` 大約是 `7s`
- 真實 `subscription -> first visible inference` 接近 `60s`

因此目前 config 中的 `30s` 並不對應單一真實事件。

### Canonical replay startup model

replay 應改成顯式維護每個 group 的 `StartupTimingProfile`。

建議欄位：

- `subscription_epoch`
- `first_urr_signal_epoch`
- `anchor_established_epoch`
- `first_data_slot_epoch`
- `first_visible_inference_epoch`
- `warmup_end_epoch`
- `first_monitor_epoch`

它們的語義如下。

#### `subscription_epoch`

- 對應 testbed 的 subscription 建立完成，且 NWDAF notification scheduler 已可開始工作
- 也是 accuracy monitor / warmup 的起點

#### `first_urr_signal_epoch`

- 對應 UPF pseudo-driver 收到第一個 kernel URR signal 的時間
- 這是 data timeline 的第一個外部觸發點

#### `anchor_established_epoch`

- 對應 pseudo-driver 完成 `anchorTime / GridAnchor` 建立的時間
- 之後 Phase 1 / Phase 2 才能在正確的 time grid 上推進

#### `first_data_slot_epoch`

- 對應 replay 中第一個能被 NWDAF 視為有效 slot 的 sim-time
- 它不一定等於 `first_urr_signal_epoch`
- 也不應強迫等於 `subscription_epoch + constant`

#### `first_visible_inference_epoch`

- 對應 testbed 第一個 visible aggregated slot / ML inference 出現的時間
- 這通常晚於 first URR，也晚於 anchor established
- 它會受到 input window 與 hidden history / phase1 preload 影響

#### `warmup_end_epoch`

- `subscription_epoch + warmupDuration`
- 不依賴 data timeline

#### `first_monitor_epoch`

- `warmup_end_epoch + checkInterval`
- 不依賴 first visible inference 是否已出現

### Replay semantics under the new model

新的 replay 應遵循以下規則：

1. monitor timeline 與 data timeline 分離
   - warmup 與 monitor round 由 `subscription_epoch` 決定
   - slot push / prediction 生成由 `first_data_slot_epoch` 後的資料到達決定
2. monitor round 允許早於第一個 prediction 出現
   - 若當輪沒有 pending predictions，仍應記錄 empty round 或至少保留其語義
3. warmup discard 只看 `warmup_end_epoch`
   - 到點就丟棄當下全部 pending predictions
   - 不應依賴「第一個 slot 是否已經到達」
4. policy baseline accumulation 跟著 monitor round 前進
   - 不應被單一 startup offset 重新對齊成資料驅動

### Config design

建議把現有 `subscription_to_first_urr_delay_sec` 降級為 legacy compatibility。

新的 config 建議改成：

```yaml
replay:
  startup_timing:
    mode: explicit_offsets
    subscription_to_first_urr_signal_sec: 3
    first_urr_signal_to_anchor_sec: 4
    anchor_to_first_visible_inference_sec: 52
```

或在有 log 可用時：

```yaml
replay:
  startup_timing:
    mode: from_logs
    nwdaf_log: 5G_Infrastructure/.agent/tmp/0514-22/nwdaf.log
    upf_logs:
      group-test-001: 5G_Infrastructure/.agent/tmp/0514-22/upf-ees/upf.log
      group-test-002: 5G_Infrastructure/.agent/tmp/0514-22/upf-ees2/upf.log
```

建議支援三種 mode：

1. `from_logs`
   - 直接從 NWDAF / UPF log 萃取 startup profile
   - 作為 parity work 的首選
2. `explicit_offsets`
   - 沒有 log 時，用多段 offset 明確指定
3. `legacy_single_offset`
   - 保留舊 `subscription_to_first_urr_delay_sec`
   - 但只作 backward compatibility

### Derivation rules for each mode

#### `from_logs`

建議從 log 直接抓：

- `subscription_epoch`
  - NWDAF `Subscription created` 或 `Notification scheduler started`
- `first_urr_signal_epoch`
  - UPF `first URR signal broadcasted`
- `anchor_established_epoch`
  - UPF `grid alignment established`
- `first_visible_inference_epoch`
  - NWDAF 第一筆 `ML inference`

再推導：

- `warmup_end_epoch = subscription_epoch + warmupDuration`
- `first_monitor_epoch = warmup_end_epoch + checkInterval`

`first_data_slot_epoch` 可先採：

- 第一筆 replay slot 的 `slot_start`

若後續需要更嚴格 parity，再補成從 log 萃取第一筆可見 slot。

#### `explicit_offsets`

若缺少 log，則以：

- `subscription_epoch = first_visible_inference_epoch - subscription_to_first_visible_inference_sec`

或等價的多段 offset 推導 profile。

但即使在 explicit mode，也應維持多段語義，而不是退化回單一 scalar。

#### `legacy_single_offset`

只保留舊邏輯：

- `monitor_start = first_slot.slot_start - legacy_offset`

但文件與 trace 必須明確標示：

- 這不是 testbed parity mode
- 只是一個 coarse approximation

### Implementation impact

這個設計會影響 replay 的三個面向。

#### 1. Schedule initialization

目前的：

- `monitor_start = first_slot.slot_start - group_delay`

應改為：

- 先建立 `StartupTimingProfile`
- 再由 `subscription_epoch` 算出 `warmup_end_epoch / first_monitor_epoch`

#### 2. Warmup discard behavior

warmup discard 需要依賴：

- `warmup_end_epoch`

而不是依賴：

- 某個 batch 是否已經到達第一個 slot

#### 3. Trace and compare surface

replay output 應新增 startup trace，至少包含：

- `subscriptionEpoch`
- `firstUrrSignalEpoch`
- `anchorEstablishedEpoch`
- `firstDataSlotEpoch`
- `firstVisibleInferenceEpoch`
- `warmupEndEpoch`
- `firstMonitorEpoch`
- `timingMode`

### Backward compatibility policy

短期內可以保留 `subscription_to_first_urr_delay_sec`，但必須：

1. 改名或在文件中重新定位
   - 建議定位為 `legacy startup offset`
2. 不再作為 parity mode 預設
3. compare report 中顯式印出：
   - 目前使用的是 `from_logs / explicit_offsets / legacy_single_offset`

### Validation plan for startup timing

這一段要單獨驗證，不應只看最終 trigger。

每次修改後，除了既有 `exp64.yaml` + `compare_nwdaf_replay_testbed.py` 外，
還應額外檢查：

1. startup profile 是否與 `0514-22` log 對上
   - subscription
   - first URR
   - anchor established
   - first inference
   - first monitor
2. warmup 完成時，replay store 內 pending prediction 狀態是否合理
3. 第一輪 monitor 是否發生在與 testbed 同一條 monitor timeline 上
4. 第一個 policy row 是否不再因 startup offset 粗估而偏移一整輪

### Acceptance criteria for this sub-problem

若 startup timing parity 完成，至少應看到：

- replay 不再需要用單一 `30s` 去硬調 first monitor timing
- 第一輪 monitor 與第一個 inference 的相對順序能和 testbed 一致
- warmup discard 與 early pending prediction 組成更接近 testbed
- 後續 `policy.parquet` 的 early-row 差異顯著縮小

---

## Refactor strategy

建議拆成五個 workstreams，按順序執行，避免一次混太多變數。

### Workstream A. 明確重建 testbed-style prediction timing

#### Goal

讓 replay 的 prediction record 明確對應 testbed 現在的：

- `PredictedAt`
- `TargetTime`
- `TargetSlotTime`

#### Planned changes

1. 在 replay 內把目前的 prediction 生成改成兩層概念：
   - `predicted_at_sim_time`
   - `base_target_sim_time`
2. `base_target_sim_time` 不再用泛化的 `slot_end + offset`
   - 而是明確表達為「最後 historical slot 的下一格」
3. 若 replay 內仍保留 `prediction_target_offset_sec`
   - 應降為 compatibility/debug 參數
   - 不應再作為主語義
4. `predictions.parquet` 補出與 testbed 對齊的欄位語義註記

#### Expected outcome

- replay 的 target 定義可直接對應 testbed `analytics.go`
- 後續 pairing trace 才能看出 replay 與 testbed 是否真的在用同一個 slot identity

### Workstream B. 將 replay prediction store 改成 pending lifecycle model

#### Goal

把 replay 的 prediction state 從「成熟即消費」改成「pending snapshot / resolve / retry / discard」。

#### Planned changes

1. replay `PredictionRecord` 補欄位：
   - stable record id
   - `target_slot_sim_time`
   - `miss_count`
   - `matched_actual_*`
   - `resolved_round_index`
   - `discarded_reason`
2. 新增類似 testbed `ModelAccuracyStore` 的 replay-side pending store abstraction
3. 提供明確 API：
   - `add_prediction(...)`
   - `snapshot_predictions(...)`
   - `resolve_predictions(matched_ids, missed_ids, max_miss_count)`
   - `discard_all_predictions()`
4. monitor round 內不再直接改 `pred.consumed`
   - 改成先 snapshot，再 resolve
5. trace/row 輸出增加可 debug 資訊：
   - matched count
   - missed count
   - discarded count

#### Expected outcome

- replay 可忠實模擬 delayed match
- `monitor_rounds.parquet` 的 sample 組成會更接近 testbed
- 後續 policy 差異才能收斂到真正的數學/時序差異

### Workstream C. 將 replay ground-truth pairing 改成 slot-equality semantics

#### Goal

讓 replay pairing 規則與 `e8c8978` 後的 testbed 一致。

#### Planned changes

1. 移除現行 `lookup_ground_truth_nearest()` 作為主邏輯
2. 改成明確的 slot-key pairing：
   - 以 `target_slot_sim_time` 為 slot origin
   - actual slot 先映射到 slot key
   - 只接受 slot key = 0 的 actual
3. 若需要保留 nearest lookup
   - 只作為 debug/fallback mode
   - 不應作為 default production path
4. 對應補 trace：
   - actual slot start
   - actual slot key
   - matched source
   - contributors / group members

#### Expected outcome

- replay 與 testbed pairing 規則完全同構
- 可以直接排除「因 nearest match 誤配前一格/後一格」造成的 policy 偏差

### Workstream D. 將 warmup / first-monitor 行為調整為 testbed parity

#### Goal

讓 replay 的正式 baseline 起點和 testbed 更一致。

#### Planned changes

1. warmup 結束時改成丟掉該 group / model 的全部 pending predictions
   - 不只丟掉已 matured 的 prediction
2. `inference_since_last_monitor` 重置行為對齊 testbed
3. 檢查 current replay 的 per-group warmup schedule
   - 與 testbed 的 model monitor start concept 是否一致
4. 保留 `subscription_to_first_urr_delay_sec`
   - 但把它定位為 startup schedule 參數
   - 而不是拿來彌補 pairing 邏輯偏差
5. monitor rows 補出：
   - warmup discard count
   - pending snapshot size before resolve

#### Expected outcome

- replay 第一輪正式 monitor 不再混入 testbed 不會保留的 warmup prediction
- `inferenceNum`、`sampleCount`、`windowStart` 的早期行為更接近 testbed

### Workstream E. 將 startup timing 從單一 offset 改為 event-driven profile

#### Goal

把 `subscription_to_first_urr_delay_sec` 從 coarse approximation，
改成可表達 testbed 真實 startup timeline 的資料結構與排程方式。

#### Planned changes

1. 新增 replay-side `StartupTimingProfile`
2. schedule initialization 改為先建立 profile，再算 monitor epochs
3. 支援：
   - `from_logs`
   - `explicit_offsets`
   - `legacy_single_offset`
4. trace 中輸出 startup profile
5. `exp64` 驗證流程中加入 startup timeline 對照

#### Expected outcome

- replay startup 行為不再依賴單一 `30s` 偏移
- monitor timeline 與 data timeline 能獨立對齊 testbed
- early monitor / policy rows 更容易與 `0514-22` 收斂

---

## Recommended implementation order

### Phase 0. 保護既有可對齊面

在任何實作前，先建立或保留以下 guard：

1. aggregation regression check
   - 確保 `slots.parquet` 不被新的 monitor refactor 破壞
2. inference-axis regression check
   - 確保 `predictions.parquet` 的 early target-slot axis 不倒退

### Phase 1. Prediction store refactor

先做 Workstream B。

原因：

- 這是後續 warmup discard 與 delayed match 的基礎
- 若 store 抽象不先穩定，monitor/pairing 會在修改時互相污染

### Phase 2. Pairing semantics refactor

接著做 Workstream C。

原因：

- pairing 規則是 `sampleCount` 與 `trafficScale` 的直接來源
- 不先改這裡，後面的 policy row 對齊很難有結論

### Phase 3. Warmup / monitor schedule refactor

接著做 Workstream D。

原因：

- 這會影響第一輪 monitor 的 snapshot 組成
- 應在 pairing 語義已穩定後調整

### Phase 4. Startup timing profile refactor

接著做 Workstream E。

原因：

- 這會把目前混在 `subscription_to_first_urr_delay_sec` 裡的 startup 語義拆乾淨
- 不先完成這一步，early monitor / policy 仍可能只是「調參對齊」

### Phase 5. Target-time semantics cleanup

最後做 Workstream A 的結構清理與語義收斂。

原因：

- 它對 trace 可讀性與結構正確性很重要
- 但若太早動，容易和 pending lifecycle 改動混在一起

備註：

- 若在實作過程中確認 target-time 語義是當前最大差異來源，
  可把 A 與 B 的順序互換
- 但仍不建議把 A/C/D 同時混在一個 patch 裡

---

## Initial implementation slice

本次建議先落地的範圍，不是五個 workstreams 全開，而是先完成一個
「足以改變 `monitor_rounds.parquet` 與 `policy.parquet` sample composition，
但暫時不重排整條 startup timeline」的最小閉環。

### In scope for the first implementation pass

1. Workstream B 的核心骨架
   - 將 replay prediction state 改成 pending snapshot / resolve / retry / discard
   - 補齊 stable id、`miss_count`、resolved / discarded 狀態欄位
2. Workstream C 的主邏輯
   - 將 replay ground-truth pairing 從 nearest 改成 slot-equality
   - pairing 結果由 pending store resolve，而不是直接 `consumed=True`
3. Workstream D 的 warmup parity 核心
   - warmup 結束時丟棄全部 pending predictions
   - `inference_since_last_monitor` reset 規則維持與 testbed 對齊
4. observability 最小必要面
   - `predictions.parquet` 補 `targetSlotSimTime / missCountFinal / discarded / discardReason / matchedActualSlotStart`
   - `monitor_rounds.parquet` 補 `pendingSnapshotSize / matchedPredictionCount / missedPredictionCount / discardedPredictionCount / warmupDiscardedPredictions`

### Explicitly out of scope for the first pass

1. Workstream E 的 event-driven startup profile
   - 先不改成 `from_logs / explicit_offsets / legacy_single_offset`
   - 先保留現有 per-group schedule 架構，避免一次引入整輪 timing drift
2. Workstream A 的完整 target-time cleanup
   - 先不全面改寫成新的 target/base-target 資料模型
   - 只補足 B/C 需要的 `targetSlotSimTime` 欄位與 trace
3. policy 公式重寫
   - `evaluate_policy()` 本輪不應成為主要修改面

### Why this slice goes first

先做 B/C/D，而不是先做 E/A，原因如下：

1. 目前最直接改變 `sampleCount / trafficScale / predictedTrafficScale / current`
   的，是 pending lifecycle 與 pairing，不是 startup profile 外型
2. compare baseline 已顯示 `slots.parquet` 與 early inference axis 大致收斂
   - 代表 A/E 雖然重要，但暫時不是最優先的 regression source
3. `retrain_replay.py` 目前的程式耦合點也支持先拆 B/C/D
   - `consume_mature_predictions()`、`lookup_ground_truth_nearest()`、
     `discard_warmup_predictions()` 是明確可替換的局部節點
   - `run()` 的 schedule 初始化則會牽動整個 replay 節奏，適合第二輪再動

### Files expected to change in the first pass

- [retrain_replay.py](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/retrain_replay.py)
  - `PredictionRecord`
  - pending prediction store abstraction
  - `lookup_ground_truth_nearest()` replacement
  - `discard_warmup_predictions()`
  - `run_monitor_round()`
  - parquet / event output fields
- [nwdaf-retrain-replay-testbed-parity-plan.md](/home/chingje/testbed/docs/5g-infra/design/nwdaf/nwdaf-retrain-replay-testbed-parity-plan.md)
  - implementation progress updates

### Exit criteria for the first pass

第一批改動完成後，先只要求這幾點成立：

1. `exp64.yaml` replay 仍維持 aggregated slot `shift +0`
2. compare report 中 early inference alignment 不倒退
3. `monitor_rounds.parquet` 的 early rounds 開始出現合理的
   matched / missed / discarded 差異資訊
4. `policy.parquet` 的第一批 row 在 `group-test-001` / `group-test-002`
   不再明顯沿用舊的 nearest-match 樣貌
5. first trigger 若尚未完全對齊，也應至少往 testbed group / timing 收斂，
   而不是維持既有錯誤型態

---

## Required trace and observability additions

這輪重構若缺少觀測面，後續很難驗證。

建議一併補以下輸出欄位或 debug trace。

### `predictions.parquet`

建議新增或明確化：

- `predictionId`
- `groupId`
- `predictedAtSimTime`
- `targetSimTime`
- `targetSlotSimTime`
- `missCountFinal`
- `resolvedRoundIndex`
- `discarded`
- `discardReason`
- `matchedActualSlotStart`

### `monitor_rounds.parquet`

建議新增：

- `pendingSnapshotSize`
- `matchedPredictionCount`
- `missedPredictionCount`
- `discardedPredictionCount`
- `warmupDiscardApplied`
- `warmupDiscardedPredictions`
- `startupTimingMode`
- `subscriptionEpoch`
- `firstUrrSignalEpoch`
- `anchorEstablishedEpoch`
- `firstVisibleInferenceEpoch`

### `events.jsonl`

建議新增或強化：

- `prediction_resolved`
- `prediction_missed`
- `prediction_discarded`
- `monitor_snapshot`
- `monitor_warmup_discard`
- `startup_timing_profile`

這些事件不一定都要永久保留，
但至少在這輪 parity work 中很有價值。

---

## Verification plan

### Standard validation entrypoint

之後每次 replay 重構後的驗證，都應優先走固定入口：

1. 使用 [exp64.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp64.yaml) 重跑一次新的 replay
2. 產生新的 trace output
3. 使用 [compare_nwdaf_replay_testbed.py](/home/chingje/testbed/5G_Infrastructure/.agent/compare_nwdaf_replay_testbed.py)
   將新 output 與 `0514-22` testbed log 做比較

建議把 `exp64` 視為目前這條線的標準 regression fixture。

原因是：

- 它已經是目前 `0514-22` 對齊工作的 baseline config
- 它覆蓋了這輪最重要的 comparison surfaces：
  - `slots.parquet`
  - `predictions.parquet`
  - `monitor_rounds.parquet`
  - `policy.parquet`
  - first trigger timing

### Primary verification window

優先固定在 `0514-22` 的以下視窗：

1. startup / prediction bootstrap
2. CAT1-only policy window
3. early CAT2 pre-trigger window

不要一開始就把 post-trigger / post-swap 當主要驗證面。

### Verification order

每完成一個 workstream，都用相同順序驗證：

1. 用 `exp64.yaml` 跑出新的 replay output
2. 用 `compare_nwdaf_replay_testbed.py` 產出新的 comparison report
3. `slots.parquet`
4. `predictions.parquet`
5. `monitor_rounds.parquet`
6. `policy.parquet`
7. `events.jsonl` / `retrain_jobs.parquet`

也就是：

- 先固定產生新的 replay trace
- 再固定產生新的 compare report
- 最後才解讀各 surface 是否收斂

若 compare report 已明確顯示：

- aggregated slot shift 改變
- inference shift 改變
- policy shift 或 first trigger summary 明顯退步

則應先視為 regression，再決定是否深入解讀細部 policy rows。

### Concrete checks

每輪至少要核對：

1. 新產生的 compare report 是否仍維持：
   - aggregated slot `shift +0`
   - inference `shift +0`
   - first trigger summary 不倒退
2. replay 第一批正式 monitor round 的：
   - snapshot size
   - matched/missed/discarded
3. startup profile：
   - `subscriptionEpoch`
   - `firstUrrSignalEpoch`
   - `anchorEstablishedEpoch`
   - `firstVisibleInferenceEpoch`
   - `firstMonitorEpoch`
4. replay 第一個 policy row 的：
   - `sampleCount`
   - `inferenceNum`
   - `actualTrafficScale`
   - `predictedTrafficScale`
   - `current`
   - `mean`
   - `std`
   - `zscore`
5. `2026-05-14T14:42:16Z`
   - `group-test-002` 是否出現與 testbed 一致的 signal / hits progression
6. `2026-05-14T14:43:46Z`
   - first trigger group / timing / hits 是否一致

### Acceptance threshold

這輪不要求所有浮點值 bitwise identical，
但至少應達到：

- target slot 不再偏一格
- policy row 的 sample composition 明顯一致
- first trigger group 不再跑到錯的 group
- first trigger timing 不再差一整輪 monitor

---

## Risks and caveats

### 1. Replay is group-driven; testbed is subscription-driven

即使 actual aggregation 已對齊，
replay 仍不是完整重放 SBI / callback / goroutine scheduling。

因此：

- 追求的是「語義近乎完全對齊」
- 不是保證所有內部中間狀態都與 Go runtime 完全同序

### 2. Hidden history 仍可能遮蔽 target-time bug

`match_group1_seq7.md` 已顯示：

- testbed log 可見的 aggregated slots 前面，仍有一段 hidden history

所以 replay 若只從可見 rows 倒推 target，
很容易表面對齊、實際仍錯一格。

### 3. Policy baseline accumulation is path-dependent

只要第一輪 sample composition 有差，
後面 `mean/std/zscore/hits` 都會連鎖偏掉。

所以：

- 必須先盯第一輪 monitor / policy
- 不應只看最後有沒有 retrain

---

## Deliverables

這輪重構完成後，應至少產出：

1. 更新後的 `retrain_replay.py`
2. 必要的 replay-side tests
3. 一份新的 comparison report
4. 一份簡短的 parity result note，說明：
   - 哪些面已對齊
   - 還剩哪些 residual gaps

---

## Recommended next step

目前文件已進入收斂狀態。

若仍以 `0514-22` 的 `CAT1-only` 與 `early CAT2 pre-trigger` 對齊為目標，
目前 replay 已可視為 `good enough for current parity target`。

目前建議改為：

1. 進入 maintenance / documentation 狀態
   - 目前不再主動擴張 `retrain_replay.py` 的主要重構範圍
   - 將剩餘 residual 視為已知潛在問題

2. 若未來需要更高精度
   - 優先追 `testbed 實際 inference window` 與 `可見 finalized-slot window` 的差異
   - 再決定是否要把 replay 改成更接近 raw-report driven reconstruction

3. post-trigger / post-swap lifecycle parity
   - 保留為後續獨立題目
   - 不併入目前這份 pre-trigger parity 收斂結果
