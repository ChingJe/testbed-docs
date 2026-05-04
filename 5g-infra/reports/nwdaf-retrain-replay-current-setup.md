# NWDAF Retrain Replay Current Setup

**Date:** 2026-05-04
**Scope:** 記錄目前 retrain replay 主線實驗的主要設置、testbed 依賴項、以及 Daisy / dataset 的本地修改點。

---

## 用途

這份文件回答三件事：

1. 目前 replay / testbed 主線是用什麼設定在跑
2. 哪些行為依賴本地修改過的 Daisy 或資料集
3. 若要在另一套 testbed 重現同樣效果，至少要對齊哪些設定

本文件偏向「當前狀態記錄」，不是最終設計文件。  
scaler 問題與修正方案另見：

- [daisy-retrain-scaler-issue.md](daisy-retrain-scaler-issue.md)
- [../design/daisy/daisy-global-scaler-refit-plan.md](../design/daisy/daisy-global-scaler-refit-plan.md)

---

## 當前主線

### 目前最重要的兩組實驗

目前主線比較基準是：

- `exp25_local_initial_buf5_nochronic_sharedscaler`
- `exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler`

其中：

- `exp25_*_sharedscaler`
  - 用來驗證 shared scaler 修正後，第一次 retrain 後的整體狀態是否變乾淨
- `exp27_*_sharedscaler`
  - 用來驗證 shared scaler + `retrain.window_sec=1200` 是否能把第二次 `CAT2→CAT3` trigger 拉回來

截至目前，`exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler` 是最有資訊量的結果：

- 第一次 trigger：`00:34:30 group2 degradation`
- 第二次 trigger：`01:06:00 group1 degradation`

也就是 shared scaler 修正後，`CAT2→CAT3` 的第二次 degradation trigger 已重新出現。

---

## 當前主要設置

### A. Initial bundle

目前主線不是用預設 initial bundle，而是用本地 supervised trainer 產生的：

- `out/initial_local_cat1_30s_v2/bundle`

對應訓練思路：

- centralized local training
- `CAT1 only`
- `group1 + group2`
- `period=30s`
- `seq_length=30`
- `train_ratio=0.9`
- `batch_size=8`
- `learning_rate=5e-4`

用途是提供較穩的 replay 起始模型，避免一開始就完全依賴 Daisy FL 訓練。

### B. Accuracy monitor

目前主線 monitor 設置大致如下：

| 項目 | 目前值 |
|---|---|
| `samplingInterval` | `30s` |
| `report_period_sec` | `30s` |
| `checkInterval` | `90s` |
| `maturity_lag_sec` | `60s` |
| `recentBufferSize` | `12` |
| `minBufferSamples` | `5` |
| `minSamples` | `2` |
| `primaryMetric` | `WAPE` |
| `zScoreThreshold` | `1.5` |
| `minStd` | `0.14` |
| `fixedFloor` | `0.05` |
| `decisionWindowSize` | `5` |
| `requiredHitsInWindow` | `3` |
| `chronicPolicy.enabled` | `false` |
| `lowTrafficOverpredictionPolicy.enabled` | `false` |

這組設置的意圖是：

- 保留 `90s` 的偵測速度
- 不讓 `minSamples` 貼近 `3 slots / round` 的理論上限，保留 testbed 誤差空間
- 關掉 chronic，避免在目標之外的時間點重複 retrain

### C. Retrain window

目前主要比較兩組：

| 實驗 | `retrain.window_sec` |
|---|---|
| `exp25_*_sharedscaler` | `1800` |
| `exp27_*_sharedscaler` | `1200` |

目前 `1200s` 是可用的較短窗口：

- `900s` 會讓 Daisy client dataset 樣本數不足
- `1200s` 可正常訓練，且在 shared scaler 修正後能重新出現第二次 trigger

### D. Daisy task

目前 NWDAF task config 會帶這些重要參數：

| 項目 | 目前值 |
|---|---|
| `NUM_ROUNDS` | `30` |
| `ES_PATIENCE` | `5` |
| `LR_PATIENCE` | `3` |
| `EVALUATE_INTERVAL` | `1` |
| `MODEL_META.inference.seq_length` | `30` |
| `MODEL_META.inference.out_seq_len` | `1` |
| `MODEL_META.training.local_epochs` | `3` |

另外，目前 Daisy custom strategy / client 已支援幾個額外欄位，雖然不一定每次都顯式寫在 config 內，但在 task 語意上已經是有效參數：

| 欄位 | 意義 |
|---|---|
| `ES_PATIENCE` | early stopping patience。validation loss 連續多少 round 沒改善，就提前結束這次 task。 |
| `LR_PATIENCE` | learning rate plateau patience。validation loss 連續多少 round 沒改善，就把目前 learning rate 減半。 |
| `INITIAL_LR` | custom strategy 一開始下發給 client 的 learning rate。若未指定，strategy 會用預設值。 |
| `NUM_CLIENTS_FIT` | 每個 fit round 預期參與訓練的 client 數。主要用來控制 strategy 對 client 數量的假設。 |
| `NUM_CLIENTS_EVALUATE` | 每個 evaluate round 預期參與驗證的 client 數。主要用來控制 strategy / metrics 聚合時對 evaluate client 數量的假設。 |

---

## Testbed 依賴項

這條實驗線不是只改 YAML 就能重現，還依賴幾個 testbed 前提。

### 1. `group2` 資料已換成新的 sequential 版本

目前 `go-upf` 這邊有修改過：

- `pre_data/group2/training_packets_run001.parquet`

這個檔案是 parquet，`git status` 可看出被修改，但不適合用文字 diff 檢視。  
從實驗脈絡來看，`exp22` 之後這條線已經假設：

- `group2` 使用新的 sequential dataset

若另一套 testbed 沒有這個資料版本，實驗結果不應直接對比。

### 2. Daisy example 使用自己的 Mongo port

目前 Daisy example config 是：

- `daisyconfig.json`
- `MONGO_URI = mongodb://localhost:27018/`

這表示 retrain training docs 並不是寫到預設 `27017`，而是 `27018`。  
若另一套 testbed Daisy / Mongo 沒對齊，client 會找不到 training docs。

### 3. Replay 指向 Daisy example venv

目前 `replay_config.yaml` 裡 `daisy.python_bin` 已寫成 testbed 本地的 Daisy example venv。  
若換環境，需要至少對齊：

- Daisy example 的 python 執行路徑
- 該 venv 內的套件版本

### 4. CAT transition 標記

目前 `replay_config.yaml` 已記：

- `CAT1→CAT2` at `1800s`
- `CAT2→CAT3` at `3600s`

這主要是為了 report / analysis 對齊，不直接影響訓練，但若另一套 replay timeline 不同，結果解讀方式也要跟著改。

---

## Daisy 本地修改

這裡只記與目前主線直接相關、且在 testbed 上有實際效果的改動。

### 1. Custom strategy 可由 task config 控制

目前 Daisy 不是用固定寫死的 early stopping / LR schedule，而是允許由 task config 控制：

- `custom_strategy.py`
- `task_launcher.py`

主要差異：

- `ES_PATIENCE`
- `LR_PATIENCE`
- `INITIAL_LR`
- `NUM_CLIENTS_FIT`
- `NUM_CLIENTS_EVALUATE`

現在 `task_launcher.py` 會把 `task_config` 傳入 `EarlyStoppingFedAvg`。  
若沒有這個修改，NWDAF `nwdafcfg.yaml` 裡那些 patience 設定不會真正進 Daisy strategy。

這些欄位的語意可簡化理解為：

- `ES_PATIENCE`
  - 控制「何時整個 retrain task 提前停掉」
- `LR_PATIENCE`
  - 控制「何時把學習率往下降」
- `INITIAL_LR`
  - 控制「本次 task 一開始 client 拿到的 learning rate」
- `NUM_CLIENTS_FIT`
  - 控制「strategy 預期每輪有幾個 client 參與 fit」
- `NUM_CLIENTS_EVALUATE`
  - 控制「strategy 預期每輪有幾個 client 參與 evaluate」

### 2. Client 的 `local_epochs` 可由 `MODEL_META.training` 控制

目前 `client.py` 已修改成：

- 從 `MODEL_META.training.local_epochs` 讀取本地 epoch 數

因此 NWDAF task config 裡的：

- `MODEL_META.training.local_epochs: 3`

會真正影響 Daisy client。  
若沒有這個修改，client 會退回固定值，無法從 NWDAF config 控制。

### 3. Shared scaler refit

這是目前最重要的 Daisy 本地修改，涉及：

- `dataset.py`
- `server_api_handler.py`

目前行為：

- master 在 `publish_task` 後、spawn clients 前
- 先對同一個 `tid` 的所有 training docs 做一次全域 `log1p + StandardScaler.fit`
- 產生單一 `model/{tid}/scaler.pkl`
- 所有 client 都只讀這一份 scaler
- client 不再各自 fit 並覆蓋 `scaler.pkl`

這個修改對結果有明顯影響：

- `exp25` shared scaler 版比舊版整體更乾淨
- `exp27` shared scaler 版重新出現第二次 `CAT2→CAT3` degradation trigger

若另一套 Daisy 沒有這個修改，post-retrain 行為不應直接與目前結果比較。

---

## NWDAF 這邊目前有依賴的設置

### 1. `nwdafcfg.yaml`

目前 task 區塊除了原本的 `NUM_ROUNDS` 外，還依賴：

- `ES_PATIENCE`
- `LR_PATIENCE`
- `MODEL_META.training.local_epochs`

另外目前 strategy 指向：

- `MASTER_STRATEGY: ["custom_strategy", "EarlyStoppingFedAvg"]`

也就是 Daisy example 端必須真的存在這份 `custom_strategy.py`，且支援 task-config-driven strategy construction。

### 2. `replay_config.yaml`

目前 replay 假設：

- `daisy.reuse_existing = true`
- `daisy.python_bin` 指向 Daisy example venv
- `analysis.catTransitions` 已填入 1800/3600 秒

這些雖然不全是訓練邏輯本身，但都屬於目前 testbed 主線的一部分。

---

## 目前可用的對照方式

若要快速確認某個環境是否和目前主線一致，最有用的是直接看：

### Daisy

- `git diff -- examples/07_MTLF_training/client.py`
- `git diff -- examples/07_MTLF_training/custom_strategy.py`
- `git diff -- examples/07_MTLF_training/dataset.py`
- `git diff -- src/py/daisyfl/common/task_launcher.py`
- `git diff -- src/py/daisyfl/master/server_api_handler.py`
- `git diff -- examples/07_MTLF_training/daisyconfig.json`

### NWDAF

- `git diff -- config/nwdafcfg.yaml`
- `git diff -- tools/retrain_replay/replay_config.yaml`

### Dataset

- `git status --short pre_data/group2/training_packets_run001.parquet`

最後這個只能確認檔案有沒有被換過，不能直接看內容差異。

---

## 建議的重現順序

若要在另一套 testbed 上盡量重現目前主線結果，建議至少依序確認：

1. `group2` 是否已換成新的 sequential dataset
2. Daisy 是否使用相同 Mongo port
3. Daisy 是否包含：
   - task-config-driven custom strategy
   - task-config-driven `local_epochs`
   - shared scaler refit
4. NWDAF task config 是否帶入：
   - `ES_PATIENCE`
   - `LR_PATIENCE`
   - `MODEL_META.training.local_epochs`
5. replay 是否使用：
   - `30s` period
   - `90s` check interval
   - `minSamples=2`
   - `minBufferSamples=5`
   - `WAPE`
   - `chronic=false`
6. initial bundle 是否使用 `initial_local_cat1_30s_v2`

---

## 當前判斷

目前這條主線已經不是單純「調 monitor 參數」而已，而是依賴：

- 特定 initial bundle
- 新的 `group2` sequential dataset
- Daisy custom strategy / local epochs 改動
- Daisy shared scaler refit

其中最關鍵的本地修改是：

- **shared scaler refit**

因為它已經被證明會直接改變 retrain 後的模型狀態與後續 transition detectability。
