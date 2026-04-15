# NWDAF / ML Service — 待辦項目

> 設計背景見 `daisy-nwdaf-design.md`

---

## 1. Async Callback 架構（消除 HTTP blocking）

### `internal/sbi/consumer/daisy_service.go`

- `TriggerTraining` 改為只發 POST，不等待 200
- 回傳 `(taskId string, error)`，task ID 從 Daisy response body 取出
- task payload 加入 `callback_url`：NWDAF 自己的 `/mtlf/training-complete` 完整 URL

### NWDAF config（`nwdafcfg.yaml` / `factory/`）

- `mtlf` 區塊需新增 `callbackUrl` 欄位（例如 `http://<nwdaf-host>:<port>/mtlf/training-complete`），供 `TriggerTraining` 組裝 task payload 使用
- 或：讓程式從 NWDAF 自身 SBI server 設定自動推導，視現有 factory 結構決定

### `internal/sbi/`（新增 callback endpoint）

- 新增 `POST /mtlf/training-complete` route
- Request body：`{"task_id": "...", "model_url": "http://..."}`
- Handler：從 in-flight 狀態表（見下）查出對應上下文，呼叫 `swapModelAfterRetrain(newModelUrl, ...)`

### `internal/mtlf/training.go`

- `triggerTraining`：改回傳 `(taskId string, error)`
- 新增 in-flight 狀態表（`sync.Map`）：`taskId → {oldModelUrl, store *ModelAccuracyStore}`
    - `TriggerRetraining` 發完 task 後存入狀態表
    - callback handler 收到後從狀態表取出，執行 swap，完成後從表中刪除
- `swapModelAfterRetrain` 加 `newModelUrl string` 參數（從 callback payload 取，不再讀 task config）
- 確認 `IsRetraining()` guard 在 async 架構下仍有效：callback 收到前不得重複觸發重訓

---

## 2. MTLF/AnLF 職責分離（ML Service 呼叫路徑）

目前 `swapModelAfterRetrain`（training.go:104）和 `runDelayedTraining` 直接呼叫 `mlClient`，
應改為由 AnLF 負責所有 ML Service 互動。

### `internal/mtlf/training.go`

- `swapModelAfterRetrain`：移除 `mlClient.InitializeModel` / `mlClient.UnloadModel` 呼叫
- 改為：更新 SharedModelRegistry + 訂閱狀態後，觸發 `onModelSwapReady(newModelUrl, oldModelId)` callback
- `runDelayedTraining`（startup）：改為通知 AnLF 載入 static_model_url，不直接操作 ML Service

### `internal/anlf/model.go`

- 新增 `SwapModel(newModelUrl, oldModelId string)` 或接收 MTLF 的 swap 通知的入口
    - 呼叫 ML Service load new model → 取得 new model_id
    - 更新所有使用舊 model 的訂閱狀態
    - 呼叫 ML Service unload old model
    - 重啟 accuracy monitor
- 解耦方式：在 `MtlfService` 初始化時注入 `onModelSwapReady` callback，由 AnLF 實作

---

## 3. ML Service（Python）— tar.gz Bundle 下載

> Go 側的 `ml_service.go` 已經是送 `model_url` 給 ML Service，**不需要改**。
> 需要改的是 Python ML Service 的 `/model/load` endpoint 實作。

目前 `/model/load` 直接從本地 file path 讀取模型，需改為：

- 接收 `model_url`（指向 tar.gz）
- HTTP GET 下載 tar.gz 到暫存目錄
- 解壓，以固定檔名讀取：
    - `model.py`：動態載入模型定義（`from model import Model`）
    - `config.yaml`：讀取架構超參數 + 推論介面設定
    - `model.npy`：載入訓練權重
    - `scaler.pkl`：載入 Scaler
- 原子性：下載失敗或解壓失敗直接 error，不進入半套狀態

---

## 4. SharedModelRegistry Key 設計調整（後續）

> 目前以 model URL 為 key，無法支援多版本並存。可在前三項完成後再處理。

- `NWDAFContext.sharedModels` map key 從 URL 改為 series ID（例如 `"ue-comm-v1"`）
- `SharedModelInfo` 加入版本列表與 active/standby status
- `MlModelInfo.ModelUrl` 改為 `SeriesId`
- 影響範圍：`training.go` / `trigger.go` / `monitor.go` / `anlf/model.go`
