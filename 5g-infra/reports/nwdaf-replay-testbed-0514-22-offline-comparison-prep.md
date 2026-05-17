# NWDAF Replay/Testbed 0514-22 Offline Comparison Prep

**Date:** 2026-05-15, updated 2026-05-17  
**Scope:** 針對 `5G_Infrastructure/.agent/tmp/0514-22` 準備新的 skip-Daisy replay 基準，定義 experiment YAML、比對窗口、主要輸出欄位，並補充目前已確認的 alignment insights。

---

## 1. Purpose

這份文件先做兩件事：

1. 定義新的 replay experiment YAML
2. 定義這次與 `0514-22` testbed run 對照時，哪些時間窗口與哪些欄位應該視為主要基準

這份文件**不**處理：

- Daisy 訓練流程調整
- 對 `NWDAF` / `go-upf-ees` 程式碼的直接修補

---

## 1.1 Current Status Summary

截至 `2026-05-17`，這條 `0514-22` 對齊線的現況可先收斂為：

1. replay `exp64` 已完成，輸出位於：
   - `5G_Infrastructure/NWDAF/NWDAF/tools/retrain_replay/out/exp64_v7_minscale_1m_skip_daisy_0514_22_alignment`
2. `aggregated slot` 表面已基本對齊：
   - testbed `latest aggregated slot`
   - replay `slots.parquet`
   在 `group1` / `group2` 上都可做 `shift +0` 對照，前段 `actualTotal` 幾乎逐格一致。
3. `ML inference` 的 target slot 與輸出也已高度接近：
   - testbed `ML inference`
   - replay `predictions.parquet`
   在 pre-first-swap 視窗內，兩組都可做 `shift +0` 對照，prediction 不是 bitwise identical，但已是同一條 time axis 上的近似結果。
4. 問題已明顯縮到 `monitor / degradation policy`：
   - `current`
   - `mean`
   - `std`
   - `zscore`
   - `predictedTrafficScale`
   - `degradationHits`
   - first trigger group / timing
   仍與 testbed 不一致。
5. 目前最合理的工作假設是：
   - `aggregation` 不是主要問題
   - `inference target slot` 也不是主要問題
   - 差異較可能集中在 `matured prediction/actual pairing`、`policy round` 構成方式、或 degradation baseline/reference sample 的累積條件。

一句話總結目前狀態：

- 問題已大致從「時間軸 / slot 對不齊」縮減到「degradation policy 路徑仍未和 testbed 對齊」。

---

## 2. Reference Artifacts

- testbed run:
  - `5G_Infrastructure/.agent/tmp/0514-22/`
- new experiment YAML:
  - [exp64.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp64.yaml)
- main reference log:
  - [nwdaf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0514-22/nwdaf.log)
- testbed config snapshot:
  - [nwdafcfg.yaml](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0514-22/nwdafcfg.yaml)
- dataset timeline reference:
  - [data.md](/home/chingje/testbed/5G_Infrastructure/go-upf-ess/go-upf/pre_data/data.md)
- current comparison report:
  - [compare_0514_22_vs_exp64.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_0514_22_vs_exp64.md)
- current inference-slot matching example:
  - [match_group1_seq7.md](/home/chingje/testbed/5G_Infrastructure/.agent/match_group1_seq7.md)

---

## 3. Current Execution Mode

這次希望先走 skip-Daisy 快速對齊線，目前可直接使用：

- `retrain_replay.py` 支援 `--skip-daisy`
- `run_experiment.py` 也已支援並會把 `--skip-daisy` 往下傳

因此目前建議的入口是：

- `exp64.yaml` 已可作為 config source of truth
- skip-Daisy fast path 直接走 `run_experiment.py`

參考命令：

```bash
uv run run_experiment.py ../../../docs/5g-infra/experiments/exp64.yaml \
  --initial-bundle out/initial_local_cat1_30s_v7/bundle \
  --skip-daisy
```

---

## 4. Experiment Definition

`exp64` 的意圖是：

- 以 `0514-22` 的 `NWDAF` monitor / policy / retrain window 設定為準
- 在 replay 端重建同一條線
- 先跳過 Daisy 訓練
- 專注比較：
  - `latest aggregated slot`
  - `ML inference`
  - `Accuracy policy`
  - 以及第一次 retrain 前的 trigger 行為

### Main config assumptions

- `samplingInterval: 30`
- `inputWindow: 30`
- `outputWindow: 1`
- `checkInterval: 90`
- `maturity_lag_sec: 60`
- `minSamples: 2`
- `zScoreThreshold: 1.3`
- `requiredHitsInWindow: 2`
- `degradationPolicy.minDecisionTrafficScale: 1048576`
- `retrainWindow: 1200`
- `breaking_time_sec: 900`
- `use_pseudo_warmstart: true`
- `subscription_to_first_urr_delay_sec: 30`

### Initial bundle assumption

這次先沿用目前主線理解：

- initial bundle 先視為 `out/initial_local_cat1_30s_v7/bundle`

原因是：

- `0514-22` 的 config 與 `exp63` 主線一致
- `exp63` 也是目前最接近這條 `v7 + minscale=1MiB` 的歷史對照

如果後續確認 VM 端 `artifacts/initial` 不是這個 bundle，再另外修正 baseline 假設。

---

## 5. Dataset and Time-Axis Assumptions

目前 `pre_data` 應視為 sequential line：

- 每個 UE 都是 `CAT1 -> CAT2 -> CAT3`
- 每段 `1800s`
- UE 間有 `0s / 15s / 30s` offset

根據 `0514-22` 的第一批 `latest aggregated slot`：

- first observed live slot `ts=2026-05-14T14:24:59Z`

結合 `breaking_time_sec=900`，可先把這次 run 的 wall-clock 轉換近似記成：

- dataset `t=900s` 對應 `2026-05-14T14:24:59Z`
- dataset `t=1800s` 對應 `2026-05-14T14:39:59Z`

因此 `CAT1 -> CAT2` 的 wall-clock 切換窗口可先記成：

- UE1: `2026-05-14T14:39:59Z`
- UE2: `2026-05-14T14:40:14Z`
- UE3: `2026-05-14T14:40:29Z`

也就是 group-level transition zone 約為：

- `2026-05-14T14:39:59Z .. 2026-05-14T14:40:29Z`

---

## 6. Comparison Windows

## A. Startup / Prediction Bootstrap Window

用途：

- 只確認 replay 與 testbed 是否在 phase2 起點進入同一條 prediction time axis
- 這段先不把 policy 差異視為主判準

窗口：

- 從第一批 `latest aggregated slot` / `ML inference` 開始
- 到第一個 `Accuracy policy` round 前

在 `0514-22` 中，大致對應：

- `2026-05-14T14:24:59Z` 開始的 live slot
- 到 `2026-05-14T14:27:16Z` 第一輪 policy

主看：

- slot timestamp
- base target time
- prediction target time

## B. CAT1-Only Policy Window

用途：

- 這是目前最重要的 deterministic 對照窗口
- 還沒進 Daisy randomness
- 也還沒正式進入 `CAT1 -> CAT2` transition slots

操作上，先以 testbed 的 accuracy rounds 定義為：

- `2026-05-14T14:27:16Z`
- `2026-05-14T14:28:46Z`
- `2026-05-14T14:30:16Z`
- `2026-05-14T14:31:46Z`
- `2026-05-14T14:33:16Z`
- `2026-05-14T14:34:46Z`
- `2026-05-14T14:36:16Z`
- `2026-05-14T14:37:46Z`
- `2026-05-14T14:39:16Z`
- `2026-05-14T14:40:46Z`

這段窗口的期待是：

- 先讓 replay 對上 testbed 的
  - aggregated actual slots
  - inference target slots
  - first several policy rows
- 若這段都對不上，後面 `CAT2` 段的 policy 差異通常不值得直接解讀

## C. CAT1→CAT2 Transition / Early-CAT2 Pre-Trigger Window

用途：

- 這是這輪真正要觀察 degradation policy 是否一致的核心窗口
- 這裡會開始看到 `predictedTrafficScale`、`current/mean/std/zscore`、
  `degradationHits` 的差異

在 `0514-22` 中，先以這兩輪 policy 為主：

- `2026-05-14T14:42:16Z`
- `2026-05-14T14:43:46Z`

這段的重要事件是：

- `2026-05-14T14:42:16Z`
  - `group-test-002` 出現 `degradationSignal=true`
  - `degradationHits=1/2`
- `2026-05-14T14:43:46Z`
  - `group-test-001` 出現 `degradationSignal=true`
  - `degradationHits=2/2`
  - 進入第一次 retrain trigger

這個窗口的比較終點先定在：

- `2026-05-14T14:43:46Z` retrain trigger round

也就是：

- 比到「第一次 trigger 發生」為止
- 不把 Daisy callback / hot-swap 之後的行為當成主要 alignment reference

## D. Post-Trigger / Pre-Swap Observation Window

用途：

- 只做次要觀察，不當主判準

窗口：

- `2026-05-14T14:43:49Z` Daisy submit
- 到 `2026-05-14T14:44:46Z` hot-swap complete

這段可以保留做：

- retrain pending 期間的 slot / inference 行為 spot-check

但不應把它當成主要 deterministic 對照面，因為：

- skip-Daisy replay 與 real Daisy pending state 的 lifecycle 不完全相同

---

## 7. Primary Comparison Surfaces

## A. `slots.parquet`

用途：

- 對照 testbed `latest aggregated slot`

主要欄位：

- `simTime`
- `windowIndex`
- `groupId`
- `slotStart`
- `slotEnd`
- `actualUl`
- `actualDl`
- `actualTotal`
- `actualUlPkts`
- `actualDlPkts`
- `actualUlThr`
- `actualDlThr`
- `actualUlPktThr`
- `actualDlPktThr`

## B. `predictions.parquet`

用途：

- 對照 testbed `ML inference`

主要欄位：

- `predictedAtSimTime`
- `targetSimTime`
- `groupId`
- `modelVersion`
- `modelSource`
- `predUl`
- `predDl`
- `confidence`
- `matchedActualUl`
- `matchedActualDl`
- `maturedAtSimTime`

## C. `monitor_rounds.parquet`

用途：

- 對照 testbed `Accuracy` / `Accuracy scope`
- 補看每輪實際成熟樣本數與 monitor window 範圍

主要欄位：

- `simTime`
- `modelVersion`
- `scope`
- `groupId`
- `sampleCount`
- `inferenceNum`
- `windowStart`
- `windowEnd`
- `metricsJson`
- `trafficScale`
- `predictedTrafficScale`

## D. `policy.parquet`

用途：

- 對照 testbed `Accuracy policy`
- 這是本輪最重要的主表

主要欄位：

- `timestamp`
- `model`
- `scope`
- `metric`
- `current`
- `mean`
- `std`
- `zscore`
- `degradationEligible`
- `degradationSignal`
- `baselineReady`
- `degradationBaselineReady`
- `recentBaselineReady`
- `actualTrafficScale`
- `recentTrafficScaleMean`
- `predictedTrafficScale`
- `degradationReferenceSamples`
- `recentSamples`
- `degradationHits`
- `degradationRequired`
- `hitReason`

## E. `events.jsonl` / `retrain_jobs.parquet`

用途：

- 只對照第一次 retrain trigger timing
- 不把 skip-Daisy 的 mock lifecycle 視為真實 Daisy wall-time 對照物

主看：

- first `retrain_trigger`
- `triggerSimTime`
- `scope`
- `reason`

---

## 8. Comparison Priorities

這輪的優先順序應固定為：

1. `slots.parquet` 是否已和 testbed `latest aggregated slot` 對上
2. `predictions.parquet` 是否使用了同一批 historical slots 與 target slot
3. `policy.parquet` 的 `current / mean / std / zscore / hits` 是否在 CAT1 與 early CAT2 收斂
4. 第一次 trigger 的 group / 時間 / hits 是否一致

只有前 3 點都大致收斂時，才值得解讀後續 trigger 差異。

### Updated priority after `exp64`

截至目前，優先順序可再細化成：

1. `slots.parquet` vs testbed `latest aggregated slot`
   - 已基本收斂
2. `predictions.parquet` vs testbed `ML inference`
   - 已基本收斂到同一條 target-slot time axis
3. `policy.parquet` vs testbed `Accuracy policy`
   - 目前仍是主要 mismatch surface
4. `monitor_rounds.parquet`
   - 應優先拿來追 `sampleCount` / `inferenceNum` / `windowStart` / `windowEnd`
5. `events.jsonl` / `retrain_jobs.parquet`
   - 只保留作 first trigger timing 對照，不再當主要根因面

也就是說，後續分析的主戰場應明確鎖定在：

- `monitor round pairing`
- `policy baseline accumulation`
- `degradation signal / hits progression`

---

## 9. If Mismatch Remains

若 replay 與 testbed 仍明顯對不上，下一步應優先做：

1. 暴力枚舉並核對 inference 實際吃了哪些 slot
2. 檢查 testbed `latest aggregated slot` 與 replay `slots.parquet` 的 group aggregation 是否仍有隱性差異
3. 檢查每一輪 policy 所使用的 matured prediction/actual pair 是否配到同一組 target slot

也就是：

- 先查 `inference input slots`
- 再查 `group aggregation`
- 最後才查 `policy math`

### Updated interpretation after current checks

目前這三步已可更新為：

1. `inference input slots`
   - 已有強證據顯示 replay 與 testbed 使用的是同一條 aggregated-slot 序列
   - 但仍保留逐點 spot-check 空間
2. `group aggregation`
   - 在目前已比對的窗口中已基本對齊
3. `policy math`
   - 這裡現在最值得投入分析時間

也就是：

- 先前的排查順序仍正確
- 但目前可實務上把焦點收斂到 `policy / monitor pairing`

---

## 10. Current Analysis Tooling

目前已先放了兩支輔助腳本在 `5G_Infrastructure/.agent`：

### A. `compare_nwdaf_replay_testbed.py`

用途：

- 以整體報表方式比較：
  - testbed `nwdaf.log`
  - replay `slots.parquet`
  - replay `predictions.parquet`
  - replay `policy.parquet`

主要輸出：

- aggregated slot surface
- inference surface
- policy surface
- first trigger summary

目前對應報告：

- [compare_0514_22_vs_exp64.md](/home/chingje/testbed/5G_Infrastructure/.agent/compare_0514_22_vs_exp64.md)

### B. `match_testbed_inference_slots.py`

用途：

- 針對單一 testbed `ML inference` 做更細的 slot-window 對照
- 先用 log 中的 `group + targetTime + predicted UL/DL` 選定某次 inference
- 再用 replay / `pre_data` 聚合出的 slot 序列暴力匹配模型輸入窗口

原理分兩段：

1. `log-visible window`
   - 從 testbed log 中回推出該次 inference 時，log 內可見的 `latest aggregated slot`
2. `replay/pre_data candidate window`
   - 直接以 replay `slots.parquet` 作為候選 aggregated slots
   - 載入同一個 bundle 與 scaler
   - 對候選 window 重跑模型
   - 以 testbed `ML inference` 的 `predUl/predDl` 做暴力匹配

這支腳本的重要補充是：

- 它不假設 `last aggregated slot` log 就完整代表實際輸入窗口
- phase1 補齊的 hidden history 可以透過 aggregated slot 在 replay/pre_data 中的精準定位回推出來

目前對應示例：

- [match_group1_seq7.md](/home/chingje/testbed/5G_Infrastructure/.agent/match_group1_seq7.md)

這個例子已顯示：

- testbed log 中可見的 8 筆 aggregated slots
- 可在 replay/pre_data 聚合序列中做 exact match，對應到 `rawSeq 29..36`
- 因此可反推出完整模型輸入窗口為 `rawSeq 7..36`
- 也就是在第一個可見 logged slot 之前，仍有 `22` 個 phase1 / hidden history slots

---

## 11. Immediate Next Step

在這份準備文件之後，下一步應是：

1. 以 [exp64.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp64.yaml) 與 `exp64` 輸出作為目前 baseline
2. 針對 `CAT1-only` 與 `early-CAT2 pre-trigger` 窗口，聚焦分析：
   - `monitor_rounds.parquet`
   - `policy.parquet`
3. 如有需要，再用 `match_testbed_inference_slots.py` 針對關鍵 policy rounds 前的 inference 做 spot-check

若這兩段已收斂，再決定是否需要補更細的 per-slot / per-inference trace tooling。
