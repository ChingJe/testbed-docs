# Rendering

本文描述 `5g-viz` 目前的 Grafana dashboard 如何查 Prometheus、切換 session，並呈現 live、replay 與 pseudo-live 三種圖表語意。

## 1. Dashboard 結構

目前 dashboard 由兩類 panel 組成：

- 每個 `GRAFANA_GROUPS` 對應一個 traffic panel
- 最後一個 panel 是 `Model Deviation`

panel 數量與排列完全在 `grafana_setup.py` 內動態生成，不是從預存 JSON 匯入。

## 2. Traffic Panel 查哪些 series

每個 traffic panel 都固定查四條 PromQL：

- `nwdaf_ground_truth_ul_bytes`
- `nwdaf_ground_truth_dl_bytes`
- `nwdaf_predicted_ul_bytes`
- `nwdaf_predicted_dl_bytes`

查詢條件同時綁兩個維度：

- `session="$session"`
- `target="group=<group_id>"`

並以 `sum by (target)(...)` 彙總。

### `group=<group_id>` 的意義

這裡不是 Grafana 的 panel group，而是 metric `target` label 的值。也就是說，後端事件中寫入的 `target` 必須剛好等於：

```text
group=<某個 GRAFANA_GROUPS 值>
```

否則對應 panel 就查不到資料。

## 3. 為什麼 prediction query 要 `offset 5s`

prediction 兩條 query 目前都固定加上：

```promql
offset 5s
```

目的是把預測曲線整體向左對齊一個 slot，讓它在視覺上和 ground truth 比較接近。

這是 dashboard 定義本身的呈現策略，不是資料寫入時真的改了 timestamp。

## 4. Deviation Panel 的查詢語意

deviation panel 查的是：

```promql
nwdaf_deviation{session="$session"}
and on(session, model)
topk by (session) (1, timestamp(nwdaf_deviation{session="$session"}))
```

這個查法的效果是：

- 仍以 `session` 和 `model` 作為 series 維度
- 但只保留「目前時間上最新的一批 deviation series」

這和 live mode 裡 `model_swap` 會刪除舊 deviation series 的行為互相呼應，用來避免圖上無限制堆積舊 model label。

## 5. `session` Variable 是整個 dashboard 的主鍵

dashboard 內建一個 template variable：

```promql
label_values(nwdaf_ground_truth_ul_bytes, session)
```

也就是說，Grafana 可切換的 session 候選集合目前完全來自：

```text
nwdaf_ground_truth_ul_bytes
```

這帶來兩個直接效果：

- 若某個 session 沒有任何 ground-truth traffic metric，它就不會出現在 session 下拉選單
- deviation-only 或 retrain-only 的 session 不會單靠其他 metrics 出現在 dashboard variable 中

前端 iframe 則是直接用 `var-session=<session_id>` 指定當前要看的 session。

## 6. Retrain Annotation

dashboard 另外建立一條 annotations query：

```promql
idelta(nwdaf_retrain_total{session="$session"}[15s]) > 0
```

效果是：

- 當 `nwdaf_retrain_total` 在最近 15 秒內有增加，就在圖上打一個 `Retrain` 標記
- annotations datasource 與 panel 相同，都是同一個 Prometheus datasource

這條 annotation 在 live 與 pseudo-live 下都能工作，因為兩者都會持續產生新的 retrain counter 樣本。

## 7. Live、Replay 與 Pseudo-Live 的圖表差異

Grafana 查詢看起來都只是 Prometheus query，但實際上會落到三種不同資料語意。

### Live

live mode 圖表來自兩條條件同時成立：

- Prometheus 持續 scrape `/metrics`
- iframe 用 `from=now-<window>m&to=now&refresh=5s`

這時圖表反映的是 app 內目前的 gauges / counters 狀態。

### Replay Backfill

replay 啟動時，backend 會把原始 session 的 metric event 以原始 timestamp remote write 進 Prometheus。

之後 iframe 若帶：

- `var-session=<orig_session>`
- 絕對 `from/to`

查到的就是原始錄製 session 的歷史圖。

### Replay Pseudo-Live

replay 播放期間，前端會把 `var-session` 切到一個 `_live_<orig>__<token>` pseudo session，並把查詢時間窗切回 `now-<window> ~ now`。

這時 Grafana 查到的不是原始 replay session，而是 `MetricPlayer` 重新映射到現在的另一批樣本。

## 8. 圖表與資料不完全對稱的地方

目前有幾個 live 與 replay 並不完全對稱的細節。

### `model_swap` 清除只在 live handler 生效

live mode 的 `model_swap` 會刪除既有 deviation series，但 replay backfill 與 pseudo-live 都沒有真正重播刪除動作。

因此 replay session 的 deviation panel 可能會保留比 live 更多的舊 model 痕跡；目前主要靠 dashboard query 的「只取最新 timestamp」策略降低這個差異。

### Session variable 依賴 ground truth metric

若未來某條資料路徑只寫 prediction 或 deviation，不寫 `nwdaf_ground_truth_ul_bytes`，Grafana 端可能完全看不到這個 session。

### Prometheus TSDB 在 replay 啟動時會被清空

這使得 replay 圖表相對乾淨，但也表示 Grafana 上同時可查的 session 範圍取決於當前這次 `start.sh` 啟動之後寫進 TSDB 的內容，而不是一個長期累積的資料倉。

## 9. 呈現參數

目前 dashboard 還有幾個固定 rendering 參數：

- dashboard refresh：`5s`
- panel query interval：`5s`
- datasource `timeInterval`：`5s`
- panel `maxDataPoints`：`4096`
- legend 顯示在 panel 底部

這些值都是寫在 `grafana_setup.py` 的常數，調整後會在下一次 app 啟動時覆蓋進 dashboard。

## 10. 目前限制

- dashboard variable 候選只看 ground truth UL metric，對其他 metric 類型不夠健壯
- prediction `offset 5s` 是固定常數；若 slot 定義或資料粒度改變，圖上對齊方式可能失真
- deviation panel 目前用 query 技巧壓低舊 model 殘留，而不是從 replay 寫入路徑完整重播 series delete
- query 與 panel 結構全由 Python 程式生成；要在 Grafana UI 做細部客製化時，可讀性不如直接維護 dashboard JSON
