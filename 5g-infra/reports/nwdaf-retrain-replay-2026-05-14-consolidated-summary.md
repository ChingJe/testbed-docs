# NWDAF Retrain Replay 2026-05-14 Consolidated Summary

**Date:** 2026-05-14
**Scope:** 統整這一輪 `v7` initial bundle 重建、`v8/v9/v10` initial-model 嘗試、以及 `exp49` 到 `exp63` 的 replay / skip-Daisy / Daisy 驗證結果。

---

## 1. 這輪要解的問題

目標原本是：

1. replay 與 testbed 對齊後，重新找到合理的 retrain trigger 行為
2. 讓 retrain 盡量在 `CAT` 切換後才觸發
3. 避免第一輪 retrain 太晚，導致 train window 已經接近下一個 `CAT` 切換點
4. 若可能，希望整體接近「有意義的前兩輪 retrain，避免不必要的晚期第三輪」

---

## 2. 初始模型線的主要結論

### 2.1 `train_initial_local.py` 與當前 replay aggregation 邏輯一致

已確認：

- `train_initial_local.py` 直接共用 `retrain_replay.load_group_slots()`
- 現在的 loader 已經是 alignment 後的 per-UE rebasing + group aggregation 語意

因此：

- 當前重新訓練出的 initial bundle 與目前 replay 實作是一致的
- 舊 bundle 是否語意過時，要看它是不是 alignment 後重訓

### 2.2 `v7` 是目前較乾淨的基準 initial bundle

`v7` 是 alignment-era 之後重建的：

- `CAT1 only`
- `CAT1 scaler`
- `train_ratio = 0.9`
- `batch_size = 8`
- `learning_rate = 5e-4`

離線上它是目前這輪最合理的 replay 基準 initial bundle。

### 2.3 單靠 initial bundle 調整，沒有解掉 first-trigger timing

本輪測過：

- `v7`
- `v8`：`fit-all CAT1 + reuse_train_for_val`
- `v9`：`train_ratio=0.9`, `batch_size=32`, `learning_rate=1e-3`
- `v10`：`v9 + loss=mse`
- 也回頭驗證過歷史 `v3`

結論：

- `v7 / v8 / v9 / v10` 在原本那條 policy 線上，first trigger 都卡在 `00:57:50`
- `v3` 更差，直接 `0` 次 retrain
- `v10` 的 `mse` 比 `v9` 離線結果更差，也沒有帶來 replay 改善

因此：

- 問題主因不在 initial model 本身的 training hyperparameter
- 問題主因在 degradation baseline 如何吃進 early low-traffic samples

---

## 3. 為什麼 first trigger 會卡在 `00:57:50`

這輪最重要的發現是：

- `CAT1` 前期存在少數 very-low-traffic observations
- 初始模型在這些點上會出現極端高 `WAPE`
- 這些 observation 在 baseline ready 前就被吸進 degradation baseline
- 結果導致 `CAT2` 開始後，雖然預測仍然很差，`z-score` 卻不夠早拉高

關鍵現象可概括為：

- `group2 @ 00:18:50`：`actual ≈ 180 KB`，`pred ≈ 3.10 MB`，`WAPE ≈ 16.15`
- `group1 @ 00:21:50`：`actual ≈ 334 KB`，`pred ≈ 1.71 MB`，`WAPE ≈ 4.47`

這些點不會當場 trigger，但會污染 degradation baseline。

---

## 4. 歷史 baseline 對照：`exp27`

`exp27` 沒有現在這種 baseline 被 early low-traffic outlier 撐壞的現象。

它的特徵是：

- first trigger 很早：`group2 @ 00:34:30`
- `CAT1` baseline 相對乾淨
- `CAT2` 開始後 `group2` 的 degradation hit 可以快速累滿

因此可以確認：

- 現在遇到的問題不是 NWDAF trigger 設計本身壞掉
- 而是目前這條 replay 線上的 early baseline 狀態和 `exp27` 不一樣

---

## 5. Daisy 行為的主要結論

### 5.1 Daisy 不會部署 best validation checkpoint

目前 Daisy 路徑的行為是：

- early stopping 只影響停訓時機
- 實際打包出去的是最後一輪 `model.npy`
- 不是最佳 validation round 的 checkpoint

這件事對 swapped model 表現有明顯影響，但不是 first trigger 卡住的主因。

### 5.2 `ES_PATIENCE=2` 能縮短第一輪 wall time，但不足以恢復第二輪

`exp50` 已確認：

- 降低 `ES_PATIENCE` 可縮短第一輪 Daisy 時間
- 但不會單獨解掉第二次 retrain 消失的問題

---

## 6. 實驗結果總結

### 6.1 以原始 policy 線為基準的結果

- `exp49`
  - policy 放鬆為 `z=1.3`, `requiredHits=2`
  - 只有 `1` 次 retrain
  - first trigger：`group2 @ 00:57:50`
- `exp50`
  - 在 `exp49` 上加 `ES_PATIENCE=2`
  - 仍只有 `1` 次 retrain
  - first trigger 仍是 `00:57:50`
- `exp51`
  - 換成 `v7`
  - first trigger 仍是 `00:57:50`
- `exp52`
  - `exp51` 的 skip-Daisy 對照
  - 兩次 retrain，但 first trigger 仍是 `00:57:50`
  - 說明 first-trigger 問題與 Daisy wall time 無關

### 6.2 初始模型實驗線

- `exp53`
  - `v8` + skip-Daisy
  - first trigger 仍 `00:57:50`
- `exp54`
  - `v9` + skip-Daisy
  - first trigger 仍 `00:57:50`
- `exp55`
  - `v3` + skip-Daisy
  - `0` 次 retrain
- `exp56`
  - `v10` + skip-Daisy
  - first trigger 仍 `00:57:50`

結論：

- initial model 線無法解 first-trigger timing

### 6.3 `minDecisionTrafficScale` 探索線

這是本輪真正有用的線。

#### `327680` / `384 KiB`

- `exp58` skip-Daisy
  - first trigger：`group2 @ 00:33:50`
  - second trigger：`group2 @ 00:57:50`
  - third trigger：`group1 @ 01:20:20`
- `exp59` full Daisy
  - `3` 次 retrain
  - `group2 @ 00:33:50`
  - `group1 @ 01:02:20`
  - `group1 @ 01:20:20`
  - 第一輪 swap `00:36:00`，仍在 `CAT2`

這證明：

- `minDecisionTrafficScale` 的方向是對的
- 而且能把第一輪 retrain 拉回有意義的 `CAT2`
- 但第三輪仍存在

#### `1572864` / `1.5 MiB`

- `exp60` skip-Daisy
  - `2` 次 retrain
  - `group2 @ 00:33:50`
  - `group1 @ 01:09:50`
  - 第三輪消失
- `exp61` full Daisy
  - 只剩 `1` 次 retrain
  - 只有 `group2 @ 00:33:50`
  - 第一輪 swap 仍在 `CAT2`
  - 第二輪在真實 Daisy 路徑下消失

這表示：

- `1.5 MiB` 太高
- 它會連第二輪一起壓掉

#### `1048576` / `1 MiB`

- `exp62` skip-Daisy
  - `2` 次 retrain
  - `group2 @ 00:33:50`
  - `group1 @ 01:09:50`
  - 第三輪消失
- `exp63` full Daisy
  - 又回到 `3` 次 retrain
  - `group2 @ 00:33:50`
  - `group1 @ 01:02:20`
  - `group1 @ 01:20:20`
  - 第一輪 swap `00:35:00`
  - 第二輪 swap `01:05:30`

這表示：

- `1 MiB` 在 skip-Daisy 下看起來漂亮
- 但 full Daisy 下第三輪仍然回來

---

## 7. 目前最重要的結論

### 7.1 已經成功解掉「第一輪 retrain 太晚」這個最大問題

使用：

- `initial bundle = v7`
- `degradationPolicy.minDecisionTrafficScale = 327680`

時，full Daisy 的第一輪 retrain 已經變成：

- trigger：`00:33:50`
- hot swap：`00:36:00`

也就是：

- 第一輪 swap 仍留在 `CAT2`
- retrain window 不再像以前那樣已經逼近 `CAT2 -> CAT3`

這是本輪最大的有效進展。

### 7.2 但「只保留兩輪、避免第三輪」還沒有在 full Daisy 下找到穩定設定

目前 full Daisy 的觀察是：

- `327680`：3 次 retrain
- `1048576`：3 次 retrain
- `1572864`：1 次 retrain

因此：

- 只靠 `minDecisionTrafficScale`，目前還找不到 full Daisy 下剛好 `2` 次的甜蜜點

### 7.3 第三輪比較像是第二輪 retrain 後的模型仍然在 `group1/CAT3` 某些 pocket 上表現差

從 `exp59` / `exp63` 來看：

- 第三輪都來自第二輪 swapped model 後的 `group1`
- 不是單純第一輪 baseline 問題

所以第三輪更像是：

- second swapped model 對 `CAT3` 的某些低量級區段仍然不穩
- 因此晚期又重新累出兩次 degradation hit

---

## 8. 當前建議

### 8.1 若優先目標是「先讓第一輪 retrain 有意義」

目前最可用的設定是：

- initial bundle：`v7`
- `degradationPolicy.minDecisionTrafficScale = 327680`

因為它能穩定做到：

- 第一輪 retrain 早進 `CAT2`
- 第一輪 real Daisy swap 仍留在 `CAT2`

### 8.2 若下一步要繼續追求「full Daisy 下只剩兩輪」

不建議再只掃 `minDecisionTrafficScale`。

更合理的下一步應轉向：

- `enable_continue_learning = true`
- 或 `use_fixed_scaler = true`

因為：

- 第三輪問題比較像 second swapped model 的後續表現問題
- 不再只是 early baseline 污染問題

---

## 9. 當前總結

這一輪的整體結論可以濃縮成三句話：

1. `initial model` 不是主因，換 `v7/v8/v9/v10` 都沒有解掉原本 `00:57:50` 的 first-trigger 問題。
2. 真正有效的 knob 是 `degradationPolicy.minDecisionTrafficScale`，它已經成功把第一輪 real Daisy retrain 拉回有意義的 `CAT2`。
3. 但 full Daisy 下想同時滿足「保留前兩輪、消除第三輪」，目前只靠這個參數還做不到；下一步應該從 second swapped model 的後續行為著手。
