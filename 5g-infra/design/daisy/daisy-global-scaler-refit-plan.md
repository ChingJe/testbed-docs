# Daisy Global Scaler Refit Plan

## 目標

修正 Daisy retrain 流程中的 scaler 不一致問題，使每次 retrain task：

- 只產生一份全域共享 `scaler.pkl`
- 所有 Daisy clients 在同一座標系下訓練
- 最終 artifact 中的 scaler 與 retrain 使用的 scaler 完全一致

本設計只處理 scaler，不涵蓋 warm-start 或 fine-tune。

---

## 背景

目前 `07_MTLF_training` 在 `tid` 模式下：

- 每個 client 只抓自己 `group_id` 的資料
- 各自 fit 一份 `StandardScaler`
- 各自寫入同一個 `model/{tid}/scaler.pkl`

這會造成：

- 本地訓練使用不同 scaler
- 聚合模型與最終 artifact scaler 不一致
- retrain 後某一個 group 可能明顯劣化

問題報告見：

- [../../reports/daisy-retrain-scaler-issue.md](../../reports/daisy-retrain-scaler-issue.md)

---

## 方案 A 摘要

採用 **master-side global refit**：

1. replay 照舊呼叫 Daisy `upload_data`
2. replay 照舊呼叫 Daisy `publish_task`
3. Daisy master 在 `publish_task` 內，先從 MongoDB 查出此 `tid` 的所有 training docs
4. master 以所有 groups 的資料一起做 `log1p + StandardScaler.fit`
5. 將結果存為 `model/{tid}/scaler.pkl`
6. spawn Daisy clients
7. 所有 clients 都只讀取這同一份 scaler
8. clients 不再自行 fit 或覆蓋 scaler
9. retrain 完成後，artifact 直接打包這份 scaler

這個方案不需修改 NWDAF 與 replay 的 Daisy API 使用方式。

---

## 設計原則

### 1. 單一 retrain task 對應單一 scaler

對同一個 `tid`：

- 只允許一份 `model/{tid}/scaler.pkl`
- 由 master 建立
- 由所有 clients 共用

### 2. scaler 的資料範圍與 training docs 完全一致

global refit 使用的資料應與 retrain 實際使用的 docs 相同，也就是：

- MongoDB 中 `tid=<task_id>` 的全部文件
- 不額外擴大到其他 task
- 不只看單一 group

### 3. feature extraction 與 client dataset 保持一致

master 計算 scaler 時，必須沿用與 `dataset.py` 相同的前處理邏輯：

- 相同的 notification parsing
- 相同的 10 維 feature 定義
- 相同的 `log1p`

否則 master 計算出的 scaler 與 client dataset 仍可能不一致。

---

## 流程設計

### Phase 1. Upload

NWDAF / replay 維持現況：

- `POST /upload_data`
- `POST /publish_task`

此階段不改 API 契約。

### Phase 2. Master-side scaler refit

在 Daisy master 的 `publish_task()` 中，spawn clients 前執行：

1. 讀取 `tid`
2. 從 MongoDB 查出 `{"tid": tid}` 的所有 docs
3. 將 docs 解析成 feature rows
4. 對每列 feature 做 `log1p`
5. 用所有 rows fit 一個 `StandardScaler`
6. 將 scaler 存到 `model/{tid}/scaler.pkl`

### Phase 3. Spawn clients

clients 啟動後：

- 仍以 `tid + group_id` 查自己的資料
- 但 dataset 不再 fit scaler
- 而是直接 load `model/{tid}/scaler.pkl`

### Phase 4. Training and packaging

訓練完成後：

- master 照既有流程聚合權重
- artifact 直接打包 `model/{tid}/scaler.pkl`
- 這份 scaler 即為 retrain 真正使用的 scaler

---

## 實作變更

### 1. `server_api_handler.py`

檔案：

- `server_api_handler.py`

需新增一個 helper，例如：

- `_build_global_scaler_for_tid(tid: str, scaler_path: str) -> None`

責任：

- query MongoDB `tid`
- parse docs 成 feature rows
- `log1p`
- `StandardScaler.fit`
- `joblib.dump` 到 `model/{tid}/scaler.pkl`

接著在 `publish_task()` 內：

- 在 spawn clients 前呼叫此 helper

### 2. `dataset.py`

檔案：

- `dataset.py`

需調整 `get_dataloaders(...)` 的 `tid` 分支：

- 若 `model/{tid}/scaler.pkl` 已存在：
  - `joblib.load(...)`
  - `EESTCNDataset(..., scaler=loaded_scaler)`
  - `save_scaler=False`
- 不再以本地 docs 自行 fit 並寫回 scaler

目標是讓 `tid` 模式下的 scaler 來源明確改成 master。

### 3. `client.py`

檔案：

- `client.py`

client 本身不需要新增新協定，但要確保：

- `get_dataloaders(...)` 在 `tid` 模式下走共享 scaler 路徑
- 不再讓每個 client 寫 `model/{tid}/scaler.pkl`

### 4. 測試

建議補最少測試：

- 單一 `tid`、多個 `group_id` 時，只產生一份 scaler
- 不同 clients 讀到相同 scaler
- artifact 內 `scaler.pkl` 存在且可被 ML Service 載入

若現有測試可延伸，優先考慮：

- `test_async_callback.py`

---

## 建議的實作細節

### A. 優先重用現有 parsing 邏輯

不要在 master 端重寫另一套 feature extraction。

較佳方式：

- 從 `dataset.py` 抽出可重用 helper
- 例如：
  - `parse_notifications_list(...)`
  - `notifications_to_feature_rows(...)`

讓 master 與 client dataset 共用同一套邏輯。

### B. `tid` 模式預設禁止 client save scaler

`tid` retrain 路徑下，client 不應再寫 scaler。

建議把 `get_dataloaders(...)` 的 `save_scaler` 行為改成：

- 離線本地資料訓練可保留原行為
- `tid` 模式預設只 load，不 save

### C. scaler 建立失敗應直接中止 task

若 master 無法建立 scaler，例如：

- Mongo 無資料
- feature parsing 失敗
- rows 為空

則 `publish_task` 應直接回錯，不應帶著不完整狀態繼續 spawn clients。

---

## 預期效果

完成後，每次 retrain 應具備以下性質：

- 所有 client 在同一個標準化座標系下訓練
- shared model 與 artifact scaler 一致
- 不再出現每個 client 各自覆蓋 `scaler.pkl` 的競態
- retrain 後某一個 group 特別差的現象應有機會明顯改善

---

## 非目標

以下項目不屬於本方案：

- warm-start / fine-tune
- scaler blend / partial update
- group-balanced scaler aggregation
- accuracy monitor / retrain policy 調整

這些應等 scaler 一致性修正完成後再評估。
