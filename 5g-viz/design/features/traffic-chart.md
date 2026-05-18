# Traffic Chart

本文從功能角度描述 `5g-viz` 的 traffic chart：哪些事件會進圖、Prometheus 與 Grafana 如何承接它們，以及 live / replay 各自看到的是哪一段時間窗。

若要看子系統契約，可再對照：

- [`../backend/metrics.md`](../backend/metrics.md)
- [`../grafana/rendering.md`](../grafana/rendering.md)
- [`../frontend/grafana-embed.md`](../frontend/grafana-embed.md)
- [`../backend/profiles.md`](../backend/profiles.md)

## 1. 功能範圍

traffic chart 指的是前端 `Chart` 區塊內由 Grafana dashboard 顯示的：

- 各 `group` 的 ground truth / prediction 流量 panel
- `Model Deviation` panel
- `Retrain` annotations

這不是前端自畫折線圖，而是嵌入的 Grafana iframe。

## 2. 直接相關的事件

目前最直接影響 traffic chart 的 event type 有：

| Event | 作用 |
|---|---|
| `aggregated_slot` | 產生 ground truth UL / DL metrics |
| `ml_inference` | 產生 predicted UL / DL metrics |
| `accuracy` | 產生 `nwdaf_deviation` |
| `retrain_trigger` | 更新 `nwdaf_retrain_total` 並觸發 retrain start annotation |
| `retrain_done` | 觸發 retrain done annotation |
| `model_swap` | 終止舊 deviation series |

若 parser 沒有正確產生這些事件，後面的 Prometheus 與 Grafana 就不會有對應圖。

## 3. 端到端資料流

### 1. Log 先轉成事件

collector tail 出來的 log line 先進 parser，再由 `rules/nwdaf.py` 轉成結構化事件。

對 traffic chart 最關鍵的 payload 是：

- `aggregated_slot.sub_id`
- `aggregated_slot.target`
- `aggregated_slot.ul_vol` / `dl_vol`
- `ml_inference.sub_id`
- `ml_inference.target`
- `ml_inference.ul_vol` / `dl_vol`
- `accuracy.model`
- `accuracy.deviation`

### 2. 事件再投影成 metrics

live 模式下，這些事件會被 metric handlers 寫進 `/metrics` exporter state。

replay 模式下，這些事件則會被後端從 `events.jsonl` 重建後，以 remote write backfill 到 Prometheus。

### 3. Grafana dashboard 查 Prometheus

dashboard 目前固定查：

- ground truth UL / DL
- predicted UL / DL
- deviation
- retrain annotations

前端不理解 PromQL，也不自己計算折線值；它只負責控制 iframe 的：

- `var-session`
- `from`
- `to`
- `refresh`

## 4. Cross-layer 契約

### `target` 必須對上 `group=<group_id>`

traffic panel 不是靠 `sub_id` 分圖，而是靠：

```text
target="group=<group_id>"
```

因此 `aggregated_slot.target` 與 `ml_inference.target` 若不符合這個格式，Grafana panel 就不會有資料。

### `grafana.groups` 決定 panel 結構

目前 panel 數量與名稱由 profile `config.yaml` 內的 `grafana.groups` 決定。

它同時影響：

- dashboard 會建立幾張 traffic panel
- 每張 panel 查哪一個 `group=<group_id>`
- session metadata 中可預期的 group 範圍

### `session` label 是圖表主鍵

整個 chart 功能都靠 `session` label 區分不同批資料。

這也是為什麼：

- live run 要建立自己的 session
- replay 要先決定 reuse / overwrite / skip
- Grafana variable 只要切換 session，就能切換整輪歷史圖表

### prediction 對齊靠 query `offset 5s`

prediction 與 ground truth 的視覺對齊不是靠改寫入時間戳，而是 Grafana query 固定用 `offset 5s`。

因此這條功能線的視覺語意，不只依賴 event / metrics，也依賴 dashboard query 策略。

## 5. Live 模式的 chart 行為

live mode 時，chart 的路徑是：

```text
log -> event -> metric handler -> /metrics -> Prometheus scrape -> Grafana iframe
```

前端停留在 live edge 時，iframe 會查：

```text
var-session=<live_session>
from=now-<window>m
to=now
refresh=5s
```

此時使用者看到的是：

- ground truth / prediction 持續往右延伸
- deviation 持續更新
- retrain annotations 在圖上即時出現

## 6. Live DVR 的 chart 行為

當 live 使用者按下 pause 或拖曳 timeline 時：

- topology / event log 主要來自前端 `_events`
- chart 仍然來自 Prometheus / Grafana

iframe 會改成：

```text
var-session=<live_session>
from=<absolute_from>
to=<absolute_to>
refresh=off
```

也就是查同一個 live session 的歷史樣本。

## 7. Replay 模式的 chart 行為

### paused / scrubbed

replay 啟動後，若該 session 已 backfill 或本來就存在於 Prometheus，iframe 會查：

```text
var-session=<replay_session>
from=<absolute_from>
to=<absolute_to>
refresh=off
```

這條路徑讓使用者在靜止觀察某個時間點時，直接看錄製當時的歷史曲線。

### playing

目前 replay `PLAYING` 時不再建立 pseudo session。

iframe 仍查原始 replay session，但把時間窗改成 historical relative：

```text
var-session=<replay_session>
from=now-(offset+window)
to=now-offset
refresh=5s
```

這讓 Grafana 可以像 live 一樣平滑刷新，但底層仍是同一份歷史 session。

## 8. 使用者真正看到的幾種差異

| 模式 | Grafana 查的是什麼 |
|---|---|
| live | 當前 live session 的 scraped exporter state |
| live paused / scrubbed | 同一 live session 的歷史絕對時間窗 |
| replay paused / scrubbed | 原始 replay session 的歷史絕對時間窗 |
| replay playing | 原始 replay session 的 historical relative 時間窗 |

因此現在 replay chart 的差異只在時間窗，不在 session identity。

## 9. 目前限制

- `model_swap` 在 live 與 replay 的底層機制不同：live 用 `remove()`，replay 用 `NaN` cut-off
- prediction 對齊目前固定寫死 `offset 5s`；若 slot 長度改變，dashboard query 也要一起調整
- 前端只控制 iframe URL，不理解 panel 內部 zoom、legend toggle 或 query inspector
- `skip` policy 下若 Prometheus 沒有該 session 的 metrics，chart 會是空的

若看到本文提到 pseudo-live、`MetricPlayer`、`/api/replay/play` 或 `.env` Grafana config，應視為舊版本設計，不是目前 runtime。
