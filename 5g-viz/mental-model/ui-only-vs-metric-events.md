# UI-only Vs Metric Events

## 這份文件在說什麼

本文件說明一個常見混淆：

- 有些 event 只會改變 Topology / Event Log
- 有些 event 也會再被投影成 metrics，進到 Prometheus / Grafana

這個邊界若不先分清楚，很容易把「Grafana 沒線」誤判成系統壞掉。

## 簡介

可以先把 event 分成兩類：

```text
UI-only event   = 影響畫面與事件序列，不進 metrics
metric event    = 既影響畫面或事件序列，也進 Prometheus / Grafana
```

## 目前會進 metrics 的主要 event

目前最重要的 metric event 包括：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `model_swap`

這些事件會被 metrics handler 處理，最後影響：

- traffic panels
- deviation panel
- retrain counter
- 某些 live deviation series 的清理

## 目前只影響 UI 的主要 event

目前常見的 UI-only event 包括：

- `sbi_call`
- `upf_volume`
- `threshold_breach`
- `retrain_done`
- `adrf_stored`
- `adrf_retrieval_start`
- `adrf_retrieval_notify`
- `adrf_fetch`

這些事件仍然有價值，因為它們會影響：

- Topology 的 edge / pulse / 持久 class
- Event Log 的事件序列

只是它們不會直接變成 Prometheus 樣本。

## 為什麼這個邊界很重要

### Topology 有反應，不代表 Grafana 也一定有反應

只要發生的是 UI-only event，就可能出現：

- edge 在閃
- Event Log 有事件
- Grafana 沒有任何新線

這是資料層分工的結果，不是自動代表 bug。

### Event Log 很熱鬧，不代表 chart 一定很多變化

Event Log 收錄的是結構化事件。  
Grafana 只吃其中一部分會進 metrics 的事件。

### 某些重要操作同時跨兩層

有些 event 同時影響 UI 與 metrics，例如：

- `ml_inference`
- `accuracy`
- `retrain_trigger`

這類事件比較容易在 Topology、Event Log、Grafana 三個區塊都留下痕跡。

## 幾個代表性例子

### `sbi_call`

`sbi_call` 主要用來驅動：

- 邊動畫
- pulse
- event log 記錄

它不直接進 metrics。

### `retrain_trigger`

`retrain_trigger` 同時影響：

- Topology 上的 `retraining` 狀態
- Event Log
- `nwdaf_retrain_total`

這是一種跨 UI 與 metrics 的事件。

### `retrain_done`

`retrain_done` 會改變 Topology 的持久狀態，但本身不直接寫 metrics。

因此 retraining 狀態消失，不代表 Grafana 一定會多出一條新線。

### `model_swap`

`model_swap` 是少數容易誤會的例子。

它在 UI 上有動作，在 live metrics path 也有作用，但這個作用不是增加一條新數值線，而是清理舊的 deviation series。

因此看到的結果不一定是「多了一條曲線」，而可能是：

- 舊 series 消失
- dashboard 只留下新的 model 對應曲線

## 怎麼判斷某個現象應先看哪一層

### 先看 Topology / Event Log

適合的問題：

- 某條 SBI flow 是否真的發生
- 某個 node 為什麼突然 pulse
- 某段 replay 歷史裡到底先後發生了哪些事件

### 再看 Grafana

適合的問題：

- 數值是否改變
- prediction 是否偏離 ground truth
- deviation 是否上升
- retrain counter 是否增加

## 對照閱讀

- Topology：[`../ui-workflows/topology.md`](../ui-workflows/topology.md)
- Event Log：[`../ui-workflows/event-log.md`](../ui-workflows/event-log.md)
- Grafana：[`../ui-workflows/grafana.md`](../ui-workflows/grafana.md)
- deeper reference：
  - [`../design/overview/event-schema.md`](../design/overview/event-schema.md)
  - [`../design/backend/metrics.md`](../design/backend/metrics.md)
