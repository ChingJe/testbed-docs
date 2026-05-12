# NWDAF Retrain Replay Current Setup

**Date:** 2026-05-12
**Scope:** 記錄目前 retrain replay 主線實驗的主要設置、testbed 依賴項、以及 Daisy / dataset / initial-local trainer 的本地修改點。

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

目前需要同時記住兩條線：

- `exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler`
- `exp46_local_initial_v6_freshinit_fedscaler_es4_lrpat2`

其中：

- `exp27_*_sharedscaler`
  - 歷史上最重要的 shared-scaler baseline
  - 用來驗證 shared scaler + `retrain.window_sec=1200` 是否能把第二次 `CAT2→CAT3` trigger 拉回來
- `exp46_*_freshinit_fedscaler_*`
  - 目前最值得延續的 fresh-init 主線
  - 初始模型使用目前重建的 `initial_local_cat1_30s_v6`
    - 語意上維持 `CAT1-trained + CAT1 scaler`
  - Daisy retrain 仍走舊主線：
    - fresh init per task
    - per-task federated aggregated scaler
  - 在不破壞兩次 CAT-switch retrain 的前提下，對 Daisy task 參數做最小調整：
    - `LR_PATIENCE 3 -> 2`
    - `ES_PATIENCE 5 -> 4`

兩者都能在兩次 CAT 切換時觸發 retrain，但語意不同：

- `exp27`
  - 第一次 trigger：`00:34:30 group2 degradation`
  - 第二次 trigger：`01:06:00 group1 degradation`
- `exp46`
  - 第一次 trigger：`00:34:30 group2 degradation`
  - 第二次 trigger：`01:07:30 group1 degradation`

也就是：

- `exp27` 仍是最重要的歷史 baseline
- `exp46` 是目前較值得延續的 fresh-init current setup

另外這次實驗已確認：

- `exp29`（`v3 initial bundle`）不會觸發 retrain
- `exp33`（`v4 initial bundle` + `zScoreThreshold=1.5`）可恢復第一次 trigger，但第二次只累積到 `2/5 hits`
- `exp36`（`v5 initial bundle + federated scaler`）證明 oracle-scaler baseline 有效，但不足以單獨改變 retrain 結構
- `exp37`（`fixed scaler + continue learning`）技術路徑已可運作，但尚未穩定優於 fresh-init 路線
- `exp41` ~ `exp44` 顯示：
  - 過於激進的 early stopping / 降 learning rate 會讓第二次 retrain 消失
- `exp45` 是第一個不破壞兩次 retrain 條件、且明顯改善 `CAT3` second-model 表現的 fresh-init 調整
- `exp46` 進一步在 `CAT2` / `CAT3` 表現與 retrain wall time 之間取得較佳平衡

---

## 當前主要設置

### A. Initial bundle

目前主線不是用預設 initial bundle，而是用本地 supervised trainer 產生的：

- 歷史 baseline：`out/initial_local_cat1_30s_v2/bundle`
- 歷史 aligned-line 主線：`out/initial_local_cat1_30s_v4/bundle`
- 目前 fresh-init 主線：`out/initial_local_cat1_30s_v6/bundle`

`v4` 延續 `v2` 的主要超參數，但把 scaler fit timing 改成與 Daisy federated scaler 語意更接近：

- centralized local training
- `CAT1 only`
- `group1 + group2`
- `period=30s`
- `seq_length=30`
- `train_ratio=0.9`
- `batch_size=8`
- `learning_rate=5e-4`
- scaler：先用各 group 全部 CAT1 rows fit，再做 train/val split

`v6` 則是在目前 `train_initial_local.py` 上，重新按 `v4` 語意訓練出的 bundle：

- `CAT1-trained model`
- `CAT1-fitted scaler`
- `train_ratio=0.9`
- `batch_size=8`
- `learning_rate=5e-4`

目前用途是：

- 作為 fresh-init 調參線的穩定對照起點
- 與 `v5`（oracle scaler）刻意分開，避免把 initial bundle 改善和 Daisy retrain 調參混在一起

`v4` 的本地 validation 明顯優於先前 exploratory 的 `v3`：

- `best_epoch=18`
- `final_val_loss=0.023509`
- validation `WAPE=0.1837`
- validation `NRMSE=0.2169`

### B. Accuracy monitor

目前主線 monitor 設置如下：

| 項目 | `exp27` | `exp46` |
|---|---:|---:|
| `samplingInterval` | `30s` | `30s` |
| `report_period_sec` | `30s` | `30s` |
| `checkInterval` | `90s` | `90s` |
| `maturity_lag_sec` | `60s` | `60s` |
| `recentBufferSize` | `12` | `12` |
| `minBufferSamples` | `5` | `5` |
| `minSamples` | `2` | `2` |
| `primaryMetric` | `WAPE` | `WAPE` |
| `zScoreThreshold` | `1.5` | `1.4` |
| `minStd` | `0.14` | `0.14` |
| `fixedFloor` | `0.05` | `0.05` |
| `decisionWindowSize` | `5` | `5` |
| `requiredHitsInWindow` | `3` | `3` |
| `chronicPolicy.enabled` | `false` | `false` |
| `lowTrafficOverpredictionPolicy.enabled` | `false` | `false` |

這組設置的意圖仍然是：

- 保留 `90s` 的偵測速度
- 不讓 `minSamples` 貼近 `3 slots / round` 的理論上限，保留 testbed 誤差空間
- 關掉 chronic，避免在目標之外的時間點重複 retrain
- 對於目前 aligned line，優先透過微調 `zScoreThreshold`，而不是改 `requiredHitsInWindow`

### C. Retrain window

目前主線實驗都使用：

- `retrain.window_sec = 1200`

目前 `1200s` 仍是可用的較短窗口：

- `900s` 會讓 Daisy client dataset 樣本數不足
- `1200s` 可正常訓練，且在 `exp27` / `exp34` 都能拉回第二次 trigger

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
- `MONGO_URI = mongodb://127.0.0.1:27018`

這表示 retrain training docs 並不是寫到預設 `27017`，而是 `27018`。  
這次已實際踩到：

- 若 Daisy 端錯設成 `127.0.0.1:27017`
- 或 VM-side config 仍指向錯的 host/port

`/upload_data` 會直接回 `500`，Daisy retrain 無法開始。

目前本地 / VM 的語意應記成：

- 本地 replay / Daisy example：`127.0.0.1:27018`
- VM 內 NAT 路徑：`10.0.2.2:27018`

注意：repo 中有些 NWDAF `nwdafcfg.yaml` / 舊實驗輸出仍記著 `127.0.0.1:27017`，這不代表它是目前這台 testbed 的正確 Mongo port。

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

### 3. Scaler lifecycle 已往 DLinear branch 對齊

目前最重要的 Daisy 本地修改，涉及：

- `dataset.py`
- `server_api_handler.py`
- `client.py`
- `custom_strategy.py`

目前**有效行為**是：

- round 1 不做正式模型訓練，而是讓 client 各自計算 local scaler stats
  - `mean / var / n`
- master strategy 聚合 local stats，寫出單一 `model/{tid}/scaler.pkl`
- round 2 之後 client 重新載入 dataloader，正式訓練
- 所有 client 最終都使用同一份 shared scaler
- 這個方向是往 `origin/NWDAF-daisy-Dlinear` 的 `client local stats -> master aggregation` 對齊

這條線的工程背景要分清楚：

- 舊的 master-side shared scaler refit 是暫時性止血方案
- 但它會把不同 group 混在一起再按 timestamp 聚合，語意上不正確
- 目前 aligned line 不再把那個 temporary mitigation 當正式方向
- 仍可能保留部分 server-side 相容性邏輯，但最終有效 scaler 由 stats round 覆蓋

這個修改對結果有明顯影響：

- `exp29`（`v3 initial` + aligned scaler）不會觸發 retrain
- `exp33`（`v4 initial` + aligned scaler + `z=1.5`）恢復第一次 trigger
- `exp34`（`v4 initial` + aligned scaler + `z=1.4`）恢復兩次 trigger

若另一套 Daisy 沒有這個修改，post-retrain 行為不應直接與目前結果比較。

### 4. Daisy example Python 相容性修補

這次 aligned line 另外確認到：

- Daisy example venv 的 Python 比目前本地 shell 環境舊
- `dataset.py` 若直接使用新版 annotation 語法，master 會在 import 階段失敗

目前本地已做最小相容性修補：

- `dataset.py` 加入 `from __future__ import annotations`
- 避免 `str | None` / `list[...]` 這類 annotation 在舊 Python 下炸掉

這不改變訓練邏輯，但若另一套環境 Python 版本不同，這個邊界要一起記住。

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

### 3. `train_initial_local.py`

目前 initial local trainer 本身也已經是主線依賴的一部分，因為：

- 它不再只是產生 historical `v2` bundle 的工具
- `v4` 已成為目前 aligned line 的主要 initial bundle

目前要點：

- 使用 `CAT1`、`group1 + group2`
- `train_ratio=0.9`
- `batch_size=8`
- `learning_rate=5e-4`
- scaler fit timing 已改成：
  - 先以各 group 全部 CAT1 rows fit scaler
  - 再做 train/val split

若另一套環境沿用舊腳本邏輯，initial model 與目前 aligned line 不應直接比較。

---

## 目前可用的對照方式

若要快速確認某個環境是否和目前主線一致，最有用的是直接看：

### Daisy

- `git diff -- examples/07_MTLF_training/client.py`
- `git diff -- examples/07_MTLF_training/custom_strategy.py`
- `git diff -- examples/07_MTLF_training/dataset.py`
- `git diff -- examples/07_MTLF_training/master.py`
- `git diff -- src/py/daisyfl/common/task_launcher.py`
- `git diff -- src/py/daisyfl/master/server_api_handler.py`
- `git diff -- examples/07_MTLF_training/daisyconfig.json`

### NWDAF

- `git diff -- config/nwdafcfg.yaml`
- `git diff -- tools/retrain_replay/replay_config.yaml`
- `git diff -- tools/retrain_replay/train_initial_local.py`

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
   - client-local-stats -> master-global-scaler aggregation
   - Python annotation compatibility fix
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
6. initial bundle 是否使用目前主線的 `initial_local_cat1_30s_v4`
7. 若要重現 aligned line，policy 是否使用：
   - `zScoreThreshold=1.4`
   - `requiredHitsInWindow=3`

---

## 當前判斷

目前這條主線已經不是單純「調 monitor 參數」而已，而是依賴：

- 特定 initial bundle（目前建議 `v4`）
- 新的 `group2` sequential dataset
- Daisy custom strategy / local epochs 改動
- Daisy federated scaler aggregation path
- Daisy / Mongo 27018 邊界
- Daisy example Python 相容性修補

其中最關鍵的本地修改現在要分成兩類看：

- 歷史 baseline 關鍵：
  - `exp27` 的 shared-scaler line
- 目前 aligned line 關鍵：
  - `v6 initial bundle`
  - fresh init + federated aggregated scaler
  - `zScoreThreshold=1.4`

就目前判斷來看：

- `exp34` 已經把兩次 CAT 切換 retrain 都拉回來
- `exp46` 則是目前 fresh-init 線下最佳已知 Daisy task 設定
  - `INITIAL_LR = 1e-3`
  - `LR_PATIENCE = 2`
  - `ES_PATIENCE = 4`
  - `local_epochs = 3`
  - 保住兩次 retrain
  - `CAT2` 表現接近 `exp40`
  - `CAT3` second-model 明顯優於 `exp40`
  - retrain wall time 也優於 `exp45`

因此目前更合理的定位是：

- `exp27`：最重要的 historical best shared-scaler baseline
- `exp36` / `exp37`：oracle-scaler / continue-learning 參考線
- `exp46`：目前最值得延續的 fresh-init current setup
