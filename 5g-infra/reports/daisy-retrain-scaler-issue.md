# Daisy Retrain Scaler Issue Report

**Date:** 2026-05-04
**Component:** Daisy FL framework (`daisy/examples/07_MTLF_training`)
**Symptom:** retrain 完成後，常出現某一個 group 的推論效果明顯劣化，導致 replay / testbed 上某一側 WAPE 長時間偏高。

---

## 根本問題

目前 Daisy retrain 流程中，`scaler.pkl` 並不是以「單一 retrain task 的全域共享 scaler」方式產生，而是由各 client 依自己的 group 資料各自 `fit`，再寫回同一個 task 目錄。

這會造成：

- 不同 client 在不同 feature 座標系下進行本地訓練
- 聚合後的 shared model 對應不到單一一致的 scaler
- 最終 artifact 內的 `scaler.pkl` 可能只是某一個 client 最後覆蓋寫入的結果

這是 retrain 後某一個 group 特別差的高度可疑主因。

---

## 問題鏈

### 1. 每個 client 各自依本地 group 資料 fit scaler

在 `tid` 模式下，client 會依自己的 `group_id` 從 MongoDB 查資料，再建立 dataset：

- `client.py`
- `dataset.py`

若沒有外部提供 scaler，dataset 會直接對本次資料 `fit` 一個新的 `StandardScaler`：

- `dataset.py`

也就是說：

- `group1` client 用 `group1` 的 scaler
- `group2` client 用 `group2` 的 scaler

但兩邊最後仍會把模型權重聚合成一個 shared model。

### 2. 不同 scaler 下的本地更新被直接聚合

目前 Daisy 的 FL 流程沒有額外保證所有 client 使用同一份 scaler。

因此每個 client 實際上是在不同標準化座標系下訓練，最後卻直接 FedAvg 聚合。這會讓 shared model 學到彼此不一致的輸入/輸出對應關係。

### 3. 各 client 會覆蓋寫入同一個 `scaler.pkl`

在 `tid` 模式下，dataset 會把 scaler 存到：

- `model/{tid}/scaler.pkl`

對應程式：

- `dataset.py`

由於每個 client 都會寫這個路徑，實際上存在覆蓋競態：

- 最後 artifact 內留下來的 `scaler.pkl`
- 很可能只是最後一個完成寫入的 client 所產生的 scaler

### 4. Master 打包 artifact 時只會帶走那一份 scaler

Master 打包 retrain artifact 時，會把 task 目錄下的 `scaler.pkl` 收入最終 tar.gz：

- `server_api_handler.py`

這代表 retrain 後 deployed bundle 雖然只有一份模型，也只有一份 scaler，但這份 scaler 並不保證和聚合後模型一致，更不保證對所有 group 都合理。

---

## 為什麼這會放大成 group-specific 劣化

這個問題不只影響輸入特徵，也會影響訓練標籤與推論反標準化：

- dataset 先對完整 10 維 feature 做 `log1p + StandardScaler`
- 再從標準化後的 feature 中取 `ul_vol / dl_vol` 當 target
- 推論端最後還要依 bundle 內的 scaler 做 inverse transform

因此若 artifact 內的 scaler 偏向某一個 group：

- 該 group 的輸入分布與模型較一致，效果可能較好
- 另一個 group 的輸入/target 座標系就會偏掉，效果可能明顯變差

這與目前 retrain 後常觀察到「某一個 group 特別差」的現象一致。

---

## 目前判斷

這個 scaler 問題不一定是唯一原因，但它已足以單獨造成：

- retrain 後 shared model 表現不穩
- 某一個 group 長時間 WAPE 偏高
- post-retrain baseline 失真
- 後續 transition 偵測結果被污染

因此目前應優先處理 scaler 的一致性問題，再評估是否需要進一步調整 retrain 策略。

---

## 建議方向

下一步應改成：

- 每次 retrain task 只產生一份全域共享 scaler
- 所有 client 共用這份 scaler
- client 不再各自 `fit` 並覆蓋 `scaler.pkl`

具體方案見：

- [../design/daisy/daisy-global-scaler-refit-plan.md](../design/daisy/daisy-global-scaler-refit-plan.md)
