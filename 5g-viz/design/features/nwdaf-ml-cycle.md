# NWDAF ML Cycle

本文從功能角度描述 `5g-viz` 如何把 NWDAF 的資料聚合、推論、準確度檢查、重訓、ADRF 取數與模型切換這條循環轉成可觀察的事件、拓樸狀態與圖表。

若要看子系統細節，可再對照：

- [`../overview/event-schema.md`](../overview/event-schema.md)
- [`../backend/metrics.md`](../backend/metrics.md)
- [`../grafana/rendering.md`](../grafana/rendering.md)
- [`../frontend/topology.md`](../frontend/topology.md)
- [`../dvr/replay.md`](../dvr/replay.md)

## 1. 功能範圍

這條 feature 關心的是 NWDAF 內部的 ML 監控與重訓閉環，包含：

- UPF 週期性流量回報進入 NWDAF
- AnLF 聚合出最新 slot 並做 ML inference
- AnLF 對目前模型做 accuracy 檢查
- MTLF 判斷是否 threshold breach、是否需要重訓
- 重訓前是否走 ADRF retrieval 路徑
- Daisy 訓練完成後回到 MTLF，再 hot-swap 到 AnLF

這條循環會同時影響三個觀察面：

- topology：AnLF / MTLF / ADRF / UPF 的動畫與 class 狀態
- Grafana：ground truth、prediction、deviation、retrain annotation
- DVR / replay：這整條循環如何被重播與重新觀察

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

其中：

- `upf_volume` 比較像「新資料進入 NWDAF」的訊號
- 真正進 Grafana 主圖的核心事件是：
  - `aggregated_slot`
  - `ml_inference`
  - `accuracy`
  - `retrain_trigger`
  - `retrain_done`
  - `model_swap`

## 3. 事件如何把這條循環拆開

### `upf_volume`

表示 NWDAF 收到 UPF 週期性資料回報。

它主要影響 topology：

- 畫出 `UPF -> NWDAF` notify 動畫
- 表示新一輪資料已進入 NWDAF

它本身不直接寫 Prometheus metrics。

### `aggregated_slot`

表示 AnLF 聚合出最新 traffic slot。

它是：

- ground truth 曲線的主要來源
- traffic chart 建立時最關鍵的事件之一

### `ml_inference`

表示 AnLF 對同一輪資料做 prediction。

它同時影響：

- topology：AnLF pulse
- Grafana：predicted UL / DL metrics

### `accuracy`

表示 AnLF 對目前模型做準確度檢查。

它同時影響：

- topology：`AnLF -> MTLF` 的 accuracy report 邊
- Grafana：`nwdaf_deviation`

### `threshold_breach`

表示 MTLF 判定目前 accuracy 已進入重訓條件。

目前主要只影響 topology：

- MTLF self-edge
- 橘色 pulse

它不直接進 Prometheus。

### `retrain_trigger`

表示 MTLF 真正把重訓任務送進 Daisy。

它會同時影響：

- topology：MTLF 加上 `retraining` class，並畫 retrain edge
- Grafana：
  - `nwdaf_retrain_total`
  - `nwdaf_retrain_start_event`

### ADRF 相關事件

若系統啟用了 ADRF 路徑，會看到：

- `adrf_stored`
- `adrf_retrieval_start`
- `adrf_retrieval_notify`
- `adrf_fetch`

這些事件主要用於 topology，讓使用者看見 retrain 前的資料保存與取回流程；它們本身不直接進 Prometheus。

### `retrain_done`

表示 Daisy 非同步訓練完成並回到 MTLF。

它主要作用在：

- topology：移除 `MTLF.retraining`
- Grafana：寫 `nwdaf_retrain_done_event`

### `model_swap`

表示 AnLF 完成 model hot-swap。

它同時影響：

- topology：AnLF deploy loop edge + 綠色 pulse
- metrics：終止目前 deviation series 的舊模型部分

## 4. 這條循環如何同時投影到 topology 與 Grafana

### Topology 面

拓樸主要看 reactions：

- `upf_volume`：`UPF -> NWDAF`
- `ml_inference`：AnLF pulse
- `accuracy`：`AnLF -> MTLF`
- `threshold_breach`：MTLF self-edge + 橘色 pulse
- `retrain_trigger`：MTLF `retraining` class + edge
- `retrain_done`：移除 `retraining` + `MTLF -> AnLF`
- `model_swap`：AnLF deploy loop
- ADRF 事件：`NWDAF <-> ADRF`、`MTLF <-> ADRF`

因此 topology 呈現的是控制與狀態轉換，不是圖表值本身。

### Grafana 面

圖表主要看 metric handlers：

| Event | 寫入的 metric |
|---|---|
| `aggregated_slot` | ground truth UL / DL |
| `ml_inference` | predicted UL / DL |
| `accuracy` | `nwdaf_deviation` |
| `retrain_trigger` | `nwdaf_retrain_total`、`nwdaf_retrain_start_event` |
| `retrain_done` | `nwdaf_retrain_done_event` |
| `model_swap` | 終止舊 deviation series |

Grafana 看不到：

- `threshold_breach`
- ADRF retrieval 動畫
- `retrain_done` 的 provision 邊

這些只存在於 topology 與 event log。

## 5. 為什麼 ground truth 與 prediction 能同框比較

最容易混淆的地方是：

- `aggregated_slot` 與 `ml_inference` 在同一輪附近發生
- 但兩者描述的時間語意不同

目前 dashboard 對 prediction query 固定加：

```promql
offset 5s
```

因此使用者看到的 prediction 線，不是寫入時真的把樣本時間往前改，而是 Grafana 在顯示時間 `T` 時查 `T-5s` 的 prediction 值，讓它和對應 slot 的 ground truth 視覺對齊。

## 6. `target` 與 `session` 如何決定圖表切片

### `target`

`aggregated_slot.target` 與 `ml_inference.target` 目前都必須長成：

```text
group=<group_id>
```

Grafana traffic panels 會用它對應到 profile `config.yaml` 裡的 `grafana.groups`。

### `session`

整條循環進 Grafana 時，都會依附在某個 `session` label 上。

現在的 replay 模型也已收斂成：

- replay backfill：把原始 session 寫回 Prometheus
- replay playing：仍查同一個原始 session，只改查 historical relative 時間窗

因此目前沒有 pseudo session，也不再有 chart 專用的第二套 replay runtime。

## 7. 與 replay 的關係

### replay backfill

replay 啟動後，後端會從 `events.jsonl` 篩出 metric-bearing events，將其 backfill 到 Prometheus。

這讓 replay 在 paused / scrubbed 時可以直接查錄製當時的歷史曲線。

### replay playing

播放時：

- topology：前端本地重播事件
- Grafana：查同一個原始 session 的 historical relative 時間窗

因此目前的 replay `PLAYING` 只是觀察同一份歷史資料的另一種時間窗，不再需要 pseudo-live remap。

### `model_swap` 的 replay 對齊方式

live 會直接移除舊 deviation series；replay backfill 則用 `NaN` cut-off sample 終止舊線段。

所以：

- 使用者看到的畫面語意大致一致
- 底層機制仍然不同

## 8. 使用者看到的整體效果

從使用者角度，這條 feature 最終會呈現成三種互補視角：

| 視角 | 看到什麼 |
|---|---|
| topology | 哪個 NF 正在推論、警告、重訓、取數、部署模型 |
| Grafana | ground truth / prediction / deviation / retrain annotation 的時間序列 |
| event log | 每一輪 cycle 的結構化事件與 payload 細節 |

這也是為什麼這條 feature 不能只看某一層。

## 9. 目前限制

- `threshold_breach` 與 ADRF 事件目前只有 topology 語意，沒有對應 metrics
- retrain annotations 目前靠 pulse gauges，而不是從 counter 間接推導
- `model_swap` 在 live 與 replay 的底層實作不同
- prediction 對齊固定使用 `offset 5s`；若未來 slot 長度或推論步長改變，dashboard query 也要一起調整

若看到 pseudo-live、`MetricPlayer` 或 replay speed 的描述，應視為歷史版本設計，不是目前 runtime。
