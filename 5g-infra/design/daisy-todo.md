# Daisy — 待辦項目

> 設計背景見 `daisy-nwdaf-design.md`
> Daisy 是 submodule，改完需要另外在 Daisy repo commit。

---

## 前置工作

- 建立 `config.yaml`（參照 `daisy-nwdaf-design.md` Model 定義章節）
- 整理 `arch/v1/` 目錄結構，放入 `config.yaml`；`model.py` 留在根目錄不動

---

## 功能項目

### 1. 移除 `--init_model`，改為 task 進來時按需初始化

目前 `master.py --init_model` 在部署時將模型存至固定路徑，與 task 的 `MODEL_PATH` 各自獨立，需人工保持一致。更根本的問題是：初始化邏輯應該由 task 觸發，而不是部署時決定。

- `server_api_handler.py` 在 `receive_task` 前檢查 `MODEL_PATH` 是否存在，不存在則呼叫初始化
- 初始化邏輯（`get_model()` + `get_dataloaders()`）是 example-specific，以 callback 方式由 `master.py` 注入給 `ServerListener`
- `--init_model` flag 可移除

### 2. task.json — 新增架構定義 key

- 加入 `MODEL_DEF`（指向 `model.py`）和 `MODEL_CONFIG`（指向 `arch/v1/config.yaml`）

### 2. Async callback

- `publish_task` 改為非阻塞：立即回 202 + task_id，background 執行訓練
- 訓練完成後 POST `callback_url`（從 task payload 讀取），body 帶 `task_id` 和 `model_url`

### 3. 模型打包與 serve

- 訓練完成後將 `model.npy`、`scaler.pkl`、`model.py`、`config.yaml` 打包成 `model/<TID>.tar.gz`
- 新增 `GET /model/<filename>` endpoint serve 打包結果

### 4. 一鍵部署腳本

- 擴充 `nodes.yaml`，加入各 node 的 SSH 資訊與工作目錄
- 新增 `distribute.py`，一次將 `arch/` 目錄同步到 master 和所有 client 機器

---

## 注意事項

- `MODEL_DEF` / `MODEL_CONFIG` 只供 master 打包使用，不透過 FL 協議下發給 client
- Client 各自機器上需有對應版本的 `model.py`，架構改版時需透過 `distribute.py` 同步
- `model/model.npy` 是 FL 框架 input/output 共用路徑，不要改
- Race condition（同時多個 retrain）由 NWDAF 側負責，Daisy 不需要處理
