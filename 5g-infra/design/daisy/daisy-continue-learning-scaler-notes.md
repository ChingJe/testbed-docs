# Daisy Continue Learning / Scaler Notes

## 目的

整理 Daisy retrain 在 warm-start 與 scaler 策略上的設計觀察，先作為實驗規劃與後續實作的備忘。

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

目前本地 Daisy / NWDAF 的 scaler 處理，包含 `daisy-global-scaler-refit-plan.md` 所記錄的 master-side global refit，主要是為了快速緩解現有 retrain scaler 不一致問題。

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

### 2. 目前 retrain 預設仍是 fresh init

雖然 Daisy 會保留上一輪的本地模型，但目前 `publish_task` 流程會為新 task 建立新的 `model/<tid>/model.npy`，並依 `MODEL_META` 初始化一份新權重。

所以在現況下：

- Daisy 有保存前一輪模型
- 但下一輪 retrain 並不會自動接續它

### 3. Daisy dataset 目前同時 scale input 與 target

`07_MTLF_training/dataset.py` 的流程是：

1. 對 10 維 feature 做 `log1p`
2. 用 `StandardScaler` 做標準化
3. `x` 使用標準化後的全部 feature
4. `y` 也是從標準化後的 `ul_vol` / `dl_vol` 取出

這表示 scaler 不只定義 input 空間，也定義 target 空間。

直接更換 scaler 的副作用會比一般只 scale input 的系統更大。

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
- scaler 不是用整個 CAT1 全部 rows fit，而是用各 group 的 train split fit

因此，若後續所有 retrain 都固定使用這份 initial scaler，代表：

- model 起點是 CAT1
- scaler 參考座標系也是 CAT1 train split

這對 continue learning 來說是合理的，因為可以固定表示空間；但若後面 CAT2 / CAT3 分布漂移很大，後段資料在 normalized space 可能會越來越偏。

---

## 固定 Scaler 的幾種策略

### 策略 A. 固定使用 CAT1 initial scaler

設定：

- initial model：CAT1 訓練
- scaler：CAT1 train split fit
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

### 第一階段：先確認 warm-start 是否有效

優先比較：

1. fresh init + current retrain scaler
2. old model + fixed CAT1 scaler

如果第 2 組顯著較好，代表 continue training 本身值得做。

### 第二階段：再看 scaler 策略的影響

建議再加：

3. old model + fixed all-data scaler
4. old model + per-task refit scaler

這樣可以拆開看：

- warm-start 是否有效
- normalization 先知資訊是否影響結果
- refit scaler 會不會帶來額外收益或不穩定

### 第三階段：若 warm-start 確認有效，再試 progressive scaler

若固定 scaler 證明有效，再評估：

- EMA scaler
- rolling refit
- input / target scaler 分離

不建議一開始就直接做，否則變因太多，不容易解讀結果。

---

## 建議的短期結論

若目標是快速驗證：

- Daisy 下一輪 retrain 可以直接使用本地上一輪 task 的 `model.npy`
- scaler 先固定在 initial bundle 的 scaler，比較容易解讀 continue learning 是否有效

若目標是做離線 upper-bound：

- 可以另外測一組全資料 scaler 固定用到底
- 但必須明確標示這是 offline oracle，不應與線上可行策略混為一談

若未來要做更完整的持續學習：

- 建議考慮 input / target scaler 分離
- 並以 EMA 類型的 progressive scaler 作為較穩妥的下一步
