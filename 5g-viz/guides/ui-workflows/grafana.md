# Grafana

## 這個區塊在做什麼

Grafana 區塊負責顯示 metrics 層的時間序列圖，而不是 event 本身。

主畫面中的這塊內容來自：

- backend 把部分事件轉成 Prometheus metrics
- Prometheus 保存與查詢這些 metrics
- Grafana dashboard 以 iframe 嵌入主畫面

因此，Grafana 區塊適合回答的問題是：

- 流量數值是否在變化
- prediction 與 ground truth 是否對齊
- model deviation 是否正在變化
- retrain 事件是否出現在圖表上

## 這個區塊通常會看到哪些 panel

目前主要有兩類 panel：

### 1. Traffic panels

每個 `grafana.groups` 會對應一張 traffic panel。

每張 panel 主要顯示：

- ground truth UL
- ground truth DL
- predicted UL
- predicted DL

這些 panel 不是靠 `sub_id` 分圖，而是靠 `target="group=<group_id>"` 分圖。

### 2. Model Deviation panel

這張 panel 用來顯示目前 session 下的 deviation 系列。

它與 traffic panel 的差別是：

- traffic panel 主要看 flow / volume 類 metrics
- deviation panel 主要看模型偏差

### 3. Retrain annotation

Retrain 不是獨立 panel，而是加在圖表上的 annotation。

用途是標示：

- retrain 事件何時發生

## Grafana 區塊的資料來源

Grafana 不是直接讀 Event Log，也不是直接讀 Topology。

它的資料層大致是：

```text
event
  -> metrics
  -> Prometheus
  -> Grafana
```

這代表兩件事：

1. 不是所有 event 都能出現在 Grafana
2. Grafana 的更新節奏與 Topology 的動畫節奏不是同一回事

## 哪些 event 會影響 Grafana

目前最直接相關的 event type 包括：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `model_swap`

其中：

- `aggregated_slot` 與 `ml_inference` 主要影響 traffic 線
- `accuracy` 主要影響 deviation panel
- `retrain_trigger` 主要影響 retrain annotation
- `model_swap` 影響 live path 中 deviation series 的清理

若某個事件沒有進 metrics handler，Topology 與 Event Log 可能有反應，但 Grafana 不一定有對應變化。

## Live、Replay Paused、Replay Playing 三種情況

Grafana 區塊最容易誤解的地方就在這裡。

### Live

Live 模式下，Grafana 通常查的是：

- 當前 session
- `now-window ~ now`

因此這時的畫面最接近：

- 正在跟著當前 runtime 更新的 chart

### Replay Paused / Scrubbed

Replay 在停住或拖曳時間點時，Grafana 主要查的是：

- 原始 replay session
- 某個固定時間窗

這時的畫面最接近：

- 錄製當時的歷史圖

### Replay Playing

Replay 進入播放後，Grafana 仍看：

- 原始 replay session
- historical relative time range

這時的畫面最接近：

- 用原始歷史 metrics 做一條會隨著 `now` 平滑滑動的圖

因此 replay `paused` 與 replay `playing` 的差異，主要在查詢時間窗，而不是資料被重新映射到另一個 pseudo session。

## Chart Window 在做什麼

Chart Window 決定的是：

- Grafana 圖表目前顯示的時間窗寬度

在一般情況下，改動 Chart Window 主要會讓 iframe 切到新的時間範圍。

在 replay `playing` 期間，事情稍微不同一些：

- historical relative range 需要用新的 window 重新計算 `from/to`
- 因此改動 Chart Window 不只是單純 reload iframe，也會讓圖表切到新的 relative query

## Grafana 適合看什麼，不適合看什麼

### 適合

- 數值變化
- prediction 與 ground truth 的相對變化
- deviation
- retrain 標記

### 不適合

- 某次事件的即時流向
- 某條 SBI call 的細節
- 某個節點為什麼被加上某個 class

這些內容更適合交給 Topology 與 Event Log。

## 常見誤解

### Topology 有動但 Grafana 沒線

最常見的原因是：

- 發生的是 UI-only event
- 還沒進 metrics 層
- Grafana 正在看另一個 session 或另一個時間窗

### Grafana 沒更新，不代表 backend 完全沒有事件

可能只是：

- 前端現在不在 live chart 視窗
- replay 正停在固定時間點
- 當前事件不是 metrics event

### Replay 播放中的圖，不等於 replay 停住時看到的同一條歷史圖

Replay `playing` 期間會切到另一種查詢時間語意，這是刻意設計，不是 accidental mismatch。

## 對照閱讀

- DVR Controls：[`./dvr-controls.md`](./dvr-controls.md)
- Event History：[`./event-history.md`](./event-history.md)
- deeper reference：
  - [`../../design/frontend/grafana-embed.md`](../../design/frontend/grafana-embed.md)
  - [`../../design/backend/metrics.md`](../../design/backend/metrics.md)
  - [`../../design/features/traffic-chart.md`](../../design/features/traffic-chart.md)
