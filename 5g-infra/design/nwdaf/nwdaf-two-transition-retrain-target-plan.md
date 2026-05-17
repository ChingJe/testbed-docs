# NWDAF retrain two-transition target plan

## Purpose

這份文件定義下一階段 `NWDAF / retrain_replay / Daisy` 實驗的目標與規劃。

目前 `0514-22` 的 replay parity 工作已基本收斂到可接受狀態：

- `Monitor Surface` 與 `Policy Surface` 在 first-trigger 前已可做到 `shift +0`
- first trigger 已回到與 testbed 一致的主要方向
- 剩餘 prediction residual 已降為已知潛在問題，不再阻塞目前進度

因此下一階段不再以「貼齊既有 testbed 行為」為唯一目標，
而是轉向新的實驗目標：

- 只在 `CAT1 -> CAT2` 切換後觸發一次 retrain
- 只在 `CAT2 -> CAT3` 切換後再觸發一次 retrain
- 不在 phase 穩定區間額外觸發

---

## Target behavior

### Desired trigger pattern

目標是把完整實驗的 retrain 行為收斂成：

1. `CAT1-only` 期間不觸發 retrain
2. `CAT1 -> CAT2` 切換後，在合理延遲內觸發第一輪 retrain
3. 第一輪 hot-swap 完成後，在 `CAT2` 穩定區間不再次觸發
4. `CAT2 -> CAT3` 切換後，在合理延遲內觸發第二輪 retrain
5. 第二輪 hot-swap 完成後，不再出現第三輪 retrain

### Out of scope

目前不追求：

- 與 `0514-22` 原始 testbed log 的 post-swap 行為逐列完全一致
- Daisy 訓練 wall-clock 本體的精細模擬
- 用單一參數硬湊出所有場景都完美的 trigger 行為

---

## Current understanding

### What is already good enough

目前可以視為已收斂的部分：

- replay 的 prediction / monitor / policy / first-trigger 前半段已足夠準確
- startup timing / warmup / pending lifecycle / slot pairing 已不再是主要問題
- 因此下一階段可以把注意力移到「trigger policy 如何只在兩個 transition 後動作」

### What is still likely to matter

接下來最可能影響 trigger 數量與時機的因素：

1. `degradationPolicy.minDecisionTrafficScale`
2. retrain 後的 baseline accumulation 與 cooldown 行為
3. second swapped model 在 `CAT3` 穩定區段的表現
4. Daisy 路徑下 model swap 後的新 prediction / accuracy surface

---

## Important note on `minDecisionTrafficScale`

### Why this knob matters

`degradationPolicy.minDecisionTrafficScale` 並不改模型推論本身，
它影響的是某輪 `Accuracy policy` 是否允許進入 degradation decision。

在 NWDAF 主專案中，相關判斷位於：

- [trigger.go](/home/chingje/testbed/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/trigger.go)

核心語義是：

- `actualTrafficScale >= minDecisionTrafficScale`
  才能進入 degradation-eligible 路徑

因此這個值越低：

- 越多低流量 round 會被納入 decision
- 越容易提早觀測到 transition 後的 degradation
- 但也越容易在低流量 pocket 上累積誤觸發

### Why still start with a small value

雖然先前探索已顯示「不能只靠掃 `minDecisionTrafficScale` 解掉全部問題」，
但這一輪仍可把較小值當成故意放寬的探針：

- 先把 `minDecisionTrafficScale` 拉回較小值，例如 `1024`
- 觀察 trigger 是否明顯變多、變早、或穩定期誤觸發
- 若誤觸發明顯，再把它往上收回合理區間

這樣做的好處是：

- 能先確認目前真正受限的是 traffic gate，還是其他 post-swap 問題
- 能快速看出 phase transition 偵測能力是否被 `1 MiB` 門檻壓住

### Important caveat

`1024` 不應直接被視為預設最終值。

它在這份計畫中的角色是：

- 第一輪探索用的 intentionally permissive probe

而不是：

- 預設長期設定

---

## Prior evidence

根據既有整理：

- `327680` 曾成功把第一輪 retrain 拉回較有意義的 `CAT2`
- `1048576` 在 skip-Daisy 下表面漂亮，但 full Daisy 下仍可能出現第三輪
- `1572864` 會把 trigger 壓得過頭，連第二輪也可能消失

因此目前判讀是：

- `minDecisionTrafficScale` 仍然是重要 knob
- 但它無法單獨保證 full Daisy 路徑只剩兩輪
- 它更適合作為第一層 traffic gate，後面還要配合 post-swap 行為一起看

---

## Proposed experiment order

### Phase 1: permissive gate probe

先用較小門檻確認系統在「幾乎不擋 traffic gate」時會怎麼動。

建議起點：

- `degradationPolicy.minDecisionTrafficScale = 1024`

觀察重點：

1. `CAT1-only` 是否就開始累積 hits
2. 第一輪 trigger 是否明顯早於 `CAT1 -> CAT2`
3. `CAT2` 穩定段是否出現額外 trigger
4. `CAT3` 穩定段是否出現第三輪 trigger

### Phase 2: raise to practical range

若 `1024` 明顯過度 permissive，則往上收回 practical range。

建議回看順序：

1. `327680`
2. `1048576`
3. 必要時再補中間值

目的不是盲掃，而是回答兩個問題：

1. 哪個值能保留兩個 transition trigger
2. 哪個值能避免穩定區間誤觸發

### Phase 3: post-swap behavior review

若即使 traffic gate 已合理，仍出現第三輪，
則問題很可能不再是 early gate，而是：

- second swapped model 在 `CAT3` 某些 pocket 上仍不穩
- retrain 後 baseline / cooldown / reset 行為不足

這一階段才需要進一步看：

- Daisy 訓練產物
- swap 後 accuracy surface
- post-swap monitor / policy 狀態累積

---

## Validation method

每輪驗證固定使用：

- [exp64.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp64.yaml)
  的對應變體
- [compare_nwdaf_replay_testbed.py](/home/chingje/testbed/5G_Infrastructure/.agent/compare_nwdaf_replay_testbed.py)
  做基準對照

但這一階段的主要觀察指標，不再只是 testbed parity，而是：

1. retrain trigger 次數
2. retrain trigger 所屬 group
3. trigger 時刻相對於 `CAT1 -> CAT2` / `CAT2 -> CAT3` 的位置
4. hot-swap 後是否在穩定 phase 再次觸發

建議每次都整理成同一張結果表：

| Run | `minDecisionTrafficScale` | Daisy mode | Trigger count | First trigger phase | Second trigger phase | Third trigger? | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |

---

## Experiment results

### Run summary

本輪實際新增並執行的變體如下：

| Run | Main change | Result |
| --- | --- | --- |
| `exp65` | `minDecisionTrafficScale=1024`, `requiredHits=2`, full Daisy | `3` 次 retrain |
| `exp67` | `exp65` 基礎上只改 `requiredHits=3`, full Daisy | `3` 次 retrain |
| `exp66` | `minDecisionTrafficScale=327680`, full Daisy | 未完成；該次執行中斷，未納入結論 |

### `exp65` result

[exp65.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp65.yaml)
使用：

- `degradationPolicy.minDecisionTrafficScale = 1024`
- `requiredHitsInWindow = 2`
- `zScoreThreshold = 1.3`

結果：

1. 第一輪 trigger：`00:34:47 group1`
2. 第二輪 trigger：`01:03:17 group1`
3. 第三輪 trigger：`01:25:47 group2`

這證明：

- `1024` 這條線確實足以保留前兩輪 transition 附近的 retrain
- 但它沒有把第三輪壓掉
- 因此若單看 trigger 數量，`1024` 對 full Daisy 來說過於 permissive

### `exp67` result

[exp67.yaml](/home/chingje/testbed/docs/5g-infra/experiments/exp67.yaml)
使用：

- `degradationPolicy.minDecisionTrafficScale = 1024`
- `requiredHitsInWindow = 3`
- `zScoreThreshold = 1.3`

結果：

1. 第一輪 trigger：`00:36:17 group1`
2. 第二輪 trigger：`01:04:47 group1`
3. 第三輪 trigger：`01:24:17 group1`

與 `exp65` 相比：

- 第一輪往後延約 `90s`
- 第二輪也往後延約 `90s`
- 第三輪沒有消失，且來源從 `group2` 轉成 `group1`

這證明：

- `requiredHitsInWindow=3` 不會讓第一輪 retrain 消失
- 它確實會延後 retrain
- 但它不能單獨解決第三輪 retrain 問題

### What the `requiredHits=3` probe proved

這次最重要的新增資訊是：

- 不能直接從 `exp65` 的 pre-trigger rows 推論 `requiredHits=3` 會不會保留第一輪 retrain
- 實測後已確認會保留
- 但它沒有換來「只剩兩輪」的結果

因此目前可以排除一種過度樂觀的假設：

- 「只要把第二輪稍微延後，讓 CAT3 訓練樣本多一些，就能自然消掉第三輪」

至少在這條 current line 上，這件事沒有發生。

---

## Z-score threshold assessment

### Why `zScoreThreshold` was considered

在 `requiredHits=3` 仍保留第一輪 retrain 後，
下一個自然會想到的 knob 是 `zScoreThreshold`。

文件中的歷史主線也曾偏向：

- 對 aligned line，優先微調 `zScoreThreshold`
- 而不是先改 `requiredHitsInWindow`

### What current runs suggest

從 `exp65` / `exp67` 的實際 `policy.parquet` 看：

- 第一輪 trigger 前的有效 `zscore` 約在 `1.84`, `3.84`, `2.24`
- 第二輪 trigger 前的有效 `zscore` 約在 `1.50`, `1.50`, `1.76`
- 第三輪 trigger 前則可能出現更高的值，例如 `5.95`, `5.30`, `2.33`

這代表：

1. 若 `zScoreThreshold` 從 `1.3` 提到 `1.4`
   - 很可能幾乎不改變當前三輪結構
2. 若提到 `1.5`
   - 更可能先傷到第二輪
   - 不一定能精準消掉第三輪
3. 若要靠更高門檻硬壓第三輪
   - 又會同時侵蝕第一輪與第二輪的設計意義

因此目前判斷是：

- `zScoreThreshold` 仍是可以測的 knob
- 但它不像是目前最有希望「保留前兩輪、只壓第三輪」的單一解法

---

## CAT3 pre-data observation

### Why inspect the dataset directly

在 `exp65` / `exp67` 都無法自然消掉第三輪之後，
一個更根本的問題是：

- 第二輪 retrain 看到的 `CAT3` 前段資料，
  是否本來就不足以覆蓋 `CAT3` 後段的 pattern

為此，本輪直接回看了：

- [group1/training_packets_run001.parquet](/home/chingje/testbed/5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet)
- [group2/training_packets_run001.parquet](/home/chingje/testbed/5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet)

並以 replay 會實際看到的 `30s slot` 粒度，
把 `CAT3` 切成前 / 中 / 後三段觀察。

### `group1` finding

`group1` 的 `CAT3` 並不是單純「前段太弱」。

觀察到的現象更接近：

- 前段已有足夠高流量樣本
- 後段則同時存在更多低谷 slot 與高 burst slot
- 後段 pattern 比前段更不穩定、更不均質

這表示：

- 第二輪 retrain 即使吃到一些 `CAT3` 前段資料
- 也不保證能泛化到 `CAT3` 後段的 low-pocket / bursty pattern

### `group2` finding

`group2` 的 `CAT3` 甚至不是「前段不夠」，
而是整段呈現較明顯的 level shift：

- 前段最強
- 中段下降
- 後段再下降

因此若模型主要從較早的 `CAT3` window 學習，
反而本來就可能難以覆蓋較晚段的較弱流量區間。

### Interpretation

目前更合理的判讀是：

- 第三輪 retrain 不完全是 policy parameter 沒調對
- 它也可能反映了 `CAT3` 前後段本身不夠同質
- 尤其是 `group1` 後段的低谷與 burst pocket，
  未必是第二輪 retrain window 自然能覆蓋的東西

---

## Current conclusion

### What this round achieved

這一輪已經把幾個重要問題釐清：

1. `minDecisionTrafficScale=1024` 的 permissive probe 已證實：
   - 它能保留前兩輪 trigger
   - 但會留下第三輪
2. `requiredHitsInWindow=3` 的 probe 已證實：
   - 第一輪 retrain 不會消失
   - retrain 會延後
   - 第三輪仍然存在
3. `CAT3` pre-data 本身顯示：
   - 第三輪問題不一定是純 policy knob 可解
   - 資料本身的 phase heterogeneity 可能就是主要來源之一

### Why not keep pushing tricky tuning

基於目前結果，繼續做很細的參數微調有幾個風險：

- 可能只是把第三輪 trigger 在不同 group / 不同時間搬來搬去
- 參數會逐漸偏離原本設計意義
- 最終得到的可能只是對單一 dataset pocket 的 overfit

因此目前比較合理的收斂方式是：

- 把當前組合視為「足夠有道理、可工作的實驗基線」
- 接受第三輪是目前這個 dataset / phase 結構下可理解的已知現象
- 不把「一定要只剩兩輪」視為必須靠 policy 調參解掉的硬性 bug

### Practical stance

目前建議：

1. 保留當前 parity-converged replay line 作為主基線
2. 把這輪 two-transition tuning 的結論記錄下來
3. 暫停進一步的 tricky policy tuning
4. 若未來還要再追 third retrain，
   優先從 dataset coverage / post-swap training semantics / model generalization 著手，
   而不是先繼續堆 policy knob

---

## Historical implementation slice

以下保留的是本輪開始前的原始規劃，作為實驗順序的背景。

## First implementation slice

這一輪實作先不要直接改 NWDAF 主程式邏輯，
優先做的是：

1. 準備新的 experiment config 變體
2. 先跑 `minDecisionTrafficScale = 1024` 的 permissive probe
3. 若誤觸發過多，再往 `327680` / `1048576` 回收
4. 先用 replay + Daisy 結果判斷哪一段最值得進一步改 policy / lifecycle

這一輪的主要目的不是立刻定版，
而是先把：

- transition-sensitive behavior
- false-positive profile
- post-swap retrigger profile

三者分開看清楚。

---

## Decision checkpoint

在進入程式或 config 實作前，先確認本輪接受的方向是：

1. 先把 `minDecisionTrafficScale` 降到 `1024` 當作 permissive probe
2. 以 replay / Daisy 實驗觀察 trigger pattern，而不是先改 NWDAF code
3. 若 `1024` 誤觸發過多，再逐步拉回 `327680`、`1048576` 一帶
4. 只有在 traffic gate 已證明不是主因後，才再看 post-swap policy / lifecycle 調整
