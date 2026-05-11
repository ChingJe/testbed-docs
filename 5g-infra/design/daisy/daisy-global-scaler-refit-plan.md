# Daisy Global Scaler Refit Plan

> 更新狀態：本文件最初記錄的是 master-side global refit temporary mitigation。依 2026-05-11 本地 Daisy 程式碼，預設 retrain path 已不再使用這條路徑作為正式 scaler lifecycle；目前本地主要流程已改為 round 1 local stats -> master aggregation -> round 2 reload shared scaler。本文件保留作為 temporary mitigation 的背景、限制與和後續對齊工作的比較基準。

## 目標

修正 Daisy retrain 流程中的 scaler 不一致問題，使每次 retrain task：

- 只產生一份全域共享 `scaler.pkl`
- 所有 Daisy clients 在同一座標系下訓練
- 最終 artifact 中的 scaler 與 retrain 使用的 scaler 完全一致

本設計只處理 scaler，不涵蓋 warm-start 或 fine-tune。

---

## 定位

本文件記錄的是 **暫時性快速緩解方案**，目的是先修正目前 retrain task 中各 client scaler 不一致、artifact scaler 與實際訓練 scaler 不一致的問題。

此方案採用：

- Daisy master 在 `publish_task` 時直接從 MongoDB 讀取同一個 `tid` 的全部 training docs
- master-side 一次 fit 出共享 `scaler.pkl`
- clients 全部讀取同一份 scaler 開始訓練

這樣做的優點是修改面小、容易快速落地；但它不是團隊長期要維持的最終方案。

### 長期方向

後續應考慮收斂到團隊 branch `origin/NWDAF-daisy-Dlinear` 的方向：

- 由各 client 先基於自己的本地資料計算 local scaler statistics
- 將 `mean / var / n` 回傳給 master
- 由 master 聚合出 global scaler
- 再讓所有 clients 使用同一份 global scaler 進行正式訓練

也就是說：

- **現在這份文件**：master-side global refit，屬於 temporary mitigation
- **長期設計方向**：federated statistics aggregation for scaler

### `origin/NWDAF-daisy-Dlinear` 的已知 fallback 問題

雖然 `origin/NWDAF-daisy-Dlinear` 的方向是正確的，也就是：

- round 1 由 clients 回傳 local scaler statistics
- master 聚合出 global scaler
- round 2 起正式訓練使用 shared scaler

但就目前 branch 內容來看，還殘留一條不夠乾淨的 fallback path：

- `dataset.py` 若發現 `model/{tid}/scaler.pkl` 不存在，會先以當前 client 的本地資料 fit 一份 scaler
- 該 fallback 在 branch 內仍可能寫回 shared scaler 路徑
- `client.py` 又是在模組載入時就先建 dataloader，不是等 stats round 完成後才建

這會帶來兩個問題：

1. 若多個 clients 幾乎同時啟動，誰先寫入 `model/{tid}/scaler.pkl` 會受到啟動時序影響。
2. branch 內的 `get_local_stats()` 是透過再次呼叫 `get_dataloaders(save_scaler=False)` 取得 scaler；若 fallback 已先產生 shared scaler，這一步可能讀到 fallback scaler，而不是重新從 raw local rows 計算真正的 local statistics。

因此，`origin/NWDAF-daisy-Dlinear` 比 master-side direct refit 更接近正確設計，但嚴格來說仍未完全封死 bootstrap fallback 對 stats round 的污染風險。

對齊時應採納的是它的 **federated aggregation 設計方向**，而不是原樣保留這條 fallback。

### 目前本地狀態

本地 Daisy 目前已進一步做了兩個收斂：

- `publish_task` 已移除「先用全部 `tid` docs direct refit scaler」的路徑
- `tid` retrain path 已不再允許 client 將 local-fitted scaler 寫回 shared `model/{tid}/scaler.pkl`

目前正式 retrain scaler lifecycle 可整理為：

1. client 啟動時，若 shared scaler 尚未存在，可先用 local rows 在記憶體中建立 bootstrap dataset
2. round 1 stats round 直接從 raw local rows 計算 `mean / var / n`
3. master 聚合後寫出唯一的 `model/{tid}/scaler.pkl`
4. round 2 reload dataloader，且要求 shared scaler 必須存在
5. round 2 起的正式訓練全部使用同一份 shared scaler

因此，以「正式 retrain 訓練 rounds 使用單一聚合後 shared scaler」這個目標來看，本地目前已經符合預期；剩下尚未處理的主題已轉為 warm-start / continue learning，而不是 global scaler lifecycle 本身。

後續若要正式收斂設計，需重新評估：

- scaler 建立流程應放在 training round 前的哪個 phase
- client / strategy / master 間如何傳遞 scaler statistics
- retrain 與 continue learning 是否共用同一套 scaler lifecycle
- 是否要將 input scaler 與 target scaler 的更新策略分離

### 目前方案的重要限制

目前這份 temporary mitigation 還有一個需要明確標示的限制：

若 master-side refit 直接將同一個 `tid` 下所有 groups 的 docs 混在一起，再交給只以 timestamp 聚合的 parser，則不同 group、但同一時間點的資料會被合成同一筆 row。

在這種情況下：

- `volume / packet count` 這類 sum 型 feature 會被直接加總
- `throughput` 類 feature 會變成跨 group 的平均值
- scaler 的統計量會對應到一條「混合後的大序列」，而不是各 group 各自的 row 聯集

因此這不只是語意偏差，而是會實際改變：

- normalized feature 的量級
- target 所在的標準化座標系
- model 訓練時的 loss landscape
- retrain 後模型的實際行為

若 group 是 client / 序列的基本單位，則「先混 group 再依 timestamp 聚合」會產生錯誤的 scaler。

### 接下來的對齊方向

基於上述限制，後續不建議繼續擴充 master-side global refit，而應逐步對齊 `origin/NWDAF-daisy-Dlinear` 的 federated scaler aggregation 設計。

規劃上的目標是：

- client 端先基於各自 `group_id` 資料計算 local scaler statistics
- master 端只負責聚合 `mean / var / n`
- 產出單一 `model/{tid}/scaler.pkl`
- 正式訓練 rounds 全部使用該 global scaler

也就是說，temporary mitigation 的角色應收斂為：

- 先止血目前 scaler 不一致問題
- 幫助釐清 multi-group row semantics
- 為後續切換到 federated statistics 方案提供比對基準

而不是再把更多長期行為疊加在這套 master-side refit 上。

### 當前邊界

由於本地 Daisy 目前已存在一些額外修改，對齊規劃需要明確控制變更面，避免誤動到無關路徑。

目前已觀察到本地有修改的重點檔案包含：

- `examples/07_MTLF_training/client.py`
- `examples/07_MTLF_training/custom_strategy.py`
- `examples/07_MTLF_training/daisyconfig.json`
- `examples/07_MTLF_training/dataset.py`
- `src/py/daisyfl/common/task_launcher.py`
- `src/py/daisyfl/master/server_api_handler.py`

其中可進一步分成三類：

### A. task 參數彈性擴充

這類修改應視為本地既有能力，後續 scaler 對齊時 **盡量不要動到**：

- `src/py/daisyfl/common/task_launcher.py`
  - 目前會在建立 strategy instance 時傳入 `task_config`
- `examples/07_MTLF_training/custom_strategy.py`
  - 目前可由 task config 控制：
    - `NUM_CLIENTS_FIT`
    - `NUM_CLIENTS_EVALUATE`
    - `ES_PATIENCE`
    - `LR_PATIENCE`
    - `INITIAL_LR`
- `examples/07_MTLF_training/client.py`
  - 目前可由 `MODEL_META.training.local_epochs` 控制 local epochs

這些修改主要是為了提高 task 參數彈性，與 scaler aggregation 本身不是同一個議題；後續若要對齊 DLinear branch，應以「保留這些能力」為前提。

### B. 本地環境 / 連線配置

- `examples/07_MTLF_training/daisyconfig.json`
  - 目前調整了本地 MongoDB 連線位置

這屬於環境配置，不應納入 scaler 對齊工作本身。

### C. scaler temporary mitigation 路徑

- `examples/07_MTLF_training/dataset.py`
- `src/py/daisyfl/master/server_api_handler.py`

這兩個檔案是目前 temporary mitigation 的主要承載點，也是之後最可能需要收斂或替換的部分。

因此後續對齊建議採以下邊界：

1. 優先在 `examples/07_MTLF_training/` 內完成 scaler 對齊。
2. 優先修改與 scaler directly 相關的檔案：
   - `dataset.py`
   - `client.py`
   - `custom_strategy.py`
   - 視需要最小幅度調整 `master.py`
3. 明確避免破壞 task 參數彈性擴充：
   - `task_launcher.py` 傳遞 `task_config` 的能力要保留
   - `custom_strategy.py` 對 task keys 的讀取能力要保留
   - `client.py` 對 `MODEL_META.training.local_epochs` 的支援要保留
4. 盡量不要再擴大修改 `src/py/daisyfl/master/server_api_handler.py` 與 `src/py/daisyfl/common/task_launcher.py`，除非只是為了維持現有相容性所必需。
5. 不在同一波變更中混入以下主題：
   - async callback / artifact packaging
   - continue learning / warm-start
   - NWDAF replay policy
   - ML Service 載入行為

換句話說，**接下來的對齊工作應聚焦在 scaler lifecycle 本身，並盡可能限制在 example 層處理**。

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

但考慮到 multi-group 語意錯位與團隊長期方向，**此方案不建議再作為正式終點**；後續應以 federated statistics aggregation 取代。

---

## 對齊規劃

### Phase A. 明確凍結 temporary mitigation 的責任邊界

先將目前方案定義為：

- 只負責解決「每個 client 各自 fit scaler」造成的不一致
- 不再承擔長期 global scaler 設計責任
- 不再作為 continue learning / dynamic scaler 的最終基礎

### Phase B. 對齊 row semantics

在切到 client local stats aggregation 前，需先明確統一「一筆 row 的定義」：

- group 是基本單位
- 同 timestamp 的不同 groups 應保留成不同 rows
- global scaler 應 fit 在各 group rows 的聯集上

若這一點不先固定，即使之後採用 federated aggregation，也會和實際訓練資料語意錯位。

### Phase C. 導入 client local stats → master aggregation

以 `origin/NWDAF-daisy-Dlinear` 為參考，逐步導入：

1. `dataset.py` 提供 local scaler statistics helper
2. `client.py` 支援 stats round
3. `custom_strategy.py` 聚合 `mean / var / n`
4. master 產出 `model/{tid}/scaler.pkl`
5. 後續 training rounds reload 並使用 global scaler

補充：

- 若只「照抄」`origin/NWDAF-daisy-Dlinear`，仍可能把 bootstrap fallback 一起帶回來
- 因此本地對齊的實際目標應是：
  - 保留 federated stats aggregation
  - 移除 `publish_task` direct refit
  - 移除 shared-scaler fallback write
  - 讓 round 2 正式訓練強制依賴聚合後的 shared scaler

### Phase D. 驗證與替換

完成 federated aggregation 後，需驗證：

- 同一個 `tid` 下，多個 clients 最終使用的 scaler 完全一致
- artifact 打包的 scaler 與訓練使用的 scaler 一致
- 與 temporary mitigation 相比，多 group 場景下 row semantics 已正確保留

若驗證通過，再逐步移除或停用 master-side direct refit 路徑。

### Phase E. 對齊 initial model scaler lifecycle

除了 retrain path，initial model 的 scaler 計算方式也需要和長期方案對齊。

目前 `train_initial_local.py` 的做法是：

- 先收集各 group 的完整 feature rows
- 先 fit scaler
- 之後才按時間順序切出 train / val

若長期要以 `origin/NWDAF-daisy-Dlinear` 現有語意作為基準，則 initial local baseline 也應同步對齊為：

- 先以各 group 的完整資料形成 feature rows
- 先完成 scaler 統計量計算
- 之後再做 train / val split

也就是說，**若以現有 DLinear branch 作為 source of truth，`initial_local` 的 split 時機需要後移**。

補充：

- 這不是唯一可能的設計；理論上也可以反過來調整 DLinear branch 讓它只用 train split 算 scaler
- 但若目前的團隊共識是「後續採用 DLinear branch 的寫法」，則規劃上應以對齊它為優先，而不是維持 `initial_local.py` 的現況

因此接下來的對齊工作可拆成兩條：

1. `initial_local.py` 對齊 DLinear branch 的 scaler fit 時機
2. Daisy retrain path 對齊 DLinear branch 的 client local stats → master aggregation

在這兩條都完成前，不應假設 initial model 與 retrain model 使用的是同一套 scaler lifecycle。

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

但這裡的「同一批資料」要更精確地理解為：

- scaler 應 fit 在「各 group 各自產出的 feature rows 的聯集」上
- 而不是先把不同 group 的 docs 混成單一 group-level 時序後再 fit

也就是說，training docs 的範圍可以跨多個 groups，但 row construction 不能失去 `group_id` 邊界。

### 3. feature extraction 與 client dataset 保持一致

master 計算 scaler 時，必須沿用與 `dataset.py` 相同的前處理邏輯：

- 相同的 notification parsing
- 相同的 10 維 feature 定義
- 相同的 `log1p`

除此之外，還必須保留與 client 相同的 group 邊界：

- client 端是對每個 `group_id` 各自解析、各自形成 row
- master 若要做等價的 global refit，也應先對各 group 分開產 row，再將 row 聯集拿來 fit scaler

若直接把所有 groups 的 docs 一次送進 parser，而 parser 的 key 只有 timestamp，則會導致 master 與 client 的 row construction 不一致。

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

補充：

- 第 3 步的 row 建立必須保留 `group_id` 邊界
- 不能直接把多個 groups 混成單一 timestamp 聚合序列
- 否則這裡 fit 出來的 scaler 會與 clients 實際資料分布不一致

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

補充：

- 這部分是 temporary mitigation 的核心修改點
- 後續若切換到 federated statistics aggregation，應以「縮小責任」為方向，而不是再往這裡疊更多 scaler 邏輯

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

但需注意：

- 不能只重用 parser 名稱
- 必須同時重用正確的 row construction 邏輯

若 parser 目前以 timestamp 為唯一聚合 key，則 master-side global refit 在多 group 場景下仍會產生錯誤 row。

因此後續正式修正時，應優先將 helper 調整為：

- 先按 `group_id` 分流
- 對每個 group 各自產出 feature rows
- 最後把所有 groups 的 rows 串接後再 fit scaler

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

但若未先修正多 group row construction，則上述效果只能視為止血，不應視為與長期 federated scaler aggregation 等價的正確結果。

---

## 非目標

以下項目不屬於本方案：

- warm-start / fine-tune
- scaler blend / partial update
- group-balanced scaler aggregation
- accuracy monitor / retrain policy 調整

這些應等 scaler 一致性修正完成後再評估。

---

## 目前結論摘要

若目標是與 `origin/NWDAF-daisy-Dlinear` 的 scaler 設計對齊，當前主要處理項目為：

1. `initial_local` 的 scaler fit 時機需與 DLinear branch 一致。
   - 也就是 split 順序應重新檢查，必要時改為先算 scaler、後做 split。
2. Daisy retrain path 應收斂到 DLinear branch 的作法。
   - client 計算 local stats
   - master 聚合 global scaler
   - 正式 training rounds 使用同一份 global scaler
3. row semantics 必須先固定。
   - group 是基本單位
   - 不能把多個 groups 混成同一條 timestamp 聚合序列

在這三點對齊前，目前本地 temporary mitigation 只能視為止血方案，不應直接視為最終正確設計。
