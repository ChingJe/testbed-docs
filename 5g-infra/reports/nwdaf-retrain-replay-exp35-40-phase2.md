# NWDAF Retrain Replay Exp35-40 Notes

**Date:** 2026-05-11  
**Scope:** 記錄 `exp35` 到 `exp40` 的 scaler / continue-learning 實驗結果，並整理目前可成立的初步結論。

---

## 用途

這份文件回答三件事：

1. `v5` initial bundle 與 fixed-scaler baseline 帶來了什麼差異
2. Daisy phase 2 的 `fixed scaler + continue learning` 實際跑起來後結果如何
3. 目前為止，為什麼多次迭代沒有穩定優於舊有 fresh-init 路線

本文件是分析紀錄，不取代主線設置說明。當前配置仍以：

- [nwdaf-retrain-replay-current-setup.md](./nwdaf-retrain-replay-current-setup.md)

為準。

---

## 實驗矩陣

### `exp35`

- baseline：`v4 initial bundle`
- Daisy scaler：per-task federated aggregated scaler
- retrain：仍是舊主線行為

### `exp36`

- baseline：`v5 initial bundle`
- `v5 = CAT1-trained model + full-data oracle scaler`
- Daisy retrain 其餘設置與 `exp35` 一致
- 用途：驗證 fixed full-data scaler baseline 本身的影響

### `exp37`

- baseline：`v5 initial bundle`
- Daisy：`fixed scaler + continue learning`
- 第一次 retrain 從 `v5 bundle/model.npy` warm-start
- 第二次 retrain 從上一輪 Daisy 本地 `model/<prev_tid>/model.npy` warm-start
- 兩次 retrain 都固定使用 `v5 bundle/scaler.pkl`

### `exp38`

- 延續 `exp37`
- 只調整：
  - `INITIAL_LR: 1e-3 -> 5e-4`

### `exp39`

- 延續 `exp38`
- 再調整：
  - `MODEL_META.training.local_epochs: 3 -> 1`

### `exp40`

- regression check after the recent replay / Daisy bridge changes
- initial bundle：`v6`
- `v6 = current train_initial_local.py` 重新訓練的 `v4-style` bundle
  - `CAT1-trained model`
  - `CAT1-fitted scaler`
- Daisy retrain：
  - `use_fixed_scaler = false`
  - `enable_continue_learning = false`
  - 每次 task 都回到 fresh init + federated aggregated scaler
- 用途：確認近期 phase 1 / phase 2 改動沒有污染舊 retrain 路線

---

## Initial Bundle 結果

`v5` 是 phase 1 產出的新 initial bundle：

- training rows：仍只用 `CAT1`
- scaler rows：改為整段 replay 全資料

相較 `v4`，`v5` 的 supervised validation 明顯較好：

| Bundle | Val Loss | WAPE | NRMSE |
|---|---:|---:|---:|
| `v4` | `0.023509` | `0.183668` | `0.216852` |
| `v5` | `0.012205` | `0.137686` | `0.187332` |

這表示：

- full-data oracle scaler 作為 offline baseline 是有效的
- 但它只證明 initial bundle 本身更穩，不等於 replay / retrain 行為一定同步改善

---

## 實驗摘要

| Exp | 主要差異 | Trigger 1 | Trigger 2 | Swap 1 | Swap 2 | Notes |
|---|---|---|---|---|---|---|
| `exp35` | `v4` + federated scaler | `00:34:30 group2` | `01:07:30 group1` | `00:36:23` | `01:17:25` | clean federated-scaler baseline |
| `exp36` | `v5` + federated scaler | `00:34:30 group2` | `01:07:30 group1` | `00:36:23` | `01:15:08` | oracle scaler baseline |
| `exp37` | `v5` + fixed scaler + continue learning | `00:34:30 group2` | `01:04:30 group1` | `00:36:43` | `01:06:26` | phase 2 main run |
| `exp38` | `exp37` + lower LR | `00:34:30 group2` | `01:04:30 group1` | `00:37:03` | `01:06:43` | LR reduced to `5e-4` |
| `exp39` | `exp38` + fewer local epochs | `00:34:30 group2` | `01:04:30 group1` | `00:37:44` | `01:07:03` | `local_epochs=1` |
| `exp40` | `v6` + fresh-init federated scaler | `00:34:30 group2` | `01:07:30 group1` | `00:36:23` | `01:16:28` | regression check for old path |

所有實驗的 replay trace summary 都是：

- `slots=362`
- `predictions=302`
- `retrainJobs=2`

差異主要落在：

- second trigger 時間
- Daisy 訓練耗時
- hot swap 提前或延後
- retrain 後部署模型的區段表現

---

## 關鍵觀察

### 1. `exp36` 證明 oracle scaler baseline 有效，但影響有限

`exp36` 相對 `exp35`：

- 兩次 trigger 時間相同
- 流量軌跡相同
- second swap 較早完成

解讀上較合理的是：

- `v5` initial bundle 本身比 `v4` 更穩
- 但這個改善還不足以改變 replay 上的 trigger 結構

也就是：

- oracle scaler baseline 有價值
- 但它不是足以單獨扭轉整體 retrain 行為的因素

### 2. `exp37` 證明 phase 2 路線可以真正跑通

在修正 Daisy runtime import path 後，`exp37` 已確認：

- 第一次 retrain 確實從 `v5 bundle/model.npy` seed
- 第二次 retrain 確實從上一輪 Daisy 本地 `model/<prev_tid>/model.npy` seed
- 兩次 retrain 都固定使用 `v5 bundle/scaler.pkl`
- Daisy 不再走 per-task scaler aggregation

這表示 phase 2 的技術路徑已經接通，不再只是 replay 端的 metadata 模擬。

### 3. `exp37` 的 second trigger 提前，但仍在正確 CAT 切換區段後

`exp37` 的 second trigger 從 `01:07:30` 提前到 `01:04:30`。

這不是切換前誤觸發，因為：

- `CAT2 -> CAT3` 標記在 `01:00:00`
- `exp37` second trigger 仍在切換後 `4m30s`

因此它比較像：

- 更早偵測到 drift
- 更早開始 retrain
- 更早完成 second swap

但這不代表 continue learning 已經全面優於舊方法。更細看後會發現：

- 第一輪 retrain 後部署到 `CAT2` 的模型，沒有比 `exp36` 更好
- 第二輪 retrain 後部署到 `CAT3` 的模型，`exp37` 的早段實際運行效果較佳

目前較合理的判讀是：

- `exp37` 的優勢主要來自 second retrain 提前啟動並提早完成 hot swap
- 不是因為第一輪 retrain 模型本身更強

### 4. `exp38` 與 `exp39` 不支持「單純調 optimizer 就能改善 phase 2」

`exp38`：

- 降低 learning rate 後，loss 曲線稍微平一點
- 但部署後模型表現沒有穩定優於 `exp37`

`exp39`：

- 再把 `local_epochs` 降到 `1`
- 整體比較像更新不足，部署後表現反而退步

所以目前看起來：

- `exp39` 可以排除
- `exp38` 只有局部微調效果，不足以取代 `exp37`
- phase 2 暫時沒有因簡單超參數調整而得到更穩的收益

### 5. `exp40` 確認舊 fresh-init 路線仍然存在

`exp40` 的目的不是追求更好的 initial bundle，而是驗證：

- 在目前 code 下，如果 `use_fixed_scaler` / `enable_continue_learning` 都關掉
- replay / Daisy 是否真的會回到舊 fresh-init 路線

這次結果可以確認：

- 兩次 retrain 的 replay log 都是 `seed_model=None`、`seed_scaler=None`
- second trigger 回到 `01:07:30`
- first retrain duration 回到 `113s` 級別
- second retrain duration 也回到長訓練型態，而不是 phase 2 的短 warm-start 型態

因此目前較合理的結論是：

- phase 2 改動沒有污染舊邏輯
- `retrain_replay.py` 的新 flags 已形成清楚的分流：
  - flags 關閉時：fresh init + federated scaler
  - flags 開啟時：fixed scaler + continue learning

---

## 初步結論

### A. 目前沒有證據顯示 Daisy phase 2 實作本身接錯

目前已確認：

- fixed scaler mode 實際生效
- warm-start lineage 實際生效
- Daisy runtime 已吃到 workspace 中修改過的 source tree

因此到目前為止，`exp37` 到 `exp39` 沒有穩定優於 `exp36`，不應優先解讀成「continue learning 沒真正跑到」。

### B. 更可能的原因是資料 regime 本身不利於 continual learning

目前 replay 場景有三個特徵：

1. `CAT1 -> CAT2 -> CAT3` 是明確的階段式切換，不是平滑 drift
2. 每次 retrain window 只有 `1200s`，有效樣本量很小
3. 舊模型若帶有前一個 CAT 的偏置，小窗口內未必能把這個偏置修正掉

對 continue learning 來說，這種情況比較不理想。較合理的高層判讀是：

- warm-start 不一定是資產
- 在 abrupt regime switch 下，它也可能成為包袱

### C. 多 group 混訓不一定是錯，甚至可能是合理設計

目前 retrain 雖由單一 `scope` 觸發，但實際上使用該時間窗口內所有 group 的資料一起訓練。

這在目前資料設計下未必是問題，因為：

- 同一時間點各 group 本來就處在相同 CAT
- 增加 group 可有效增加樣本量
- 模型學到的可能是該 CAT 的共通結構，而不是某個 group 的偶然噪音

因此目前不應草率把「全 group 混訓」直接判成錯誤；它比較像是：

- 對小資料 regime 的一種必要補強

### D. 目前較務實的結論

截至 `exp40`：

- `v5` initial bundle 值得保留
- `exp36` 是穩定的 oracle-scaler baseline
- `exp37` 是 phase 2 可運行的主線，但尚未證明明確優於 `exp36`
- `exp38`、`exp39` 不支持繼續只靠調整 `lr / local_epochs` 來救 phase 2
- `exp40` 則確認目前程式仍可乾淨地跑回舊 fresh-init 路線

所以接下來若要繼續驗證，優先順序應該是：

1. 先把問題定義成「這個資料 regime 是否真的適合 continual learning」
2. 再決定是否值得繼續沿 phase 2 調參

而不是先假設繼續調 optimizer 就能得到結構性改善。

---

## 後續建議

### 1. 做乾淨的 scratch vs warm-start 對照

在同一個 retrain window 下固定：

- 同一份 training docs
- 同一份 scaler
- 同一組 Daisy 超參數

只比較：

- fresh init
- warm-start

這比繼續混著調多個因素更能回答「舊權重到底是資產還是包袱」。

### 2. 保留多 group 資料，但把它視為資料設計前提

若目前資料集本來就刻意讓各 group 在相同時間點共享同一個 CAT 結構，則：

- 多 group 混訓應該被視為 intentional design
- 後續分析不要預設它是問題來源

### 3. 將 phase 2 視為探索線，而非新主線

在出現更明確的正向證據前，建議：

- `exp36` 仍可視為較穩的 baseline
- `exp37` 保留作為可運行的 continue-learning reference
- 不把 phase 2 直接升格為主線
