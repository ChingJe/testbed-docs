# Traffic Chart

本文從功能視角描述 `5g-viz` 目前的流量圖表如何跨越 parser、metrics、Prometheus、Grafana 與前端 iframe。

若要看各子系統的細部契約，可再對照：

- [`../overview/event-schema.md`](../overview/event-schema.md)
- [`../backend/metrics.md`](../backend/metrics.md)
- [`../grafana/rendering.md`](../grafana/rendering.md)
- [`../frontend/grafana-embed.md`](../frontend/grafana-embed.md)
- [`../reference/env-config.md`](../reference/env-config.md)

## 1. 功能範圍

這條功能線處理的是前端 `#charts` 區塊中與 NWDAF 流量預測有關的圖表，包括：

- 每個 `GRAFANA_GROUPS` 對應的一張 traffic panel
- 一張 `Model Deviation` panel
- traffic panel 上方共用的 `Retrain` annotation

這裡的「traffic chart」不是前端自行畫的折線圖，而是嵌在畫面中的 Grafana dashboard。

## 2. 這條功能真正依賴哪些事件

traffic chart 並不是直接從 log 畫出來，而是經過 event 與 metrics 兩層轉換。

目前最直接相關的 event type 有：

| Event | 作用 |
|---|---|
| `aggregated_slot` | 產生 ground truth UL / DL metrics |
| `ml_inference` | 產生 predicted UL / DL metrics |
| `accuracy` | 產生 `nwdaf_deviation` |
| `retrain_trigger` | 遞增 `nwdaf_retrain_total`，供 annotation 使用 |
| `model_swap` | live 路徑會清除舊 deviation series |

如果這些事件沒有被 parser 正確產生，後面的 Prometheus 與 Grafana 就不會有對應圖形。

## 3. 端到端資料路徑

### 1. Log 進入 parser

collector 從 VM tail 出來的 log line 先進 `parser.py`，再由 `rules/nwdaf.py` 轉成結構化事件。

其中對 traffic chart 最關鍵的 payload 欄位是：

- `aggregated_slot.sub_id`
- `aggregated_slot.target`
- `aggregated_slot.ul_vol` / `dl_vol`
- `ml_inference.sub_id`
- `ml_inference.target`
- `ml_inference.ul_vol` / `dl_vol`
- `accuracy.model`
- `accuracy.deviation`

### 2. Event 投影成 metrics

live 模式下，`main.py` 會在每筆事件進入主處理流程後呼叫 `_update_metrics(event)`。

目前會寫進 Prometheus 的主圖相關 metrics 是：

| Metric | Labels | 來源事件 |
|---|---|---|
| `nwdaf_ground_truth_ul_bytes` | `session`、`sub_id`、`target` | `aggregated_slot` |
| `nwdaf_ground_truth_dl_bytes` | `session`、`sub_id`、`target` | `aggregated_slot` |
| `nwdaf_predicted_ul_bytes` | `session`、`sub_id`、`target` | `ml_inference` |
| `nwdaf_predicted_dl_bytes` | `session`、`sub_id`、`target` | `ml_inference` |
| `nwdaf_deviation` | `session`、`model` | `accuracy` |
| `nwdaf_retrain_total` | `session` | `retrain_trigger` |

這代表 traffic chart 看的不是 event log 本身，而是這批 event 投影出來的 time series。

### 3. Prometheus 保存時間軸

live 時，Prometheus 透過 scrape `/metrics` 持續保存目前 app 記憶體中的 gauge / counter 值。

replay 時，backend 不走 scrape 路徑重建歷史，而是把 replay session 的 metric 樣本以 remote write 寫回 Prometheus，保留原始時間戳。播放中的 pseudo-live 則會把同一批事件重新映射到現在時間附近，再寫成另一個 pseudo session。

### 4. Grafana dashboard 查 Prometheus

`grafana_setup.py` 在啟動時動態建立 dashboard。每個 group 的 traffic panel 都固定查四條 PromQL：

```promql
sum by (target)(nwdaf_ground_truth_ul_bytes{session="$session",target="group=<group_id>"})
sum by (target)(nwdaf_ground_truth_dl_bytes{session="$session",target="group=<group_id>"})
sum by (target)(nwdaf_predicted_ul_bytes{session="$session",target="group=<group_id>"} offset 5s)
sum by (target)(nwdaf_predicted_dl_bytes{session="$session",target="group=<group_id>"} offset 5s)
```

另外還有：

- deviation panel：查 `nwdaf_deviation{session="$session"}` 的最新一批 model series
- retrain annotation：查 `idelta(nwdaf_retrain_total{session="$session"}[15s]) > 0`

### 5. 前端只負責把 dashboard 嵌進來

前端 `events.js` 啟動時會讀：

- `GET /api/grafana-config`
- `GET /api/session-info`

之後建立單一 iframe，並透過 URL 控制：

- `var-session=<session>`
- `from`
- `to`
- `refresh=5s` 或關閉 refresh

所以前端這一層不理解 PromQL，也不自己算折線值；它只在不同模式間切換要看的 session 與時間窗。

## 4. Cross-layer 契約

這條功能線有幾個不能鬆動的契約。

### `target` 必須對上 `group=<group_id>`

Grafana traffic panel 不是靠 `sub_id` 分圖，而是靠：

```text
target="group=<group_id>"
```

因此 `aggregated_slot` 與 `ml_inference` 事件裡的 `target` 若不是這個格式，或值不在 `GRAFANA_GROUPS` 內，對應 panel 就不會出現資料。

### `GRAFANA_GROUPS` 決定 panel 數量與名稱

`GRAFANA_GROUPS` 來自 profile `.env`。它同時決定：

- dashboard 會建立多少張 traffic panel
- 每張 panel 查哪一個 `group=<group_id>`
- live session `meta.json` 會記錄哪些 groups

因此這不是單純的前端顯示清單，而是 dashboard 結構的一部分。

### `session` label 是整個圖表功能的主鍵

這條功能線不論在 live、replay backfill 或 pseudo-live，都靠 `session` label 區分資料來源。

Grafana dashboard 的 session variable 目前查的是：

```promql
label_values(nwdaf_ground_truth_ul_bytes, session)
```

也就是說：

- 只有出現在 ground truth UL metric 裡的 session 才能被 Grafana 選到
- deviation-only 或 retrain-only session 不會單獨出現在下拉選單

### prediction 線不是改 timestamp，而是 query 固定 `offset 5s`

預測線的「向左對齊一個 slot」是 Grafana query 層的呈現策略，不是 metrics 寫入時真的把樣本時間戳往前改。

因此這個 feature 的視覺語意有一部分是由 dashboard query 決定，而不是單靠 event 或 metrics 本身決定。

## 5. Live 模式下的圖表行為

live mode 時，traffic chart 的資料流是：

```text
log -> event -> prometheus_client metrics -> /metrics -> Prometheus scrape -> Grafana iframe
```

前端在 live 且停留於即時點時，iframe 使用：

```text
var-session=<orig_session>
from=now-<window>m
to=now
refresh=5s
```

這時使用者看到的是：

- ground truth / prediction 持續向右延伸
- deviation panel 持續更新目前 session 的最新 model 偏差
- retrain annotation 隨 `nwdaf_retrain_total` 的變化出現

## 6. Live DVR 下的圖表行為

當 live 使用者按下 Pause、拖曳 timeline 或離開即時點時，拓樸與圖表會分別走不同資料路徑：

- topology / log：主要來自前端 `_events` 緩衝
- traffic chart：仍然來自 Prometheus / Grafana

這時 iframe 會切成：

```text
var-session=<orig_session>
from=<timelinePos - window>
to=<timelinePos>
refresh=off
```

也就是說，live DVR 下的圖表不是用前端 buffer 自行重建折線，而是改成查同一個 live session 在 Prometheus 內已經被 scrape 到的歷史樣本。

## 7. Replay 模式下的兩條圖表路徑

replay 的 traffic chart 有兩條路徑。

### Pause / Scrub：Replay Backfill

replay 啟動後，backend 先把原始 session 的 metric 事件 backfill 到 Prometheus。此時 iframe 仍看原始 session：

```text
var-session=<orig_session>
from=<timelinePos - window>
to=<timelinePos>
refresh=off
```

這條路徑讓 replay 在靜態觀察某個時間點時，可以直接查錄製當時的歷史曲線。

### Play：Replay Pseudo-Live

當 replay 進入 `PLAYING`，前端會呼叫 `/api/replay/play`，backend 啟動 `MetricPlayer`，建立新的 pseudo session。

之後 iframe 切到：

```text
var-session=<pseudo_session>
from=now-<window>m
to=now
refresh=5s
```

這時使用者體感上看到的是「正在播放的即時圖」，但實際上 Grafana 查的是另一批被重新映射到現在時間附近的 metrics。

### 改 chart window 時為什麼要重啟 pseudo-live

如果 replay 正在播放，前端改 chart window 不只是換 iframe URL，而是會：

1. 暫停目前 pseudo-live
2. 用目前 playhead 與新 window 重新呼叫 `/api/replay/play`
3. 讓 backend 依新的 trailing window 重新 pre-seed pseudo-live metrics

這是因為 pseudo-live 的圖表不是單純查現成歷史資料，而是和目前播放視窗寬度耦合。

## 8. 使用者真正看到的幾種差異

雖然畫面上都叫 traffic chart，但三種模式看到的資料語意不同：

| 模式 | Grafana 查的是什麼 |
|---|---|
| live | 當前 app 內 gauges / counters 被 Prometheus 持續 scrape 的結果 |
| replay paused / scrubbed | 原始 replay session 被 backfill 到 Prometheus 的歷史樣本 |
| replay playing | pseudo session 被 `MetricPlayer` 映射到現在時間的樣本 |

因此相同一條曲線在不同模式下，外觀看起來相似，但資料來源並不相同。

## 9. 目前已知限制

- `model_swap` 造成的 deviation series 清除目前只有 live handler 會真實執行；replay backfill 與 pseudo-live 沒有完整重播 delete 語意
- Grafana session variable 只看 `nwdaf_ground_truth_ul_bytes`，若未來某個 session 缺少 ground truth 樣本，就可能無法在 dashboard 被選到
- replay 啟動時 `start.sh` 會清空本機 Prometheus TSDB，因此 Grafana 能看到的 session 範圍取決於這次啟動後寫進去的資料
- 前端只控制 iframe URL，不理解 panel 內部的 legend toggle、zoom 或 query inspector；Grafana 內互動不會回寫到 5g-viz 控制列
- prediction 對齊目前固定寫死 `offset 5s`；若未來 slot 長度改變，這條 feature 的視覺語意也需要一起調整
