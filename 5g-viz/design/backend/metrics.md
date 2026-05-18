# Metrics

本文描述目前 `5g-viz` 的 metrics 子系統：哪些 event 會投影成 Prometheus series、live 與 replay 各自怎麼寫入、Grafana 又如何查這些 series。

若要先看系統級心智，請讀：

- [`../overview/data-flow.md`](../overview/data-flow.md)
- [`api.md`](./api.md)
- [`../grafana/rendering.md`](../grafana/rendering.md)

## 1. 定位

目前 metrics 不是獨立收集器，而是 event pipeline 的第二層投影：

1. log 先被 parser 轉成 event
2. 只有部分 event type 會再投影成 Prometheus metrics

因此 metrics 永遠是事件流的衍生資料，不是與 event log 平行的原始來源。

## 2. 目前的 metric 集合

目前主要 metrics 仍定義在 `5g-viz/rules/nwdaf.py`：

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
| `nwdaf_session_anchor` | Gauge | `session` | session 建立或 replay backfill |

另有 accuracy policy / scope 類 metrics 也會經 replay backfill 邏輯重建，但主圖與常用 Grafana panel 仍主要圍繞上述集合。

## 3. `session` label 的角色

整個 metrics 子系統的主鍵是 `session` label。

它負責切分：

- 不同 live run
- 不同 replay session
- 同一 Prometheus TSDB 內的多批歷史資料

Grafana panel 本身不直接知道這是哪一次實驗，只知道要查哪個 `session`。

## 4. Live 路徑

### 1. event 進 handler

live 模式下，`backend.app` 在每筆 event 進主流程時會呼叫 metric handlers。

路徑是：

```text
event
  -> rules.ALL_METRIC_HANDLERS
  -> prometheus_client exporter state
  -> /metrics
  -> Prometheus scrape
```

### 2. session 初始化

live mode 建立新 session 時，backend 會：

- 設定 metric session id
- 初始化 `nwdaf_session_anchor`
- 初始化 retrain pulse gauges

這保證：

- 後續 handler 即使 event payload 沒自帶 `session`，也能寫到正確 series
- Grafana session variable 很快就能看到這次 live session

### 3. 各 handler 的主要語意

- `aggregated_slot`
  - 更新 ground truth UL / DL gauges
- `ml_inference`
  - 更新 predicted UL / DL gauges
- `accuracy`
  - 更新 `nwdaf_deviation{session,model}`
- `retrain_trigger`
  - `nwdaf_retrain_total` 自增
  - `nwdaf_retrain_start_event` 先設 `1`，之後 reset 回 `0`
- `retrain_done`
  - `nwdaf_retrain_done_event` 先設 `1`，之後 reset 回 `0`
- `model_swap`
  - live 路徑會移除當前 session 已追蹤的舊 deviation series

## 5. Replay backfill 路徑

### 1. 目的

replay backfill 的目的，是把 session 內的 metric 事件以「原始事件時間戳」重新寫回 Prometheus，讓 Grafana 能直接查歷史曲線。

它不是播放時的第二套 runtime，也不再建立 pseudo session。

### 2. 寫入方式

目前 replay backfill 由 `backend.app` 直接：

1. 從 `_events` 挑出 metric 相關事件
2. 在記憶體中重建各條 series
3. 編碼為 Prometheus remote write payload
4. `snappy` 壓縮
5. POST 到本機 Prometheus `/api/v1/write`

### 3. `--backfill` policy

這條路徑的啟動策略由 `run.py replay ... --backfill=...` 決定：

- `auto`
  - Prometheus 已有該 session 就 reuse
  - 否則 backfill
- `overwrite`
  - 先刪該 session series，再 backfill
- `skip`
  - 完全不寫 Prometheus；此時 replay chart 可能為空

這個 policy 決策由 `services/replay_session_service.py` 與 `services/prometheus_service.py` 協作完成。

### 4. `model_swap` 的 replay 語意

replay backfill 不能重播 `prometheus_client` 的 `remove()` API，所以 `model_swap` 在 replay 下會用 `NaN` cut-off sample 表達「舊 model series 在這裡結束」。

因此：

- live：真實移除舊 deviation series
- replay：在時間軸上寫一個截斷點

兩者畫面效果接近，但底層機制不同。

## 6. Grafana 如何使用這些 metrics

Grafana dashboard 主要透過：

- `session` label 選定哪一輪資料
- `target="group=<group_id>"` 切 traffic panel
- `offset 5s` 對齊 prediction 與 ground truth

重要的是：

- replay `PAUSED` / `SCRUBBED`：查原始 session 的絕對時間窗
- replay `PLAYING`：仍查原始 session，只是改成 historical relative `from/to`

因此目前 chart 行為已不需要 pseudo-live remap。

## 7. 與長駐 Prometheus 的關係

目前 Prometheus 已是長駐服務，`run.py` 不再把它當 disposable cache。

這代表：

- live scrape 與 replay backfill 共同寫入同一個持久 TSDB
- 是否重灌某個 replay session，變成 session-scoped decision
- session status 可以透過 CLI 檢查與刪除，不需全域 wipe

## 8. 目前的限制與契約

- 只有定義了 metric handlers 的 event type 會進 Prometheus
- replay backfill 依賴 remote write receiver 與 `python-snappy`
- `session` label 是整個圖表功能的主鍵；若自訂 handler 未正確帶入 `session`，Grafana 就無法切 session
- `target` 必須符合 dashboard 期望的 `group=<group_id>` 語意
- 若未來新增新的 metric-bearing event type，必須同步考慮：
  - live handler
  - replay backfill builder
  - dashboard query 是否需要更新

若看到本文仍提到 pseudo-live、`MetricPlayer`、`/api/replay/play` 或 replay speed，應視為歷史版本，不是目前 runtime。
