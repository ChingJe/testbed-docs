# NWDAF Accuracy Policy Migration Plan

本文件規劃 `5g-viz` 對新版 NWDAF metric 監控模式的對齊改造。重點不是單純更新幾條 regex，而是把
目前仍殘留舊版 `accuracy` / `threshold_breach` 假設的事件模型，升級為以
`Accuracy scope`、`Accuracy policy`、`Retrain trigger` 為主的跨層觀察模型。

## 背景

`5g-viz` 目前仍殘留舊版 NWDAF 假設：

- parser / metrics / Grafana 仍依賴舊 `accuracy` 路徑
- Grafana 還以 `nwdaf_deviation` 為主要 NWDAF 監控圖
- replay / pseudo-live 也還在重播這套舊事件語意

但新版 NWDAF 的主資料已收斂為兩層：

1. `Accuracy scope [model]`
   per-scope 原始 metric 輸出
2. `Accuracy policy [model]`
   per-scope retrain 決策輸出

而且舊的 `Accuracy [model]` log 之後會被移除，因此 `5g-viz` 不應再依賴它。

## 目標

1. 讓 parser 能完整識別新版 NWDAF accuracy monitoring log。
2. 移除舊 `accuracy` 與 `threshold_breach` 依賴。
3. 讓 Grafana 主顯示收斂成三種類型：`traffic`、`degradation`、`chronic`。
4. 讓 live、replay、pseudo-live 三條路徑共用同一套事件與 metric 建模。
5. 讓 `docs/5g-viz/design/` 與實作一致。

## 目前進度

截至目前，`5g-viz` 已經完成第一輪主幹實作：

- parser 已改成新版 `accuracy_scope` / `accuracy_policy` / `retrain_trigger`
- 舊 `accuracy` / `threshold_breach` / `nwdaf_deviation` 主路徑已移除
- live / replay / pseudo-live 都已改成同一套 policy metrics
- Grafana 已改成 `traffic / degradation / chronic` 三個主區塊
- `baselineReady`、`hits`、`retrain start/done` 已改成 annotation，而不是普通曲線
- `z-score`、`chronicValue`、`trafficScale` 已有虛線 threshold
- topology 的 `accuracy_policy` 已改成只有 state change 才會跳出，不再每輪都出現

目前剩餘工作偏向收尾與文件對齊，而不是主資料模型重建。

## 非目標

- 不在這次改動中重寫整個 topology framework。
- 不在這次改動中把 offline `retrain_analysis` 報告完整搬進 `5g-viz`。
- 不保留舊版 `accuracy` 或 `threshold_breach` 相容性。

## 目前已確認事實

- 新版 NWDAF 已穩定輸出 `Accuracy scope [...]`、`Accuracy policy [...]`、`Retrain trigger [...]`。
- 舊的 `Accuracy [model]` log 之後會被移除，因此不能再作為 parser、metrics 或圖表依賴。
- NWDAF 可由 `accuracyMonitor.metricsToRecord` 決定要輸出哪些 accuracy metrics；`5g-viz`
  不應把 `MAE/MSE/NRMSE/WAPE/sMAPE` 寫死成唯一集合。
- 目前 retrain 主要來自兩條決策路徑：
  - z-score path：對 `primaryMetric` 做 baseline / z-score 判斷
  - chronic path：對 `chronicPolicy.metric` 的 `chronicValue` 做長期劣化判斷
- 目前預設 config 中：
  - `primaryMetric = MAE`
  - `chronicPolicy.metric = WAPE`
  - 因此主要 retrain 觀察應聚焦 `MAE z-score` 與 `WAPE chronicValue`

## 核心設計決定

### 1. 事件模型升級

新版主事件只保留：

- `accuracy_scope`
- `accuracy_policy`
- `retrain_trigger`
- `retrain_done`
- `model_swap`

直接移除：

- `accuracy`
- `threshold_breach`

移除內容包含：

- parser 規則
- event schema 定義
- live metrics 映射
- replay / pseudo-live 映射
- Grafana 舊 panel 依賴

### 1.5. Config Source Ownership

`5g-viz` 的圖表與 parser 顯示邏輯，不應直接依賴 testbed 內某份 NWDAF config 檔。

實作原則：

- `5g-viz` 應有自己的 config / profile 設定來源
- 使用者必須在 `5g-viz` 端自行設定：
  - `primaryMetric`
  - `chronicMetric`
  - 需要優先顯示的 metric 集合
- `5g-viz` 可以選擇性對照 NWDAF log 中實際出現的 metric 做容錯或告警
- 但不能假設 testbed 中的 `nwdafcfg.yaml` 就是前端顯示的 canonical config source

換句話說：

- NWDAF log 決定「實際發生了什麼」
- `5g-viz` config 決定「頁面主要要怎麼解讀與顯示」

### 2. 主要顯示模型

主顯示收斂成三種類型，而不是依 log 類型各拆一張：

1. `traffic`
2. `degradation`
3. `chronic`

這三種顯示類型對應三個問題：

1. traffic 本身是否已開始和 prediction 偏離？
2. z-score path 為什麼會觸發 retrain？
3. chronic path 為什麼會觸發 retrain？

### 3. Traffic

用途：

- 看 actual / predicted traffic 是否開始偏離

資料來源：

- `aggregated_slot`
- `ml_inference`

對應圖表：

- actual vs predicted

### 4. Degradation

用途：

- 看 z-score 路徑是否正朝 retrain 條件逼近

核心顯示內容：

- `Primary metric`
  - 目前預設是 `MAE`
- `mean`
- `std`
- `z-score`

資料來源：

- `Accuracy scope [model]`
- `Accuracy policy [model]`

實作要求：

- `Primary metric` 來自 `primaryMetric`
- 若未來 config 更改 `primaryMetric`，panel 內容應自動切換，不能把 `MAE` 寫死

需疊加的狀態 / annotation：

- `baselineReady`
- `degradationHits`
- `retrain start`
- `retrain done`

### 5. Chronic

用途：

- 看 chronic 路徑是否正朝 retrain 條件逼近

核心顯示內容：

- `TrafficScale`
- `ChronicValue`
  - 目前預設是 `WAPE`

資料來源：

- `Accuracy policy [model]`

實作要求：

- `ChronicValue` 對應的 metric 來自 `chronicPolicy.metric`
- 若未來 config 更改 chronic metric，panel 內容應自動切換，不能把 `WAPE` 寫死

需疊加的狀態 / annotation：

- `baselineReady`
- `chronicHits`
- `retrain start`
- `retrain done`

### 6. Overlay 原則

`baselineReady`、`hits`、`retrain start/done` 不另外拆成獨立圖，也不應作為普通 timeseries 線，
而是以 annotation 疊加在對應 decision panel 上：

- `degradation` panel 疊加：
  - `baselineReady`
  - `degradationHits`
  - `retrain start / done`
- `chronic` panel 疊加：
  - `baselineReady`
  - `chronicHits`
  - `retrain start / done`

另外：

- annotation 需要有獨立開關，使用體驗應比照 `retrain start/done`
- dashboard 需要提供一條極窄的顏色說明列，說明各 annotation 顏色代表的事件

這樣可以維持三個主區塊，而不把單一路徑的訊息打散到多張圖。

### 6.5. Future Page Layout

未來頁面可先收斂成下列布局：

```text
+----------------------------------------------------------------------------------+
| Header: Session | Time Range | Play/Replay | Metric Config | Legend/Status       |
+----------------------------------------------------------------------------------+
| Topology                                                                         |
|                                                                                  |
|  topology canvas only                                                            |
|  no internal detail sketch here                                                  |
|                                                                                  |
+----------------------------------------------------------------------------------+
| Traffic                                                                          |
| actual vs predicted                                                              |
+----------------------------------------------------------------------------------+
| Degradation                                                                      |
|                                                                                  |
|  (Primary metric: MAE)                                                           |
|  (mean / std)                                                                    |
|  (z-score)                                                                       |
|                                                                                  |
|  overlay: baselineReady                                                          |
|  overlay: degradationHits                                                        |
|  overlay: retrain start / done                                                   |
+----------------------------------------------------------------------------------+
| Chronic                                                                          |
|                                                                                  |
|  (TrafficScale)                                                                  |
|  (ChronicValue: WAPE)                                                            |
|                                                                                  |
|  overlay: baselineReady                                                          |
|  overlay: chronicHits                                                            |
|  overlay: retrain start / done                                                   |
+----------------------------------------------------------------------------------+
| Log                                                                              |
| event / detail log tail                                                          |
+----------------------------------------------------------------------------------+
```

布局原則：

- 上半部先看 topology，不在布局草圖中預畫 NF 細節
- 不額外畫 `events/details` 側欄
- 不在 topology 草圖中預畫 `Daisy` 或 `SMF`
- 中段固定三類主顯示：
  - `traffic`
  - `degradation`
  - `chronic`
- `degradation` 與 `chronic` 改為上下堆疊，各自佔一個完整區塊
- `degradation` 區塊內再拆成三個子圖：
  - `Primary metric`
  - `mean / std`
  - `z-score`
- `chronic` 區塊內再拆成兩個子圖：
  - `TrafficScale`
  - `ChronicValue`
- `baselineReady`、`hits`、`retrain start/done` 以 overlay / annotation 疊在各自區塊
- log 固定放在最下方

### 7. 拓樸語意重新對齊

拓樸應拆開「AnLF 回報」與「MTLF 判斷」兩個階段：

- `accuracy_scope`
  - 新增 `AnLF -> MTLF` 的 `AccuracyScope` 邊
  - 表示 AnLF 已輸出本輪 per-scope metrics
- `accuracy_policy`
  - 新增 MTLF self-edge，顯示 policy evaluation
  - label 不直接顯示 `AccuracyPolicy`
  - 展示語意改為較直覺的決策型標籤，例如：
    - `Degradation Check group-test-001 2/3`
    - `Chronic Check group-test-002 1/3`
    - `Retrain Decision`
  - label 至少要直接帶出目前 hits 次數
    - degradation path: `degradationHits/required`
    - chronic path: `chronicHits/required`
  - 其餘數值如 `metric`、`zscore`、`chronicValue` 用 hover / detail 呈現，不全部塞進 edge label
  - 只有 policy state change 時才應觸發 topology
    - 不應每輪都跳出同一條 check edge
- `retrain_trigger`
  - 保持 retraining class 與 retrain self-edge
  - label 可顯示 `reason`

不再保留舊 `ThresholdBreach {n}/{total}` edge。

### 7.5. Topology Delta Summary

新版 topology 事件增減可總結如下。

新增或提升為主事件：

- `accuracy_scope`
  - edge:
    - `nwdaf_anlf -> nwdaf_mtlf`
  - label:
    - `AccuracyScope`
  - pulse:
    - `nwdaf_anlf`
- `accuracy_policy`
  - edge:
    - `nwdaf_mtlf -> nwdaf_mtlf`
  - label:
    - `Degradation Check {scopeShort} {degradationHits}/{degradationRequired}`
    - `Chronic Check {scopeShort} {chronicHits}/{chronicRequired}`
  - pulse:
    - `nwdaf_mtlf`
  - topology gate:
    - only emit when `policy_path` / `baselineReady` / hit counters change
- `retrain_trigger`
  - edge:
    - `nwdaf_mtlf -> nwdaf_mtlf`
  - label:
    - `RetrainTrigger {reason}`
  - pulse:
    - `nwdaf_mtlf`
  - class:
    - add `retraining` to `nwdaf_mtlf`
- `retrain_done`
  - edge:
    - `nwdaf_mtlf -> nwdaf_anlf`
  - label:
    - `ModelProvision`
  - pulse:
    - `nwdaf_mtlf`
  - class:
    - remove `retraining` from `nwdaf_mtlf`
- `model_swap`
  - edge:
    - `nwdaf_anlf -> nwdaf_anlf`
  - label:
    - `ModelDeploy`
  - pulse:
    - `nwdaf_anlf`

保留的既有主事件：

- `upf_volume`
  - edge:
    - `{upf} -> nwdaf`
  - pulse:
    - `{upf}`
- `ml_inference`
  - pulse:
    - `nwdaf_anlf`
- ADRF 相關事件
  - `nwdaf -> adrf`
  - `nwdaf_mtlf -> adrf`
  - `adrf -> nwdaf_mtlf`

移除的事件：

- `accuracy`
  - 移除原因：
    - 舊 `Accuracy [model]` log 之後會消失
- `threshold_breach`
  - 移除原因：
    - 已不再代表新版主要 retrain 決策訊號

因此 topology 的重心會從：

- `accuracy report -> threshold breach -> retrain`

改成：

- `accuracy_scope -> accuracy_policy -> retrain_trigger -> retrain_done -> model_swap`

## 預期修改範圍

### A. Parser / Event Schema

目標檔案：

- `5g-viz/rules/nwdaf.py`
- `docs/5g-viz/design/overview/event-schema.md`
- `docs/5g-viz/design/features/nwdaf-ml-cycle.md`

具體修改：

1. 新增 `Accuracy scope [...]` parser
   - event type: `accuracy_scope`
   - 欄位：
     - `model`
     - `scope`
     - `samples`
     - `metrics` 原始 map
     - 常用平鋪欄位：`mae`、`mse`、`nrmse`、`wape`、`smape`
2. 新增 `Accuracy policy [...]` parser
   - event type: `accuracy_policy`
   - 欄位：
     - `model`
     - `scope`
     - `metric`
     - `current`
     - `mean`
     - `std`
     - `zscore`
     - `degradationEligible`
     - `degradationSignal`
     - `baselineReady`
     - `trafficScale`
     - `chronicEligible`
     - `chronicSignal`
     - `chronicValue`
     - `degradationHits`
     - `degradationRequired`
     - `chronicHits`
     - `chronicRequired`
     - `hitReason`
3. 更新 `retrain_trigger`
   - 主要上游訊號來自 `Accuracy policy`
   - `reason`、`scope`、`metric` 都要納入 event payload
4. 刪除 `accuracy`
5. 刪除 `threshold_breach`

### B. Live Metrics

目標檔案：

- `5g-viz/rules/nwdaf.py`
- `docs/5g-viz/design/backend/metrics.md`

具體修改：

1. 保留現有 retrain lifecycle metrics
   - `nwdaf_retrain_total`
   - `nwdaf_retrain_start_event`
   - `nwdaf_retrain_done_event`
2. 新增 per-scope metrics
   - labels 建議：`session`、`model`、`scope`、`metric`
   - 指標集合不應寫死，應以 `Accuracy scope` log 中實際出現的 metric 為準
   - 第一版實作可統一成：
     - `nwdaf_accuracy_scope_value`
3. 新增 policy metrics
   - labels 建議：`session`、`model`、`scope`、`metric`
   - 指標：
     - `nwdaf_accuracy_policy_current`
     - `nwdaf_accuracy_policy_mean`
     - `nwdaf_accuracy_policy_std`
     - `nwdaf_accuracy_policy_zscore`
     - `nwdaf_accuracy_policy_traffic_scale`
     - `nwdaf_accuracy_policy_chronic_value`
     - `nwdaf_accuracy_policy_degradation_hits`
     - `nwdaf_accuracy_policy_chronic_hits`
4. 新增 policy state gauges
   - `nwdaf_accuracy_policy_baseline_ready`
   - `nwdaf_accuracy_policy_degradation_eligible`
   - `nwdaf_accuracy_policy_degradation_signal`
   - `nwdaf_accuracy_policy_chronic_eligible`
   - `nwdaf_accuracy_policy_chronic_signal`
5. 圖表疊加需求
   - degradation panel 需能取到：
     - `primaryMetric current`
     - `mean`
     - `std`
     - `zscore`
     - `baselineReady`
     - `degradationHits`
     - `retrain start/done`
   - chronic panel 需能取到：
     - `trafficScale`
     - `chronicValue`
     - `baselineReady`
     - `chronicHits`
     - `retrain start/done`
6. `hitReason` 不建議直接作為 metric label
   - 建議以 event log / annotation / edge label 表達

### C. Grafana Panels

目標檔案：

- `5g-viz/grafana_setup.py`
- `docs/5g-viz/design/grafana/rendering.md`

建議總數：**3 個主圖區塊**。

建議配置：

1. Traffic panel
   - actual vs predicted
2. Degradation panel
   - `Primary metric`
   - `mean / std`
   - `z-score`
   - overlay：
     - `baselineReady`
     - `degradationHits`
     - `retrain start / done`
3. Chronic panel
   - `TrafficScale`
   - `ChronicValue`
   - overlay：
     - `baselineReady`
     - `chronicHits`
     - `retrain start / done`

補充原則：

- `degradation` 與 `chronic` 不合併
  - 因為兩者代表不同 retrain path
- `degradation` 與 `chronic` 不並排
  - 因為各自都會再拆子圖，並排會太擠
- `baselineReady`、`hits`、`retrain start/done` 都應以 overlay / annotation 掛在 decision panel 上
  - 不另開獨立圖
- `Primary metric`、`mean / std`、`z-score`、`chronicValue` 建議都來自 `accuracy_policy`
  - 避免 `accuracy_scope` 比 policy evaluation 更早出現，造成主圖不同步
- 展示層應把 `model` label 聚合掉，讓切模型後每個 `scope` 仍沿用同一條線與同一個顏色
- `z-score`、`chronicValue`、`trafficScale` 都應提供可配置的 threshold 虛線
- 若未來想做 drill-down，再從這三張主圖延伸，不在本次主版面處理

### D. Topology Reactions

目標檔案：

- `5g-viz/profiles/default/topology.yaml`
- `docs/5g-viz/design/frontend/topology.md`

具體修改：

1. `accuracy_scope`
   - 新增 reaction，表示 AnLF scope metric report 完成
2. `accuracy_policy`
   - 新增 reaction：
     - `flash_edge: { from: nwdaf_mtlf, to: nwdaf_mtlf, label: "{policy_label}" }`
   - 但實作上應允許 event 自帶 suppression flag
     - 沒有變動的 policy round 不應出現在 topology
3. `retrain_trigger`
   - label 可調整為 `RetrainTrigger {reason}`
4. 刪除 `threshold_breach`

### E. Replay / Pseudo-Live

目標檔案：

- `5g-viz/main.py`
- `5g-viz/metric_player.py`
- `docs/5g-viz/design/dvr/replay.md`
- `docs/5g-viz/design/backend/metrics.md`

具體修改：

1. backfill 加入 `accuracy_scope` 與 `accuracy_policy` 的 metric 投影
2. pseudo-live 共享同一套事件到 metrics 的映射
3. `model_swap` 對 scope/policy series 的 cut-off 策略需明確定義
   - scope/policy metrics 在 `model_swap` 寫 NaN cut-off

### F. 文件對齊

目標檔案：

- `docs/5g-viz/design/backend/metrics.md`
- `docs/5g-viz/design/overview/event-schema.md`
- `docs/5g-viz/design/features/nwdaf-ml-cycle.md`
- 必要時補 `docs/5g-viz/design/grafana/README.md`

具體修改：

1. 重寫 NWDAF accuracy monitoring 的 canonical 描述
2. 明確區分：
   - scope-level metrics
   - policy-level decision data
   - retrain lifecycle events
3. 明確說明 `accuracy` 與 `threshold_breach` 已移除
4. 明確說明主顯示為 `traffic / degradation / chronic`

## 實作順序

### Phase 1: 事件模型與 parser 對齊

- 新增 `accuracy_scope`
- 新增 `accuracy_policy`
- 擴充 `retrain_trigger`
- 刪除 `accuracy`
- 刪除 `threshold_breach`

### Phase 2: live metrics 與 topology 對齊

- 新增 scope metrics
- 新增 policy metrics
- 更新 topology reactions

### Phase 3: Grafana 對齊

- 收斂為 `traffic / degradation / chronic` 三類主圖
- 將 `baselineReady`、`hits`、`retrain start/done` 疊到 degradation / chronic

### Phase 4: replay / pseudo-live 對齊

- backfill 與 `MetricPlayer` 支援新 metrics
- 定義 model swap cut-off 行為

### Phase 5: design docs 對齊

- 更新 canonical design docs

## 風險與注意事項

1. label cardinality
   - `scope` 與 `model` 會明顯增加 series 數量
   - `metricsToRecord` 可配置，意味著 parser 與 panel 需要容忍 metric 集合變動
2. panel 過多
   - 需要控制 Grafana layout；本方案建議固定為 3 張主圖
3. replay 一致性
   - 若 live / replay 對 `model_swap` cut-off 處理不同，圖表仍會分岔
4. 舊 log 相容
   - 這次直接切到新版模型，不再保留對舊 `accuracy` 的依賴

## 驗收標準

1. 讀取新版 NWDAF log 時，event stream 中可見：
   - `accuracy_scope`
   - `accuracy_policy`
   - `retrain_trigger`
2. topology 可明確區分：
   - AnLF scope metric report
   - MTLF policy evaluation
   - retrain trigger / lifecycle
3. Grafana 共有 3 張主圖，且可看到：
   - traffic
   - degradation
   - chronic
   - 且 `baselineReady`、`hits`、`retrain start/done` 已正確疊加在 degradation / chronic 上
4. replay / pseudo-live 與 live 的主要圖表語意一致
5. `docs/5g-viz/design/` 完成更新，且不再包含 `accuracy` 與 `threshold_breach` 作為監控主事件

## 建議本次實作切面

若本輪要控制範圍，建議優先完成：

1. parser + event schema
2. topology warning edge 改綁 `accuracy_policy`
3. Grafana 收斂成三類主圖
4. replay / pseudo-live 對齊新 policy metrics

可延後項目：

- 更細的 Grafana template variable 與 panel drill-down
- 將 `metricsToRecord` 的任意新 metric 做成更通用的 panel factory
