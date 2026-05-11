# Daisy Continue Learning / Scaler Notes

## 目的

整理 Daisy retrain 在 warm-start 與 scaler 策略上的設計觀察，先作為實驗規劃與後續實作的備忘。

> 更新狀態：本文件已依 2026-05-11 本地 Daisy / NWDAF 程式碼重新校正；部分早期假設已不再成立，以下內容以目前工作區實作為準。

本文件聚焦：

- retrain 是否從上一版模型繼續訓練
- Daisy 本地模型與 artifact 的保存位置
- fixed scaler / refit scaler / progressive scaler 的取捨
- 離線 replay 實驗中可接受但不完全合理的 oracle 設定

不涵蓋：

- Daisy async callback 打包流程
- global shared scaler refit 的 API 細節

相關文件：

- `daisy-global-scaler-refit-plan.md`
- `daisy-nwdaf-design.md`

### 與團隊 branch 的關係

目前本地 Daisy / NWDAF 的 scaler 處理，最早是以 `daisy-global-scaler-refit-plan.md` 所記錄的 master-side global refit 作為快速緩解方案，用來處理 retrain scaler 不一致問題。

但依目前本地程式碼，`07_MTLF_training` 已進一步收斂：

- `client.py` 已支援 stats round，client 會回傳 local scaler statistics
- `custom_strategy.py` 已支援 master 端聚合 `mean / var / n`
- 第 2 round 起，client 會 reload dataloader，改讀聚合後寫回的共享 `scaler.pkl`
- `publish_task` 已不再於 task 開頭先用全部 `tid` docs 直接 refit 一份 scaler
- `dataset.py` 在 `tid` retrain 路徑下，也不再將 client local-fitted scaler 寫回共享 `scaler.pkl`

也就是說，目前狀態比較接近：

- **已經以 federated scaler aggregation 作為主要 retrain 路徑**
- **已移除開頭 direct refit 與 shared-scaler fallback write**
- **warm-start / continue learning 尚未正式接上**

團隊後續長期方向，仍需考慮 `origin/NWDAF-daisy-Dlinear` 所採用的思路：

- 各 client 先計算自己的 local scaler statistics
- 回傳給 master
- 由 master 聚合成 global scaler
- 再讓 clients 使用該 global scaler 進行正式訓練

因此，本文件中對 fixed scaler / progressive scaler / continue learning 的討論，後續都應再檢查與 federated scaler aggregation 是否衝突，不能只以目前本地的 temporary mitigation 實作為前提。

---

## 目前已確認的行為

### 1. Daisy 每個 retrain task 都會在本地留下模型

以 `07_MTLF_training` 為工作目錄時，每個 task 會至少留下：

- `model/<tid>/model.npy`
- `model/<tid>/scaler.pkl`
- `artifacts/<tid>.tar.gz`

其中：

- `model/<tid>/model.npy` 是 task 訓練完成後落地的本地權重
- `artifacts/<tid>.tar.gz` 是 callback / download 用的打包產物

因此，若要做 continue learning，理論上可以直接從 Daisy 本地上一輪 task 的模型檔作為下一輪起點，不一定需要在 NWDAF 與 Daisy 間反覆傳輸 bundle。

但需要注意：

- Daisy `TaskManager` 會把 `MODEL_PATH` 同時當作「初始權重載入路徑」與「本輪訓練結果輸出路徑」
- 若直接把新 task 的 `MODEL_PATH` 指到舊 task 的 `model.npy`，新訓練結果會覆蓋舊模型

所以目前若要安全地做 warm-start，較合理的最小作法是：

- 先選定上一輪來源模型
- 複製到新 task 的 `model/<tid>/model.npy`
- 再讓 Daisy 依現有流程從這個新路徑載入並覆蓋

### 2. 目前 retrain 預設仍是 fresh init

雖然 Daisy 會保留上一輪的本地模型，但目前 `publish_task` 流程仍會為新 task 建立新的 `model/<tid>/model.npy`，並依 `MODEL_META` 初始化一份新權重。

所以在現況下：

- Daisy 有保存前一輪模型
- 但下一輪 retrain 並不會自動接續它
- async callback / artifact download 的完成，並不代表 retrain 已經具備 warm-start

### 3. Daisy dataset 目前同時 scale input 與 target

`07_MTLF_training/dataset.py` 的流程是：

1. 對 10 維 feature 做 `log1p`
2. 用 `StandardScaler` 做標準化
3. `x` 使用標準化後的全部 feature
4. `y` 也是從標準化後的 `ul_vol` / `dl_vol` 取出

這表示 scaler 不只定義 input 空間，也定義 target 空間。

直接更換 scaler 的副作用會比一般只 scale input 的系統更大。

### 4. 本地 retrain scaler lifecycle 已改以 federated aggregation 為主

依目前 `07_MTLF_training` 程式碼：

- round 1 會先進行一個 `is_stats_round`
- client 只回傳 local scaler statistics，不做正式訓練
- strategy 會聚合各 client 的 `mean / var / n`
- master 將聚合結果寫成 `model/<tid>/scaler.pkl`
- client 在 round 2 重新載入 dataloader，改用這份共享 scaler

另外需要注意一個目前仍存在、但語意上可接受的 bootstrap 細節：

- client process 啟動時，若共享 `scaler.pkl` 尚未存在，仍會先以本地資料在記憶體中建立 dataset / scaler
- 但這份 local scaler 不會再落盤為 shared scaler，也不會被當作正式 retrain scaler 發布
- round 1 stats round 仍是直接從各 client 的 local rows 計算統計
- round 2 之後，正式訓練一定要求共享 scaler 已存在，否則直接報錯

這表示：

- 目前已不是單純的 master-side temporary mitigation
- retrain 正式訓練已不再依賴 direct refit 或 fallback 存檔
- 若要評估 continue learning，主要剩下要決定的是是否固定 scaler，以及 warm-start 從哪個 model 起算

---

## Continue Learning 的基本判斷

### 1. 最乾淨的 continue learning

若只是要先驗證「接續上一版權重」是否有幫助，最乾淨的設定是：

- 沿用上一輪 `model.npy`
- 沿用同一份 scaler

這樣模型前後看到的是同一個座標系，比較接近真正的 fine-tune / warm-start。

### 2. 舊權重 + 新 scaler 也可行，但不是純 continuation

若沿用舊權重，但改用新 scaler：

- technically 可以訓練
- 但會同時改變 input 空間與 target 空間
- 實驗意義會變成 warm-start + rescaling adaptation

這種設定可能仍比隨機初始化好，但不能直接解讀成「同一座標系上的持續訓練」。

---

## Initial Bundle Scaler 的來源與含義

目前 `NWDAF/tools/retrain_replay/train_initial_local.py` 預設：

- `cat_duration=1800`，對應 CAT1
- initial model 只用 CAT1 資料訓練
- scaler 目前是收集各 group 的完整 feature rows 後一次 fit
- train / val split 是在 scaler fit 之後才做

因此，若後續所有 retrain 都固定使用這份 initial scaler，代表：

- model 起點是 CAT1
- scaler 參考座標系是 CAT1 全部可見 rows 的聯集

這對 continue learning 來說是合理的，因為可以固定表示空間；但若後面 CAT2 / CAT3 分布漂移很大，後段資料在 normalized space 可能會越來越偏。

---

## 固定 Scaler 的幾種策略

### 策略 A. 固定使用 CAT1 initial scaler

設定：

- initial model：CAT1 訓練
- scaler：CAT1 全部 rows fit
- 後續 retrain 全部沿用這份 scaler

優點：

- 座標系固定，最適合驗證 continue training
- 變因最少，結果最容易解讀

缺點：

- 若後續流量分布偏離 CAT1 很多，scaler 會逐漸失配
- 模型只能靠權重調整去吸收 distribution shift

適用：

- 第一輪快速驗證
- 需要判斷 warm-start 本身是否有效

### 策略 B. 固定使用全資料 scaler

設定：

- initial model：仍只用 CAT1 訓練
- scaler：用整段 replay 可見的全部資料 fit 一次
- 後續 retrain 全部沿用這份 scaler

優點：

- 對後段 CAT2 / CAT3 通常會比 CAT1 scaler 更穩
- 若要測試 continue learning 在較理想 normalization 下的上限，這是有價值的基線

缺點：

- 有 future leakage
- 屬於 offline oracle 設定，不適合直接當作線上可行策略
- 可能掩蓋 retrain 真正適應分布漂移的價值

適用：

- 離線 replay upper-bound baseline
- 驗證 normalization 先知資訊對結果的影響

### 策略 B 的目前落地方式（兩階段）

若目前要先用策略 B 做本地驗證，建議拆成兩個階段，避免一次把 scaler oracle 與 continue learning 混在一起：

#### 階段 1. 先改 `train_initial_local.py`，只測 fixed all-data scaler baseline

目標：

- 讓 initial bundle 的 `scaler.pkl` 改為用整段 replay 全資料 fit
- 但 `model.npy` 仍只用 CAT1 訓練

也就是：

- training rows：CAT1
- scaler rows：all replay data

這一階段的目的不是做線上可行策略，而是建立一個：

- `CAT1-trained model + all-data oracle scaler`

的 initial bundle，先觀察：

- replay trigger 是否更穩
- retrain / swap 結果是否改善
- 單純改 normalization 座標系是否已足以改變主線表現

這一步可能需要修改 `NWDAF/tools/retrain_replay/train_initial_local.py`，因為目前腳本中的資料範圍參數仍同時影響：

- 讀入哪些 rows
- scaler fit rows
- train / val sample rows

#### 階段 2. 再改 Daisy，接上 continue learning

當階段 1 的 oracle-scaler initial bundle 確認可用後，再進一步讓 Daisy retrain 接上 continue learning。

建議語意是：

- replay 起始模型使用階段 1 產出的 initial bundle
- Daisy 第一次 retrain 不再 fresh init
- 而是先複製 initial bundle 的 `model.npy` / `scaler.pkl` 到新 task 的 `model/<tid>/`
- 之後每次 retrain 再從上一輪 Daisy 本地模型接續

這樣可以把版本關係整理成：

- version 0：`train_initial_local.py` 產出的 initial bundle
- version 1：第一次 Daisy retrain，從 version 0 warm-start
- version N：第 N 次 Daisy retrain，從 version N-1 warm-start

這樣拆開的好處是：

- 階段 1 先回答「all-data oracle scaler 本身有沒有價值」
- 階段 2 再回答「在這個 scaler 座標系下，continue learning 有沒有額外價值」

若兩者一起改，結果會很難解讀。

### 策略 C. 每輪 refit 新 scaler

設定：

- 每次 retrain 都用當輪 training docs 重新 fit scaler
- 可以搭配 fresh init 或 old model warm-start

優點：

- 對當前資料分布最貼近

缺點：

- 若 target 也一起 scale，座標系每輪都會改
- 對 old model warm-start 不友善
- 實驗結果容易混入「scaler 變了」的影響

適用：

- 比較偏向完整系統適應性實驗
- 不適合作為第一個 continue-learning 驗證

### 策略 D. 每輪先做 federated stats aggregation，再固定該輪 scaler

設定：

- 每次 retrain 先由各 client 基於各自 group rows 計算 local scaler statistics
- 由 master 聚合成該輪共享 scaler
- 同一輪正式訓練全部使用這一份 global scaler
- 可搭配 fresh init 或 old model warm-start

優點：

- 比 master-side 把所有 docs 混在一起 refit 更接近長期方向
- 能保留「各 client 先各自形成 rows，再做 global aggregation」的語意
- 同一輪內所有 client 仍在同一座標系下訓練

缺點：

- 若每輪 scaler 都更新，對 continue learning 而言仍不是純 continuation
- client 啟動初期仍會短暫用 local scaler 建立 bootstrap dataset，但 round 2 後正式訓練會切回共享 scaler

適用：

- 對齊 `origin/NWDAF-daisy-Dlinear` 的中期路線
- 作為 continue learning 之後的第二階段實驗

---

## 動態 / 漸進式 Scaler 的常見做法

### 1. Rolling refit

只用最近一段資料重新 fit scaler，例如：

- 最近 N 個 slots
- 最近 30 分鐘 / 60 分鐘

特性：

- 能快速追新分布
- 但座標系跳動較明顯

### 2. Running statistics

以累積的 mean / variance 持續更新 scaler 統計量。

特性：

- 更新比 rolling refit 平滑
- 但若早期資料佔比過大，對新分布的反應會偏慢

### 3. EMA scaler

以指數移動平均更新，例如：

- `mean_t = (1 - a) * mean_(t-1) + a * mean_new`
- `scale_t = (1 - a) * scale_(t-1) + a * scale_new`

特性：

- 比直接替換 scaler 平滑
- 比純累積統計更容易追新分布
- `a` 可視為 adaptation speed

### 4. Input scaler 與 target scaler 分離

對 forecasting 類問題，較穩健的做法常是：

- input scaler 可以慢慢更新
- target scaler 固定，或更新得更慢

原因是 target scaler 一旦改變，模型最後輸出層對應的數值座標也會一起變。

在目前 Daisy `07_MTLF_training` 的實作中，input 與 target 使用的是同一套 scaler，因此若未來要做動態 scaler，建議優先考慮把兩者拆開。

---

## 對目前專案的實驗建議

### 第一階段：先建立策略 B baseline

目前較合理的第一步，不再是先驗證 warm-start，而是先建立：

- `CAT1-trained model + all-data oracle scaler`

的 baseline。

建議設定：

- initial bundle baseline：沿用 `initial_local_cat1_30s_v4` 的訓練設定作為出發點
- monitor / retrain policy：先和 `exp35` 完全一致
- 只修改 `train_initial_local.py` 的 scaler rows 來源
- `model.npy` 仍只用 CAT1 訓練
- `scaler.pkl` 改為用整段 replay 可見的全部資料 fit

也就是初版不建議同時改：

- monitor policy
- retrain window
- local epochs
- Daisy retrain lifecycle

這一階段要回答的是：

- all-data oracle scaler 本身是否會改變 replay trigger / retrain / swap 表現
- 不做 continue learning 的情況下，單純固定較理想的座標系能改善多少

若後續新增 `exp36` 一類的實驗，解讀上應該是：

- `exp35`：目前主線 initial scaler + current Daisy retrain
- `exp36`：all-data oracle initial scaler + current Daisy retrain

### 第二階段：再接 Daisy continue learning

當第一階段的 oracle-scaler initial bundle 確認可用後，再修改 Daisy 實作，讓 retrain 真正接上 continue learning。

在進入實作前，需要先明確區分：

- **目前本地 Daisy 的正式 retrain 路徑**
- **phase 2 想要驗證的 continue-learning 路徑**

兩者目前還不是同一件事。

#### 目前本地程式碼狀態（2026-05-11）

目前 retrain 的實際語意仍是：

- 每個新 task 以新的 `tid` 建立 `model/<tid>/model.npy`
- `publish_task` 仍會依 `MODEL_META` 對該路徑 fresh init 一份新權重
- `TaskManager` 再從這個新路徑載入初始參數，並在訓練後存回同一路徑
- scaler 方面則維持目前已清理過的主線：
  - round 1 local stats
  - master aggregate shared scaler
  - round 2 reload aggregated scaler

因此目前本地 Daisy 的 retrain，應解讀為：

- **fresh init**
- **per-task federated aggregated scaler**

而不是：

- warm-start
- fixed oracle scaler

這點要先寫清楚，避免後面把 phase 2 的目標和當前實作混在一起。

#### phase 2 想回答的問題

在目前這條文件主線下，phase 2 應回答的是：

- 在 **固定 `v5` all-data oracle scaler** 的座標系下
- Daisy retrain 若改成 **從上一版模型接續**
- 是否會比目前的 `fresh init + current retrain lifecycle` 再帶來額外收益

也就是：

- phase 1 / `exp36`：`all-data oracle initial scaler + current Daisy retrain`
- phase 2：`all-data oracle initial scaler + Daisy continue learning`

若 phase 2 實作時仍保留每個 task 的 federated scaler aggregation，實驗意義就會變成：

- warm-start + per-task rescaling adaptation

而不是目前文件想驗證的純：

- fixed-scaler continue learning

建議語意：

1. 第一次 retrain：
   - 不再 fresh init
   - 先將 initial bundle 的 `model.npy` / `scaler.pkl` 複製到新 task 的 `model/<tid>/`
   - 讓該 task 從 version 0 warm-start
2. 後續 retrain：
   - 再從上一輪 Daisy 本地模型接續

這一階段要回答的是：

- 在 fixed all-data oracle scaler 座標系下，continue learning 是否還能提供額外收益
- 第一次 retrain 是否應該視為從 initial bundle 接續，而不是重新開始

也就是下一組實驗應解讀為：

- `exp36`：all-data oracle initial scaler + current Daisy retrain
- 再下一組：all-data oracle initial scaler + Daisy continue learning

#### phase 2 的最小實作邊界

若要以最小改動完成上述語意，預計至少會碰到以下檔案：

- `NWDAF/tools/retrain_replay/retrain_replay.py`
- `daisy/src/py/daisyfl/master/server_api_handler.py`
- `daisy/examples/07_MTLF_training/custom_strategy.py`
- `daisy/examples/07_MTLF_training/client.py`

各自的角色大致如下：

1. `retrain_replay.py`
   - 在 publish task 時傳入 warm-start / fixed-scaler 所需 metadata
   - 第一次 retrain 指向 `v5` initial bundle
   - 後續 retrain 指向上一輪 Daisy 本地 task 的來源模型

2. `server_api_handler.py`
   - 在 `publish_task` 階段決定新 `tid` 的 seed model / seed scaler
   - 若有 warm-start source，先複製到 `model/<tid>/`
   - 有 warm-start source 時跳過 fresh init

3. `custom_strategy.py`
   - 需要有能力在 fixed-scaler mode 下停用 stats round
   - 避免每個 task 再次聚合並覆寫新的 `scaler.pkl`

4. `client.py`
   - fixed-scaler mode 下不再進入 local stats round
   - 也不再等待 round 2 reload aggregated scaler

`dataset.py` 目前不一定需要直接修改；只要 task 開始前 shared `scaler.pkl` 已由 master 正確 seed，client 初始 dataloader 就能直接載到固定 scaler。

另外要注意一個實驗解讀上的 caveat：

- 目前 Daisy 舊流程會把第 1 個 communication round 用在 stats round
- fixed scaler mode 下這個 stats round 會被停用
- 因此若 `NUM_ROUNDS` 保持不變，實際正式訓練 round 數可能會比舊流程多 1

這不一定要在 phase 2 初版特別修正，但在和 `exp35` / `exp36` 比較時，應明確把它當成潛在差異來源寫出來。

#### fixed scaler mode 的啟用語意

文件中的 fixed scaler mode，建議先用一個明確的 task-level 開關來表示，例如：

- `USE_FIXED_SCALER=true`

搭配兩個來源資訊：

- `SEED_MODEL_PATH`
- `SEED_SCALER_PATH`

在 phase 2 的目標設定下，它的簡單語意應該是：

1. replay / NWDAF 在 publish task 時傳入：
   - `USE_FIXED_SCALER=true`
   - 第一次 retrain 的 `SEED_MODEL_PATH = v5 initial bundle/model.npy`
   - 後續 retrain 的 `SEED_MODEL_PATH = 上一輪 Daisy 本地 model/<prev_tid>/model.npy`
   - `SEED_SCALER_PATH` 則固定指向 `v5 initial bundle/scaler.pkl`
2. Daisy master 在 `publish_task` 時：
   - 先把 `SEED_MODEL_PATH` 複製到新 `model/<tid>/model.npy`
   - 再把 `SEED_SCALER_PATH` 複製到新 `model/<tid>/scaler.pkl`
   - 並跳過 fresh init
3. Daisy strategy / client 在該 mode 下：
   - 不跑 stats round
   - 不生成新的 aggregated scaler
   - 直接使用這份預先 seed 好的 `model/<tid>/scaler.pkl` 完成訓練

也就是說，fixed scaler mode 的核心不是「client 自己不要 fit scaler」而已，而是：

- **在 task 開始前，就由 master 明確 seed 好該 task 的唯一 scaler**
- **並讓後續 round 全程都不再覆寫它**

若沒有停掉 stats round / aggregated scaler 產生流程，只是把一份固定 scaler 先放進 `model/<tid>/`，那仍不算這份文件裡定義的 fixed scaler mode。

### 第三階段：若前兩階段有效，再試更動態的 scaler

若第一階段證明 oracle scaler baseline 有價值，且第二階段證明 continue learning 有額外收益，再考慮更動態的 scaler 設計，例如：

- per-task federated aggregated scaler
- EMA scaler
- rolling refit
- input / target scaler 分離

這一階段才進一步回答：

- 固定 scaler 是否只是離線上限
- 若要往較接近線上的方向收斂，哪種動態 scaler 最穩妥

不建議一開始就直接做，否則會同時混入：

- oracle normalization
- continue learning
- scaler drift / adaptation

三種不同來源的效果，結果會很難解讀。

---

## 建議的短期結論

若目標是快速驗證：

- Daisy 下一輪 retrain 可以直接使用本地上一輪 task 的 `model.npy` 作為來源
- 但實作上應先複製到新 task 的 `model/<tid>/model.npy`，不要直接共用同一路徑
- scaler 先固定在 initial bundle 的 scaler，比較容易解讀 continue learning 是否有效

若目標是先做策略 B 的初版：

- 第一波先改 `train_initial_local.py`
- 產生一個 `CAT1-trained model + all-data oracle scaler` 的 initial bundle
- replay / Daisy 其他參數先盡量和 `exp35` 保持一致
- 先不要同一波就把 continue learning 接進來

若目標是做離線 upper-bound：

- 可以另外測一組全資料 scaler 固定用到底
- 但必須明確標示這是 offline oracle，不應與線上可行策略混為一談

若目標是再往下一步推進：

- 第二波再修改 Daisy，使第一次 retrain 從 initial bundle 開始 warm-start
- 之後各輪 retrain 再從上一輪 Daisy 本地模型接續
- 這樣才能把策略 B 與 continue learning 的效果分開觀察

若未來要做更完整的持續學習：

- 建議考慮 input / target scaler 分離
- 並以 EMA 類型的 progressive scaler 作為較穩妥的下一步
- 目前 `publish_task direct refit` 與 shared-scaler fallback write 已移除
- 下一步的重點已轉成：
  - 如何在 fixed scaler 與 per-task aggregated scaler 之間切換
  - 如何讓第一次 retrain 從 initial bundle 開始、之後各輪再從上一輪 Daisy 本地模型接續
