# Rendering

> Historical note: this document still uses the old live / replay / pseudo-live rendering model. The current system no longer creates pseudo-live replay sessions; replay playback now queries the original session with a historical relative time range.

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

這不是把資料寫入時的 timestamp 改掉，而是要求 Prometheus 在查 `T` 這個顯示時間點時，實際取用 `T-5s` 的 prediction 樣本。

目前比較精確的理解是：

- ground truth 樣本記錄在它自己的事件時間上
- prediction 樣本較晚產生，但 UI 想把它和對應 slot 的 ground truth 放在同一個顯示位置比較
- 因此 query 層固定用 `offset 5s` 往前取 5 秒前的 prediction 值，讓兩條線在時間軸上視覺對齊

可以把它想成：

```text
顯示時間 T      -> ground truth 查 T 的值
顯示時間 T      -> prediction 實際查 T-5s 的值
```

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
query_result(count by (session) (count_over_time(nwdaf_session_anchor[365d])))
```

並用 regex 從 query result 抽出 `session="..."` label。

也就是說，Grafana 可切換的 session 候選集合目前來自 retention 視窗內實際還有樣本的：

```text
nwdaf_session_anchor
```

這帶來兩個直接效果：

- pseudo session 若已被 `delete_series` 清掉，就不會只因為舊 label index 殘留而繼續出現在下拉選單
- pseudo-live 在 Play 開始時只要先寫入 `nwdaf_session_anchor` 錨點樣本，就能立刻出現在 session 下拉選單

前端 iframe 則是直接用 `var-session=<session_id>` 指定當前要看的 session。

## 6. Retrain Annotation

dashboard 目前建立兩條 annotations query：

```promql
(nwdaf_retrain_start_event{session="$session"} > 0) unless on(session) (nwdaf_retrain_start_event{session="$session"} offset 5s > 0)
(nwdaf_retrain_done_event{session="$session"} > 0) unless on(session) (nwdaf_retrain_done_event{session="$session"} offset 5s > 0)
```

效果是：

- `retrain_trigger` 時打紅色 `Retrain Start`
- `retrain_done` 時打綠色 `Retrain Done`
- annotations datasource 與 panel 相同，都是同一個 Prometheus datasource

這樣做的原因是：

- retrain start / done 是事件，不再需要從 counter 變化間接推導
- live、replay backfill 與 pseudo-live 都能用相同語意建模
- backfill / pseudo-live 在 `ts_ms` 寫 `1.0`、在 `ts_ms + 5000` 寫 `0.0`，每個事件只有一個上升沿，天然不需去重
- live mode 中，pulse 若跨越多個 Prometheus scrape，`unless on(session) (...) offset 5s > 0` 確保只保留連續高電位的第一個 5s step，Grafana 不會渲染重複的 annotation 線

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

目前仍有幾個 live 與 replay 並不完全對稱的細節。

### `model_swap` 的終止語意用不同機制實現

live mode 的 `model_swap` 會刪除既有 deviation series；replay backfill 與 pseudo-live 則改寫一筆
`NaN` sample，強制舊 model 線段在 swap 時截斷。

因此三條路徑在畫面上都會保留新模型部署後的 cold-start gap，只是底層實作不是同一種 API。

### Session variable 以 `nwdaf_session_anchor` 作為索引錨點

session variable 查的是 `nwdaf_session_anchor` 的 `count_over_time(...)` 結果（而非
`nwdaf_retrain_total`）。使用獨立錨點 metric，可讓 session 在建立後立刻出現在下拉選單，
無需等待第一筆 retrain 事件。這能避開 Prometheus label index 對已刪除 `_live_...` session
的殘留問題，但也代表若某條未來資料路徑完全不寫 `nwdaf_session_anchor`，Grafana 端仍可能
看不到那個 session。

### `start.sh` 每次啟動都清除所有 `nwdaf_*` series

這是 live 與 replay 共同的啟動行為：`start.sh` 透過 Prometheus admin API 刪除所有
`nwdaf_.*` series，確保前一次 session 的殘留資料不會汙染新圖表。這也表示 Grafana 上可查的
session 範圍，完全取決於當次 `start.sh` 啟動後寫入 TSDB 的內容，而不是一個長期累積的
session 資料倉。

## 9. 呈現參數

目前 dashboard 還有幾個固定 rendering 參數：

- dashboard refresh：`5s`
- panel query interval：`5s`
- datasource `timeInterval`：`5s`
- panel `maxDataPoints`：`4096`
- legend 顯示在 panel 底部

這些值都是寫在 `grafana_setup.py` 的常數，調整後會在下一次 app 啟動時覆蓋進 dashboard。

## 10. 目前限制

- dashboard variable 目前以 `nwdaf_session_anchor` 作為 session 錨點；若未來某條資料路徑完全不寫這個 metric，Grafana 端仍可能看不到那個 session
- prediction `offset 5s` 是固定常數；若 slot 定義或資料粒度改變，圖上對齊方式可能失真
- live 與 replay 對 `model_swap` 的底層實作不同：前者用 `remove()`，後者用 `NaN` cut-off
- query 與 panel 結構全由 Python 程式生成；要在 Grafana UI 做細部客製化時，可讀性不如直接維護 dashboard JSON
