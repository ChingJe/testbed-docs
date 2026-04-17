# Metrics

本文描述 `5g-viz` 目前的 metrics 生成方式，以及 live、replay、pseudo-live 三條資料路徑之間的差異。

## 1. 定位

`5g-viz` 的 metrics 子系統不是獨立收集器，而是建立在 event pipeline 上的第二層投影：

1. parser 先從 log 產生 event
2. 只有特定 event type 會再被轉成 Prometheus metrics

因此 metrics 永遠是事件流的衍生資料，而不是與 event log 平行的原始來源。

## 2. 目前的 metric 集合

目前系統定義了九條 metrics，全部在 `rules/nwdaf.py`：

| Metric | 型別 | Labels | 來源事件 |
|---|---|---|---|
| `nwdaf_ground_truth_ul_bytes` | Gauge | `session`、`sub_id`、`target` | `aggregated_slot` |
| `nwdaf_ground_truth_dl_bytes` | Gauge | `session`、`sub_id`、`target` | `aggregated_slot` |
| `nwdaf_predicted_ul_bytes` | Gauge | `session`、`sub_id`、`target` | `ml_inference` |
| `nwdaf_predicted_dl_bytes` | Gauge | `session`、`sub_id`、`target` | `ml_inference` |
| `nwdaf_deviation` | Gauge | `session`、`model` | `accuracy` |
| `nwdaf_retrain_total` | Counter | `session` | `retrain_trigger` |
| `nwdaf_retrain_start_event` | Gauge | `session` | `retrain_trigger` |
| `nwdaf_retrain_done_event` | Gauge | `session` | `retrain_done` |
| `nwdaf_session_anchor` | Gauge | `session` | session 建立時 |

這些 series 目前都以 `session` label 作為主要切分維度，Grafana 也是靠這個 label 切換不同錄製或 pseudo-live session。

## 3. Live 路徑

live 模式下，`main.py` 在處理每筆 event 時會呼叫：

```python
_update_metrics(event)
```

`_update_metrics()` 會依 event type 對應到 `ALL_METRIC_HANDLERS`，而這份 registry 來自 `rules/` 模組的動態匯總。

### session label 的注入

live mode 建立新 session 時，`main.py` 會呼叫：

```python
set_metric_session_id(session_id)
```

這裡的 `set_metric_session_id()` 是 `rules/__init__.py` 匯總後對外暴露的封裝函式。各 rule 模組若有實作內部的 `set_session_id()`，就會被 registry 收進 `_SESSION_SETTERS`，由這個封裝入口一起呼叫。

目前真正保存值的是 `rules/nwdaf.py` 內部的 `set_session_id()`；它會把 session ID 寫進 `_SESSION_ID`。後續 metric handler 會用：

```python
event.get("session") or _SESSION_ID
```

決定實際寫入 Prometheus 的 `session` label。

也就是說：

- event payload 不一定要自帶 `session`
- 只要 live session 已初始化，metric handler 仍能把樣本寫到正確的 session series

### `nwdaf_session_anchor` 的初始化

live mode 建立新 session 時，`main.py` 除了呼叫 `set_metric_session_id()` 之外，還會呼叫：

```python
ensure_metric_session(session_id)
```

這對應 `rules/__init__.py` 中的 `ensure_metric_session()`，它會呼叫每個 rule 模組的
`ensure_session_metrics()`。`rules/nwdaf.py` 的實作會：

1. 把 `nwdaf_session_anchor{session}` 設為 `1`
2. 把 `nwdaf_retrain_start_event` 與 `nwdaf_retrain_done_event` 初始化為 `0`

此外，`main.py` 也會同時以 remote write 寫入一筆 `nwdaf_session_anchor` 樣本，讓 session 在
下一次 Prometheus scrape 之前就出現在 Grafana session 下拉選單。

## 4. 各 handler 的實際行為

### `aggregated_slot`

`_on_aggregated_slot()` 會更新兩條 gauge：

- `nwdaf_ground_truth_ul_bytes`
- `nwdaf_ground_truth_dl_bytes`

labels 來自：

- `session`
- `sub_id`
- `target`

### `ml_inference`

`_on_ml_inference()` 會更新：

- `nwdaf_predicted_ul_bytes`
- `nwdaf_predicted_dl_bytes`

labels 與 `aggregated_slot` 相同。

### `accuracy`

`_on_accuracy()` 會：

1. 把 `model` 記進 `_known_models[session]`
2. 更新 `nwdaf_deviation{session, model}`

### `retrain_trigger`

`_on_retrain_trigger()` 會：

1. 對 `nwdaf_retrain_total{session}` 做一次 `inc()`
2. 將 `nwdaf_retrain_start_event{session}` 設為 `1`

之後由 `main.py` 的 asyncio task 在一個 scrape interval 之後把 pulse reset 回 `0`。

### `retrain_done`

`_on_retrain_done()` 會把 `nwdaf_retrain_done_event{session}` 設為 `1`，同樣由 `main.py`
非同步 reset 回 `0`。

### `model_swap`

`_on_model_swap()` 會把該 session 已追蹤的所有 deviation series 移除：

```python
_deviation.remove(session, model)
```

然後把 `_known_models[session]` 清空。

這代表 model hot-swap 後，Grafana 查到的 deviation series 會以新模型重建，而不是無限制保留舊 model label。

live 路徑用 exporter `remove()` 表達「舊 model 結束」。replay backfill 與 pseudo-live 無法透過
remote write 重播這個 API，因此改採等價語意：在 `model_swap` 時為所有 active models 寫一筆
`NaN` cut-off sample，讓 Grafana 線段在部署當下截斷。

## 5. Replay backfill

replay mode 啟動時，`main.py` 不靠 `prometheus_client` 逐筆重放 event，而是：

1. 先從 `_events` 挑出 metric 相關事件
2. 在 `_build_replay_metric_series()` 中重建每條 time series
3. 手動編碼成 Prometheus remote write protobuf
4. 壓成 snappy payload
5. POST 到本機 Prometheus `/api/v1/write`

### 為什麼要走 remote write

因為 replay 需要保留「原始事件時間戳」，讓 Grafana 能直接查到歷史 session 當時的圖表。若只更新當前程序的 gauge / counter，時間軸會全部落在現在。

### backfill 的 series 組裝方式

`_build_replay_metric_series()` 會把樣本暫存在：

```python
dict[(metric, sorted_labels)] -> dict[ts_ms] -> value
```

效果是：

- 同一個 metric + labels + timestamp 若重複出現，後寫的值會覆蓋前值
- 實際送出前會依 metric、labels、timestamp 排序

此外，backfill 時會在 session 第一筆 metric event 的時間戳額外寫入：

```text
nwdaf_session_anchor{session="<session_id>"} = 1.0
```

確保 replay session 出現在 Grafana session 下拉選單。

### backfill 何時會跳過

`_run_replay_backfill()` 預設會先 query：

```promql
count(nwdaf_ground_truth_ul_bytes{session="<session_id>"})
```

如果 Prometheus 內已經查得到該 session 的資料，就跳過 backfill。

只有在設定 `FORCE_BACKFILL=1` 時，才會強制重寫。

### 相依條件

remote write 需要 `python-snappy`。若匯入失敗，replay backfill 會直接失敗，`main.py` 會記 warning，並讓 Grafana 圖表對該 session 不可用。

## 6. Pseudo-live 路徑

使用者在 replay 模式按下播放時，真正運作的是 `MetricPlayer`。

`MetricPlayer` 會先從整份 `_events` 中挑出 `METRIC_EVENT_TYPES`：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `retrain_done`
- `model_swap`

這些事件都會經過同一個 `_build_metric_series_for_event()`，因此 pre-seed 與 emit loop 共用同一套
metric 建模。

然後建立一條新的 pseudo session，例如：

```text
_live_<原始session>__20260415T061530123456Z
```

## 7. Pseudo-live 的兩個階段

### pre-seed

`play(from_time, speed, window_min)` 會先做一段 pre-seed：

- 把 `from_time` 前一個 window 內的 metric event 映射到「現在之前」的一段時間
- 先 remote write 這批資料

這讓 Grafana 在切到 `now-<window> ~ now` 時，畫面不會一開始就是空窗。

### emit loop

接著 `_emit_loop()` 會：

1. 依事件原始時間差與播放速度計算 sleep 時間
2. 把同一個原始 timestamp 的事件合併成同一批次
3. 用目前 wall clock 當作新樣本時間戳
4. 持續 remote write 到 Prometheus

因此 pseudo-live 並不是把舊資料原封不動重送，而是把它重新映射到「現在」。

## 8. Retrain metrics 在 pseudo-live 中的處理

`MetricPlayer` 有三個特別處理：

1. 每筆 metric event 都會額外帶上一筆：

```text
nwdaf_session_anchor{session="<pseudo_session>"}
```

確保 pseudo session 從第一筆資料開始就出現在 Grafana session 下拉選單。

2. 即使目前事件不是 `retrain_trigger`，它仍會在輸出的 series 中帶上一筆：

```text
nwdaf_retrain_total{session="<pseudo_session>"}
```

這樣可以確保 counter 在 pseudo-live 播放過程中始終有可查詢的當前值，而不是只在 retrain 事件發生那一刻才出現樣本。

3. 當事件為 `retrain_trigger` 或 `retrain_done` 時，會額外寫：

```text
nwdaf_retrain_start_event{session="<pseudo_session>"}
nwdaf_retrain_done_event{session="<pseudo_session>"}
```

各自的 `(ts_ms, 1.0)` 與 `(ts_ms + 5000, 0.0)` pulse sample pair。這讓 pseudo-live 與 backfill
都能被同一組 Grafana annotation query 命中。

## 9. 播放中斷與變速

### pause

`pause(pseudo_session)` 會停止目前 active emit loop。若 pseudo session 不匹配，則不做事。

### update_speed

`update_speed(pseudo_session, speed, current_time)` 會：

1. 根據新的 playhead 找到對應 cursor
2. 用 prefix sum 重新計算截至該點的 retrain_total
3. 取消舊 task
4. 以新速度建立新的 emit loop

因此變速不是在原 loop 內即時調整，而是重建一條新的 loop。

## 10. 目前限制

- 目前只有 `rules/nwdaf.py` 定義 metric handlers；SMF 與 subscription 類事件不會寫入 Prometheus
- replay backfill 與 pseudo-live 都依賴 Prometheus remote write receiver 已啟用
- replay backfill 與 pseudo-live 雖然不能重播 live exporter 的 `remove()`，但已用 `NaN` cut-off
  樣本補上 `model_swap` 的線段終止語意
- 若未來 rule 增加新的 metric handler，backfill 與 `MetricPlayer` 也需要同步擴充，否則 live 與 replay 的圖表語意會再度分岔
- `/metrics` 與 remote write 會同時作用在同一個 Prometheus，但資料時間軸不同；理解圖表時必須分清楚 live series、原始 replay session 與 pseudo session
