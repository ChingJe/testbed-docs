# NWDAF Retrain Replay：實驗 1–10 報告

## 目標

在 CAT 流量模式切換（CAT1→CAT2→CAT3）時偵測 concept drift，並透過 accuracy monitor
觸發模型 retrain。目標：**以 degradation（z-score）作為主要觸發機制**，chronic 僅作用於
initial model 的第一次偵測。

資料集：`cat_sequential_90min_data`（Group1）+ `cat_overlap_90min_data`（Group2）。  
CAT 切換時間點：T=00:30（live 開始，進入 CAT2）、T=01:00（進入 CAT3）。  
CAT2 前的歷史資料透過 pseudo-warmstart（breaking_time=1780s）預載入模型。

---

## 實驗參數總覽

| 參數 | exp1 | exp2 | exp3 | exp4 | exp5 | exp6 | exp7 | exp8 | exp9 | exp10 |
|---|---|---|---|---|---|---|---|---|---|---|
| **period** | 30s | 30s | 30s | 60s | 60s | 30s | 30s | 30s | 30s | 30s |
| **checkInterval** | 150s | 150s | 150s | 300s | 180s | 120s | 60s | 120s | 150s | 150s |
| **recentBufSize** | 15 | 15 | 15 | 10 | 10 | 15 | 15 | 12 | 12 | 12 |
| **minBufSamples** | 5 | 5 | 10 | 4 | 5 | 3 | 8 | 5 | 5 | **3** |
| **minSamples** | — | — | — | — | 3 | 3 | 2 | 4 | 5 | 5 |
| **primaryMetric** | MAE | MAE | MAE | MAE | MAE | MAE | WAPE | WAPE | WAPE | WAPE |
| **zScoreThreshold** | 3.0 | 3.0 | 3.0 | 3.0 | 3.0 | 2.0 | 1.5 | 1.5 | 1.5 | 1.5 |
| **window/hits** | 5/3 | 5/3 | 5/3 | 5/3 | 5/3 | 5/3 | 5/3 | 5/3 | 5/3 | 5/3 |
| **fixedFloor** | — | — | — | — | — | 1024B | 0.05 | 0.05 | 0.05 | 0.05 |
| **chronic 閾值** | 1.0 | 0.88 | 0.88 | 0.88 | 0.88 | 0.75 | 0.90 | 0.90 | 0.90 | 0.90 |
| **seqLength** | 30 | 30 | 30 | 30 | 30 | 30 | **10** | **10** | **10** | **10** |
| **maturityLag** | 60s | 60s | 60s | 120s | 120s | 60s | 60s | 60s | 60s | 60s |
| **retrainWindow** | — | — | — | 2700s | 2700s | — | — | — | — | — |

### 演進邏輯

| 階段 | 實驗 | 關鍵變更 | 結果 |
|---|---|---|---|
| 基準建立 | exp1–3 | 調整 chronic 閾值、minBufSamples | chronic 成功對 initial model 觸發 |
| 60s 粒度測試 | exp4–5 | 60s period、45min retrain window | FL averaging 傷害 Group2，方向放棄 |
| z-score 啟動嘗試 | exp6 | 降低 zThresh/chronic（MAE 指標） | MAE std≈0，z-score 完全失效 |
| 指標切換 | exp7 | **WAPE + seqLen=10** | z-score 首次有意義 |
| 純淨基線建立 | exp8–9 | 調整 checkInterval、minBufSamples | 確保 baseline 在 T=01:00 前填滿純 CAT2 資料 |
| 最佳結果 | **exp10** | **minBufSamples=3** | z-score 在 CAT3 後命中 2/3 次，差一步觸發 |

---

## exp10 詳細結果（最佳實驗）

**設定：** 30s period、checkInterval=150s、minBufSamples=3、WAPE、zThresh=1.5、seqLen=10

### Retrain 工作記錄

| 觸發時間 | 觸發原因 | Scope | 模型生效時間 |
|---|---|---|---|
| T=00:45:00 | chronic | group1 | T=00:51:15 |
| T=01:07:30 | chronic | group2 | T=01:16:26 |

兩次觸發均為 **chronic**，無任何 degradation 觸發。

### Group1 policy trace（T≥00:45，指標：WAPE）

| 時間 | WAPE | mean | std | z-score | baselineReady | degradHits | chronicHits | 結果 |
|---|---|---|---|---|---|---|---|---|
| 00:45:00 | 0.9797 | 0.9561 | 0.0174 | 1.35 | True | 1 | **3** | **chronic → 觸發** |
| 00:55:00 | 0.8504 | — | — | — | False | 0 | 0 | 無（模型剛換，buffer 重置） |
| 00:57:30 | 0.5057 | 0.8504 | 0.0000 | -34.47 | False | 0 | 0 | skipped |
| 01:00:00 | 0.5937 | 0.6781 | 0.1723 | -0.49 | False | 0 | 0 | skipped |
| **01:02:30** | **0.9001** | **0.6500** | **0.1462** | **1.71** | True | **1** | 0 | degradation ✓ |
| **01:05:00** | **1.0795** | **0.7125** | **0.1666** | **2.20** | True | **2** | 1 | both ✓ |
| **01:07:30** | **0.9382** | **0.7859** | **0.2092** | **0.73** | True | **2** | 2 | chronic（z-score 未命中 ✗） |
| 01:27:30 | 0.9777 | 0.9591 | 0.0057 | 1.86 | True | 1 | 1 | both |
| 01:30:00 | 0.9680 | 0.9638 | 0.0095 | 0.42 | True | 1 | 2 | chronic |

**CAT3 切換（T=01:00）後：** degradation hit #1 於 T=01:02:30、hit #2 於 T=01:05:00。  
T=01:07:30 需第 3 次命中，但 z-score 從 2.20 驟降至 0.73 → **未觸發**。

### Group2 policy trace（T≥00:45，指標：WAPE）

| 時間 | WAPE | mean | std | z-score | baselineReady | degradHits | chronicHits | 結果 |
|---|---|---|---|---|---|---|---|---|
| 00:57:30 | 1.3840 | 0.5672 | 0.0000 | 81.68 | **False** | 0 | 0 | skipped（baseline 未就緒） |
| 01:00:00 | 0.8873 | 0.9756 | 0.4084 | -0.22 | False | 0 | 0 | skipped |
| 01:02:30 | 0.8942 | 0.9462 | 0.3361 | -0.15 | True | 0 | 1 | chronic |
| 01:05:00 | 0.9326 | 0.9332 | 0.2919 | -0.002 | True | 0 | 2 | chronic |
| **01:07:30** | **0.9489** | **0.9331** | **0.2611** | **0.06** | True | 0 | **3** | **chronic → 觸發** |

Group2 整個過程 degradation hits = **0**，完全靠 chronic 觸發。

---

## 問題清單

### 問題一：Sliding baseline 被 CAT3 資料汙染（主要問題，Group1）

**現象：** 前兩次 z-score hit 的高 WAPE 值被寫入 sliding buffer，使 mean 上升，壓低後續 z-score。

```
T=01:02:30  hit #1：mean=0.650，std=0.146，z=1.71
T=01:05:00  hit #2：mean=0.713，std=0.167，z=2.20  ← 第1筆 CAT3 資料進入 buffer
T=01:07:30  miss：  mean=0.786，std=0.209，z=0.73  ← mean 被拉高，z-score 崩潰
```

z-score baseline 自動適應了它本來應該偵測的異常資料。若 baseline 在 `minBufSamples`
填滿後就凍結，T=01:07:30 的假設 z 為 (0.938 - 0.650) / 0.146 = **1.97 → hit #3 → 觸發**。

**修法方向：** 雙 buffer 架構——獨立的 `ref_buffer`（填滿後凍結，供 z-score 計算
mean/std）與原有 `metric_buffer`（滑動，供 chronic percentile 使用）。
新增設定參數 `baselineFreezeAfter: 0`。

---

### 問題二：Group2 的 CAT2 baseline std 過高，z-score 完全失效

**現象：** Group2 在 CAT2 期間的 WAPE std ≈ 0.34–0.41，需要 WAPE 超過
`mean + 1.5 × std ≈ 0.93 + 0.62 = 1.55` 才能觸發，但 Group2 在 CAT3 的 WAPE 只有
0.89–0.95，遠低於閾值。

Group2 的 CAT2 baseline WAPE 樣本：`[0.692, 0.879, 0.779, 0.682, 0.567, 1.384, 0.887]`  
→ std = 0.41，mean = 0.84，觸發閾值 = **1.46**

整個 CAT3 期間 Group2 的 WAPE 未達 1.46 → degradation hits = 0。

**根本原因：** Group2 流量較小且變化較大（browsing/social vs. upload/download），
CAT2 與 CAT3 的 WAPE 本身沒有足夠的差距。T=00:57:30 出現的 WAPE=1.384 異常值
（sampleCount=1 的 round，見問題三）也進一步拉高了 baseline std。

**可能的修法：**
- 將 sampleCount=1 的 round 排除在 baseline 之外（可消除 1.384 異常值）
- 以流量大小對 baseline 加權（高流量 round 的權重較高）
- 接受 Group2 依賴 chronic 觸發，另行調整 chronic 閾值

---

### 問題三：`minSamples` gate 以全 group 加總計算，而非 per-group

**現象：** `retrain_replay.py` 第 884 行：

```python
if total_pairs < self.min_samples or ...:
    continue
```

`total_pairs` 是**所有 group 合計**的 mature pair 數，但 `continue` 的作用是在 per-group
迴圈內跳過該 group 的 policy 評估。若 Group1 有 4 筆、Group2 有 1 筆，
`total_pairs=5 >= minSamples=5`，Group2 的 policy 仍會以 1 筆樣本執行。

若真實 NWDAF 的 Go 實作邏輯相同，此問題同樣存在。

**證據：** T=00:52:30（模型在 T=00:51:15 剛 swap）出現 sampleCount=1 的 round，
通過了 gate 並進入 baseline 計算，導致 Group2 在 T=00:57:30 出現 WAPE=1.384 的異常值，
汙染 baseline std。

**修法：** 改為 per-group gate——`if len(pairs) < self.min_samples`。

---

### 問題四：MAE 不適合用於 z-score（已於 exp7 解決）

**過去狀況（exp1–6）：** MAE 以 bytes 為單位，在同一 CAT phase 內 std ≈ 50–200 bytes
（固定 UE 行為 → 預測穩定 → 低變異）。`minStd=0.01` 是針對無單位指標設計的，
在 bytes 量級下幾乎沒有作用。正常的 ±500 bytes 波動就會產生 z-score = 2–10，
導致頻繁誤觸發。

**解決方式：** 改用 WAPE（無單位，與流量量級無關）。同一 phase 內 WAPE std ≈ 0.05–0.15，
z-score 才具有判別意義。

**殘留隱憂：** WAPE = error / actual，當實際流量接近 0 時不穩定。
`minTrafficScale: 1024` 只保護了 chronic 路徑，z-score 路徑沒有對應的流量下限保護。
在真實環境中若有 UE 靜止或斷線的時段，WAPE z-score 可能產生誤觸發。

---

### 問題五：FL averaging 在 retrain 後傷害 Group2

**現象：** 訓練資料以 Group1（流量較大）為主導。Retrain 後，共享模型權重向 Group1
的流量模式偏移，Group1 改善，Group2 反而變差。

在 exp4–5（60s period）中觀察到 Group2 retrain 後 WAPE 上升而非下降。

這是 federated averaging 在 group 流量不平衡時的根本問題，exp1–10 中未處理。

---

## 總結

| 問題 | 嚴重程度 | 是否影響真實 NWDAF | 修法狀態 |
|---|---|---|---|
| Sliding baseline 被 CAT3 資料汙染 | 高 | 是 | 已提出方案（dual buffer） |
| Group2 CAT2 baseline std 過高 | 高 | 是 | 尚無修法 |
| minSamples gate 以全 group 加總計算 | 中 | 可能 | 小改動即可修復 |
| MAE 不適合 z-score | 高 | 是 | 已解決（改用 WAPE，exp7） |
| FL averaging 不平衡 | 中 | 是 | 尚未處理 |

**最優先的下一步**是實作 **frozen reference baseline**（問題一的修法）。
根據 exp10 數據，若 baseline 在填滿後凍結，T=01:07:30 的 z-score 將為 1.97，
觸發第 3 次 degradation hit，實現以 degradation 主導的 CAT3 切換偵測。
