# NWDAF ML Cycle

> Historical note: this feature flow still references the older replay pseudo-live mental model in later sections. The event/metric relationships remain useful, but replay playback now stays on the original session with historical relative Grafana queries.

本文從功能視角描述 `5g-viz` 目前如何把 NWDAF 的資料聚合、推論、準確度檢查、重訓、ADRF 取數與模型切換這條循環轉成可觀察的事件、拓樸狀態與圖表。

若要看各子系統細節，可再對照：

- [`../overview/event-schema.md`](../overview/event-schema.md)
- [`../backend/metrics.md`](../backend/metrics.md)
- [`../grafana/rendering.md`](../grafana/rendering.md)
- [`../frontend/topology.md`](../frontend/topology.md)
- [`../dvr/replay.md`](../dvr/replay.md)

## 1. 功能範圍

這條 feature 關心的是 NWDAF 內部的 ML 監控與重訓閉環，包含：

- UPF 週期性流量回報進入 NWDAF
- AnLF 聚合出最新 slot 並做一次 ML inference
- AnLF 對目前模型做 accuracy 檢查
- MTLF 判斷是否 threshold breach、是否需要重訓
- 重訓前是否走 ADRF retrieval 路徑
- Daisy 訓練完成後回到 MTLF，再 hot-swap 到 AnLF

這條循環會同時影響三個觀察面：

- topology：AnLF / MTLF / ADRF / UPF 的動畫與 class 狀態
- Grafana：ground truth、prediction、deviation、retrain annotation
- DVR / replay：這整條循環的事件與衍生 metrics 如何被重播

## 2. 主循環總覽

目前在 `5g-viz` 中，這條循環可概括為：

```text
UPF volume notify
  -> aggregated_slot
  -> ml_inference
  -> accuracy
  -> threshold_breach
  -> retrain_trigger
  -> [optional ADRF retrieval path]
  -> retrain_done
  -> model_swap
```

其中有兩件事要先分清楚：

- `upf_volume` 比較像輸入資料抵達 NWDAF 的訊號
- 真正進 Grafana 主圖的，是 `aggregated_slot`、`ml_inference`、`accuracy`、`retrain_trigger`

所以使用者看到的同一個「ML cycle」，在系統裡其實是由 event、reaction 與 metric handler 三層共同組成。

## 3. 事件如何把這條循環拆開

### `upf_volume`

這是 NWDAF 收到 UPF 週期性資料回報時的事件。它會保留：

- `upf`
- `ip`
- `ul_bytes`
- `dl_bytes`

主要作用是：

- 在拓樸上畫出 `UPF -> NWDAF` notify 動畫
- 表示新一輪資料已經進入 NWDAF

它本身不直接寫入 Prometheus metrics。

### `aggregated_slot`

這是 AnLF 對每個 subscription / group 聚合出最新 traffic slot 後產生的事件，保留：

- `sub_id`
- `target`
- `ts`
- `ul_vol`
- `dl_vol`
- `ul_thr`
- `dl_thr`

這是 ground truth 曲線的主要來源，也是前端第一次遇到它時會確保 Grafana iframe 掛上的事件。

### `ml_inference`

這是 AnLF 對同一輪資料做 prediction 後產生的事件，保留：

- `sub_id`
- `target`
- `ul_vol`
- `dl_vol`
- `confidence`

它的可見效果分兩層：

- topology：讓 AnLF node pulse，表示正在推論
- Grafana：寫進 predicted UL / DL metrics

### `accuracy`

這是 AnLF 對目前模型做 accuracy 檢查後的事件，保留：

- `model`
- `deviation`
- `accuracy`
- `samples`

它同時影響：

- topology：畫 `AnLF -> MTLF` 的 `AccuracyReport` 邊
- Grafana：更新 `nwdaf_deviation`

### `threshold_breach`

當 MTLF 判斷 accuracy 已經達到重訓條件時，會產生：

- `n`
- `total`

這目前只影響 topology：

- MTLF 上畫 self-edge，label 會帶 `ThresholdBreach {n}/{total}`
- MTLF 發出橘色 pulse

它不直接進 metrics。

### `retrain_trigger`

當 MTLF 真正把訓練任務送進 Daisy 時，會產生：

- `model`
- `tid`

這是整條 cycle 裡最重要的切換點之一，因為它同時影響：

- topology：MTLF 加上 `retraining` class，並畫出 `RetrainTrigger` self-edge
- Grafana：`nwdaf_retrain_total` counter 增加，並寫 `nwdaf_retrain_start_event` pulse

### ADRF 相關事件

若系統啟用了 ADRF 路徑，重訓前後還會出現：

- `adrf_stored`
- `adrf_retrieval_start`
- `adrf_retrieval_notify`
- `adrf_fetch`

它們目前主要作用在 topology：

- `NWDAF -> ADRF` store
- `MTLF -> ADRF` retrieval subscribe
- `ADRF -> MTLF` retrieval notify
- `MTLF -> ADRF` retrieval request

這讓使用者能看見 retrain 前的資料保存與取回流程，但這些事件本身不寫入 Prometheus。

### `retrain_done`

當 Daisy 非同步訓練完成並回到 MTLF 時，會產生 `retrain_done`。

它主要作用在 topology：

- 移除 `MTLF.retraining` class
- 畫出 `MTLF -> AnLF` 的 `ModelProvision` 邊

這是視覺上「重訓完成、模型準備回灌」的節點。

### `model_swap`

最後 AnLF 完成 model hot-swap 時，會產生：

- `model_id`

這個事件同時有兩層效果：

- topology：在 AnLF 畫出 `ModelDeploy` loop edge，並做綠色 pulse
- metrics：live 路徑會刪除目前 session 已追蹤的舊 `nwdaf_deviation{session,model}` series

所以它不只是視覺上代表「部署成功」，也會改變 Grafana deviation panel 後續看到的 series 集合。

## 4. 這條循環如何同時投影到 topology 與 Grafana

### Topology 面

拓樸層主要看的是 reaction：

- `upf_volume`：`UPF -> NWDAF`
- `ml_inference`：AnLF pulse
- `accuracy`：`AnLF -> MTLF`
- `threshold_breach`：MTLF self-edge + 橘色 pulse
- `retrain_trigger`：MTLF `retraining` class + self-edge
- `retrain_done`：移除 `retraining` + `MTLF -> AnLF`
- `model_swap`：AnLF loop deploy
- ADRF 事件：`NWDAF <-> ADRF`、`MTLF <-> ADRF`

因此 topology 呈現的是「控制與狀態轉換」，不是圖表值本身。

### Grafana 面

圖表層主要看的是 metrics handler：

| Event | 寫入的 metric |
|---|---|
| `aggregated_slot` | `nwdaf_ground_truth_ul_bytes`、`nwdaf_ground_truth_dl_bytes` |
| `ml_inference` | `nwdaf_predicted_ul_bytes`、`nwdaf_predicted_dl_bytes` |
| `accuracy` | `nwdaf_deviation` |
| `retrain_trigger` | `nwdaf_retrain_total`、`nwdaf_retrain_start_event` |
| `retrain_done` | `nwdaf_retrain_done_event` |
| `model_swap` | 終止舊 deviation series（live=`remove()`；replay=`NaN` cut-off） |

也就是說，Grafana 主圖看到的是：

- traffic 與 prediction 曲線
- deviation 變化
- retrain annotation

但它看不到：

- `threshold_breach`
- ADRF retrieval 動畫
- `retrain_done` 的 provision 邊

這些只存在於 topology 與 event log。

## 5. 為什麼 ground truth 與 prediction 能同框比較

這條功能線最容易混淆的地方，是：

- `aggregated_slot` 與 `ml_inference` 在同一輪處理附近發生
- 但兩者描述的時間語意不同

目前 dashboard 對 prediction query 固定加：

```promql
offset 5s
```

因此使用者在 Grafana 上看到的 prediction 線，不是寫入時真的把樣本時間往前改，而是 query 在顯示時間 `T` 時實際查 `T-5s` 的 prediction 值，讓它和對應 slot 的 ground truth 在時間軸上視覺對齊。

也就是說，這條 feature 的「模型預測 vs 真實值」比較，不只靠 event / metric 本身，還依賴 dashboard rendering 策略。

## 6. `target` 與 `session` 如何決定圖表切片

這條循環進 Grafana 時，主要靠兩個 label 被切分：

- `session`
- `target`

### `target`

`aggregated_slot.target` 與 `ml_inference.target` 目前都必須長成：

```text
group=<group_id>
```

Grafana traffic panel 會用它對應到 `GRAFANA_GROUPS` 的某一張 panel。

### `session`

整條循環在 live、replay 與 pseudo-live 都會被打上某個 `session` label，Grafana 也靠這個 label 切換目前看的那一輪 cycle。

因此同樣一套 ML cycle，在不同 session 下會變成不同批 time series。

## 7. 與 replay / pseudo-live 的關係

這條 feature 在 replay 時會分成兩條路徑。

### Replay Backfill

replay 啟動後，backend 會從 `events.jsonl` 篩出 metric 相關事件：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `model_swap`

再把能重建成樣本的部分 backfill 到 Prometheus。

這讓 replay paused / scrubbed 時：

- topology 由 `_events` 緩衝重建
- Grafana 由原始 replay session 的 backfilled metrics 提供圖表

### Replay Pseudo-Live

播放時，`MetricPlayer` 會把同一批 metric event 重新映射到現在時間附近，形成 pseudo session。

因此使用者在 replay 播放時看到的是：

- topology：事件再次 live-like 地 flash / pulse
- Grafana：pseudo session 的 ground truth / prediction / deviation / retrain metrics

### `model_swap` 的對齊方式

這條 feature 在 replay 下原本有一個重要差異：live 會移除舊 deviation series，但 replay remote
write 無法直接呼叫 exporter `remove()`。

- live 模式：`model_swap` 真的刪掉舊 deviation series
- replay backfill / pseudo-live：在 `model_swap` 對 active models 寫 `NaN` cut-off sample

因此使用者看到的結果已對齊：舊 model 的 deviation 線會在 swap 當下截斷，後面保留新模型部署後的
cold-start gap。

## 8. 使用者看到的整體效果

從使用者角度看，這條 feature 最終會呈現成三種互補視角：

| 視角 | 看到什麼 |
|---|---|
| topology | 哪個 NF 正在推論、警告、重訓、取數、部署模型 |
| Grafana | ground truth / prediction / deviation / retrain annotation 的時間序列 |
| event log | 每一輪 cycle 的結構化事件與 payload 細節 |

這也是為什麼這條 feature 不能只看某一層：

- 只看 topology，不知道 prediction 與 deviation 數值怎麼變
- 只看 Grafana，不知道 retrain 前後到底走了哪些控制路徑
- 只看 event log，則缺少長期趨勢與整體狀態感

## 9. 目前限制

- `threshold_breach` 與 ADRF 事件目前只有拓樸語意，沒有對應 metrics，因此無法直接在 Grafana 上量化觀察
- retrain annotation 現在靠 `nwdaf_retrain_start_event` / `nwdaf_retrain_done_event` pulse gauge，
  不再從 `nwdaf_retrain_total` 間接推導
- replay backfill 與 pseudo-live 以 `NaN` cut-off 模擬 `model_swap` 的 series terminate 語意，
  和 live 的畫面效果對齊，但底層機制仍不同
- parser 目前只保留 `aggregated_slot` 內前端與 metrics 真正需要的欄位，不是完整保留所有原始 log 欄位
- Grafana prediction 對齊固定寫死 `offset 5s`；若未來 slot 長度或推論步長變化，這條 feature 的比較語意也要一起調整
