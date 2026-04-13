# NWDAF × Daisy 整合問題與改進方向

> 設計背景見 `daisy-nwdaf-design.md`

---

## 概述

NWDAF（Go）、Daisy（FL 訓練框架）、ML Service（Python 推論引擎）三者之間的整合目前存在幾個根本性問題：重訓期間 HTTP 阻塞、模型傳遞方式 hardcode、ML Service 無法動態載入模型定義、以及元件職責邊界不清晰。以下說明各問題現狀與改進方向。

---

## 1. HTTP 阻塞（重訓期間）

### 問題

NWDAF 向 Daisy 發送 POST `/publish_task` 後，會一直等待 200 回應才繼續執行。FL 訓練可能需要數分鐘，整個重訓邏輯（trigger → swap → restart monitor）在這段時間內完全被卡住。

### 改進方向

改為 async callback 模式：

- Daisy 收到 task 後立即回 `202 Accepted`，在 background thread 執行訓練
- 訓練完成後，Daisy 主動 POST `callback_url`（由 task payload 帶入），body 帶 `{task_id, model_url}`
- NWDAF 新增 `POST /mtlf/training-complete` endpoint，收到 callback 後繼續換版流程
- NWDAF 維護 in-flight 狀態表（`sync.Map`）：`task_id → {oldModelUrl, store, mtlfCfg}`，callback 收到後從中取出對應上下文

需同時確認：`IsRetraining()` guard 在 async 架構下仍能防止重複觸發。Flask 預設 single-thread，`app.run()` 須加 `threaded=True` 才能在訓練中同時處理 callback 以外的請求。

---

## 2. 模型傳遞（目前 hardcode file path）

### 問題

目前 NWDAF 從 task config 讀取 `MODEL_PATH`（Daisy 本地 file path），直接傳給 ML Service；ML Service 從該路徑讀取模型檔案。實際上 ML Service 讀的是預先放好的固定模型，Daisy 訓練完存下的 `model.npy` 並未被真正使用，是 hardcode 的形式。

### 改進方向

改為 HTTP URL 傳遞模式：

- Daisy 訓練完後，將模型打包成 tar.gz，透過 HTTP endpoint serve
- callback body 帶上 `model_url`（指向 tar.gz 下載連結）
- NWDAF 收到後將 `model_url` 轉發給 ML Service
- ML Service 自行 GET 下載、解壓、載入

`MODEL_PATH`（Daisy 本地路徑）維持不變，FL 框架仍用它做 warm start 和存訓練結果。打包和 serve 發生在訓練迴圈結束後，不影響訓練行為本身。

職責因此清楚分明：Daisy 管訓練與存模型；NWDAF 只做協調，不碰模型本身；ML Service 管下載、載入、推論。

---

## 3. 模型打包（三件套 + Scaler）

### 問題

ML Service 的模型載入目前是 hardcode 的：模型結構直接寫死在 Python code 裡，無法動態切換架構。若要換架構，需要改 ML Service 本身的程式碼，部署成本高，且無法通用。

### 改進方向

將「一個模型」定義為四個檔案的組合，打包成 tar.gz 一起傳遞：

| 檔案 | 用途 |
|------|------|
| `model.py` | 模型架構定義（PyTorch class），ML Service 直接 `from model import Model` |
| `config.yaml` | 超參數（建構 Model class 用）+ 推論介面（feature names、seq_length、output fields 等） |
| `model.npy` | 訓練後的模型權重 |
| `scaler.pkl` | 訓練時用的資料 Scaler（均值 / 標準差） |

ML Service 收到 `model_url` 後：GET 下載 tar.gz → 解壓到暫存目錄 → 從 `config.yaml` 讀超參數建構 Model class → 載入 `model.npy` 權重 → 載入 `scaler.pkl`。全部成功才視為載入完成（原子性），失敗直接 error，不進入半套狀態。

Daisy 打包時：`model.npy` / `scaler.pkl` 是訓練輸出，`model.py` / `config.yaml` 從 task payload 指定的路徑讀取（`MODEL_DEF`、`MODEL_CONFIG`），放在 `arch/<version>/` 目錄下，以 git 版本控制，不透過 FL 協議下發給 client。

> client 各自機器上需有對應版本的 `model.py`，架構改版時須透過部署腳本同步。

---

## 4. Daisy 初始化流程

### 問題

目前 Daisy 透過 `--init_model` flag 在部署時初始化模型，需要人工觸發，且與 task 裡的 `MODEL_PATH` 各自獨立，容易不一致。初始化邏輯與 task 觸發邏輯分離，也增加了部署步驟的複雜度。

### 改進方向

移除 `--init_model`，改為按需初始化：task 進來時若 `MODEL_PATH` 不存在，則自動呼叫初始化邏輯（`get_model()` + `get_dataloaders()`）。初始化邏輯以 callback 方式由 `master.py` 注入，保持 example-specific 的彈性，不把特定模型邏輯耦合進框架本身。

---

## 5. MTLF / AnLF 職責分離

### 問題

目前 `swapModelAfterRetrain`（`training.go:104`）直接呼叫 `mlClient.InitializeModel` / `mlClient.UnloadModel`，也就是 MTLF 直接操作 ML Service。這與設計原則（AnLF 負責所有 ML Service 互動）不符，且與 Phase 1 初始化路徑不一致。

同樣的問題存在於 `runDelayedTraining`（startup trigger）：啟動時直接呼叫 ML Service 載入初始模型，繞過了 AnLF。

### 改進方向

- `swapModelAfterRetrain` 只負責更新 SharedModelRegistry 和訂閱狀態，不直接操作 ML Service
- 透過 `onModelSwapReady(newModelUrl, oldModelId string)` callback 通知 AnLF，由 AnLF 執行 load new → update registry → unload old 的完整換版流程
- `runDelayedTraining` 改為通知 AnLF 載入 static_model_url，與 Phase 1 初始化路徑對齊

解耦方式：在 `MtlfService` 初始化時注入 `onModelSwapReady` callback，由 AnLF 實作。

---

## 6. SharedModelRegistry Key 設計（後續）

### 問題

目前 `SharedModelRegistry` 以 model URL 為 key，無法支援多版本並存（active / standby），也就無法實作 rollback。

### 改進方向

改以 **series ID**（例如 `"ue-comm-group1"`）為 key，版本記錄（URL、status、model_id）下放到 value 內：

```
key: series_id（穩定，不隨重訓改變）
value:
  versions:
    - version: 1, url: "http://daisy/.../v1.tar.gz", status: standby, model_id: "uuid-1"
    - version: 2, url: "http://daisy/.../v2.tar.gz", status: active,  model_id: "uuid-2"
```

影響範圍：`NWDAFContext.sharedModels`、`SharedModelInfo` 結構、`MlModelInfo.ModelUrl` → `SeriesId`、`training.go` / `trigger.go` / `monitor.go`。

此項優先級較低，建議前五項完成後再處理。

---

## 實作優先順序

| # | 項目 | 元件 | 說明 |
|---|------|------|------|
| 1 | HTTP Blocking → async callback | Daisy | 後續所有改動的基礎 |
| 2 | 模型打包 + serve tar.gz | Daisy | `server_api_handler.py`，依賴 `MODEL_DEF`/`MODEL_CONFIG` key |
| 3 | ML Service 動態載入 | ML Service (Python) | 從本地 path 改為 HTTP 下載 + 動態載入 model.py |
| 4 | NWDAF callback endpoint | NWDAF (Go) | 接收 Daisy 通知，驅動換版流程 |
| 5 | MTLF/AnLF 職責分離 | NWDAF (Go) | 解耦 ML Service 直接呼叫 |
| 6 | Daisy 初始化流程 | Daisy | 移除 `--init_model` |
| 7 | SharedModelRegistry Key | NWDAF (Go) | 後續架構優化 |
