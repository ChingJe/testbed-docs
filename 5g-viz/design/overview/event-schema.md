# 5g-viz 事件與 Metric Schema

本文整理目前 `5g-viz` 使用的核心事件型別、欄位，以及它們與 Prometheus metrics 的對應關係。

## 1. 基本原則

### parser 事件

由 `parser.py` 與 `rules/*.py` 產生的事件都至少包含：

```json
{
  "type": "<event_type>",
  "time": "<ISO 8601 timestamp>"
}
```

其餘欄位依 event type 決定。

### `state_snapshot`

`state_snapshot` 不是 parser 從 log 解析出的事件，而是後端根據目前 `state` 動態生成的快照。

它會出現在：

- live 模式新 WebSocket 連線建立時
- `GET /api/state`

它不會寫入 `events.jsonl`，也不會由 `/api/events` 回傳。

## 2. 事件型別總表

| type | 來源規則 | 主要欄位 | 主要用途 |
|---|---|---|---|
| `nf_up` | `rules/smf.py` | `nf` | 標記 NF 已上線 |
| `sbi_call` | `rules/smf.py`、`rules/nwdaf_sub.py` | `from`、`to`、`label`，部分情況帶 `supi`、`corr_id`、`sub_id`、`n` | 驅動拓樸邊動畫 |
| `smf_sub_confirmed` | `rules/nwdaf_sub.py` | `supi`、`corr_id` | 保留訂閱確認資訊，目前無視覺反應 |
| `upf_volume` | `rules/nwdaf.py` | `upf`、`ip`、`ul_bytes`、`dl_bytes` | 表示 UPF 週期性流量回報 |
| `ml_inference` | `rules/nwdaf.py` | `sub_id`、`target`、`ul_vol`、`dl_vol`、`confidence` | 驅動 AnLF 活動與預測 metrics |
| `accuracy` | `rules/nwdaf.py` | `model`、`deviation`、`accuracy`、`samples` | 驅動 accuracy 視覺效果與 deviation metric |
| `threshold_breach` | `rules/nwdaf.py` | `n`、`total` | 驅動 MTLF 警告視覺效果 |
| `retrain_trigger` | `rules/nwdaf.py` | `model`、`tid` | 驅動 retraining 狀態與 retrain counter |
| `retrain_done` | `rules/nwdaf.py` | 無額外欄位 | 表示 retraining 完成 |
| `model_swap` | `rules/nwdaf.py` | `model_id` | 表示新模型切換完成 |
| `aggregated_slot` | `rules/nwdaf.py` | `sub_id`、`target`、`ts`、`ul_vol`、`dl_vol`、`ul_thr`、`dl_thr` | ground truth metrics 的主要來源 |
| `adrf_stored` | `rules/nwdaf.py` | `trans_id`、`supi`、`count` | 驅動 ADRF store 視覺效果 |
| `adrf_retrieval_start` | `rules/nwdaf.py` | `model` | 驅動 ADRF retrieval start 視覺效果 |
| `adrf_retrieval_notify` | `rules/nwdaf.py` | 無額外欄位 | 驅動 ADRF notify 視覺效果 |
| `adrf_fetch` | `rules/nwdaf.py` | 無額外欄位 | 驅動 ADRF fetch 視覺效果 |
| `ip_supi_map` | `rules/nwdaf.py` | `ip`、`supi` | 保留 IP/SUPI enrichment 資訊，目前無視覺反應 |

## 3. 各事件欄位

### `nf_up`

```json
{
  "type": "nf_up",
  "time": "2026-03-12T08:46:25Z",
  "nf": "SMF"
}
```

用途：

- `state.py` 會把對應 node 標記為 `up`
- `event_reactions.nf_up` 會為該 node 加上 `up` class

### `sbi_call`

```json
{
  "type": "sbi_call",
  "time": "...",
  "from": "NWDAF",
  "to": "SMF",
  "label": "Nsmf_EventExposure_Subscribe"
}
```

欄位：

- `from`：事件起點 NF
- `to`：事件終點 NF
- `label`：邊的語意名稱，前端用它查 `edge_styles`

附加欄位視 rule 而定，可能包含：

- `supi`
- `corr_id`
- `sub_id`
- `n`

### `upf_volume`

```json
{
  "type": "upf_volume",
  "time": "...",
  "upf": "UPF-EES",
  "ip": "10.10.0.2",
  "ul_bytes": 123,
  "dl_bytes": 456
}
```

用途：

- 驅動 `UPF -> NWDAF` 的 notify 邊動畫
- 本身不直接寫入 Prometheus metrics

### `ml_inference`

```json
{
  "type": "ml_inference",
  "time": "...",
  "sub_id": "sub-001",
  "target": "group=group-test-001",
  "ul_vol": 1234,
  "dl_vol": 5678,
  "confidence": 80
}
```

用途：

- 前端讓 AnLF pulse
- metric handler 更新：
  - `nwdaf_predicted_ul_bytes`
  - `nwdaf_predicted_dl_bytes`

### `accuracy`

```json
{
  "type": "accuracy",
  "time": "...",
  "model": "model://...",
  "deviation": 0.1234,
  "accuracy": 88,
  "samples": 10
}
```

用途：

- 前端畫出 `AnLF -> MTLF` 的 accuracy 邊
- metric handler 更新 `nwdaf_deviation`

### `threshold_breach`

```json
{
  "type": "threshold_breach",
  "time": "...",
  "n": 3,
  "total": 5
}
```

用途：

- 前端在 MTLF 上顯示閾值違反的 self-edge 與橘色 pulse

### `retrain_trigger`

```json
{
  "type": "retrain_trigger",
  "time": "...",
  "model": "model://...",
  "tid": "task-123"
}
```

用途：

- 前端把 MTLF 加上 `retraining` class
- metric handler 遞增 `nwdaf_retrain_total`

### `retrain_done`

```json
{
  "type": "retrain_done",
  "time": "..."
}
```

用途：

- 前端移除 MTLF 的 `retraining` class
- 畫出 `MTLF -> AnLF` 的 provision 邊

### `model_swap`

```json
{
  "type": "model_swap",
  "time": "...",
  "model_id": "model-abc"
}
```

用途：

- 前端顯示 AnLF model deploy 動畫
- metric handler 會清掉目前 session 追蹤到的舊 `model` deviation series

### `aggregated_slot`

```json
{
  "type": "aggregated_slot",
  "time": "...",
  "sub_id": "sub-001",
  "target": "group=group-test-001",
  "ts": "2026-04-15T06:33:30Z",
  "ul_vol": 1234.0,
  "dl_vol": 5678.0,
  "ul_thr": 12.3,
  "dl_thr": 45.6
}
```

用途：

- 前端遇到第一筆 `aggregated_slot` 時會確保 Grafana iframe 建立
- metric handler 更新：
  - `nwdaf_ground_truth_ul_bytes`
  - `nwdaf_ground_truth_dl_bytes`

注意：

- parser 不保留 log 中所有欄位，只保留目前前端與 metrics 需要的子集

### ADRF 相關事件

| type | 欄位 | 用途 |
|---|---|---|
| `adrf_stored` | `trans_id`、`supi`、`count` | `NWDAF -> ADRF` store 動畫 |
| `adrf_retrieval_start` | `model` | `MTLF -> ADRF` retrieval subscribe 動畫 |
| `adrf_retrieval_notify` | 無額外欄位 | `ADRF -> MTLF` notify 動畫 |
| `adrf_fetch` | 無額外欄位 | `MTLF -> ADRF` retrieval request 動畫 |

### 輔助事件

| type | 欄位 | 目前用途 |
|---|---|---|
| `smf_sub_confirmed` | `supi`、`corr_id` | 保留 SMF 訂閱確認資訊 |
| `ip_supi_map` | `ip`、`supi` | 保留 IP / SUPI enrichment 資訊 |

這兩種事件目前沒有對應的 `event_reactions`，也沒有 metric handler。

## 4. `state_snapshot` 結構

`state.snapshot()` 產生的 payload 形式如下：

```json
{
  "type": "state_snapshot",
  "nf_status": {
    "smf": "up",
    "nwdaf_anlf": "unknown"
  },
  "node_classes": {
    "nwdaf_mtlf": ["retraining"],
    "smf": ["up"]
  }
}
```

欄位意義：

- `nf_status`：node 是否已被 `nf_up` 標記為上線
- `node_classes`：由 `event_reactions` 中的 `add_class` / `remove_class` 推導出的持久 class 集合

## 5. Prometheus metric 對應

目前 metrics 都由 `rules/nwdaf.py` 的 metric handlers 負責更新。

| Metric | Labels | 來源事件 | 值 |
|---|---|---|---|
| `nwdaf_ground_truth_ul_bytes` | `session`、`sub_id`、`target` | `aggregated_slot` | `ul_vol` |
| `nwdaf_ground_truth_dl_bytes` | `session`、`sub_id`、`target` | `aggregated_slot` | `dl_vol` |
| `nwdaf_predicted_ul_bytes` | `session`、`sub_id`、`target` | `ml_inference` | `ul_vol` |
| `nwdaf_predicted_dl_bytes` | `session`、`sub_id`、`target` | `ml_inference` | `dl_vol` |
| `nwdaf_deviation` | `session`、`model` | `accuracy` | `deviation` |
| `nwdaf_retrain_total` | `session` | `retrain_trigger` | counter 遞增 |

## 6. `session` label 的注入方式

`session` label 並不是每個事件都自帶的欄位，而是由執行環境補上：

- live 模式：`main.py` 在建立 live session 後，呼叫 `set_metric_session_id(session_id)`
- replay backfill：`main.py` 直接以 replay session ID 重建 series
- replay pseudo-live：`MetricPlayer` 會建立新的 `pseudo_session`

因此：

- 同一筆事件在 event log 中可能看不到 `session`
- 但進入 Prometheus 後一定會落在某個 `session` label 下

## 7. 視覺反應與事件 schema 的關係

事件 schema 本身不直接定義 Cytoscape 動畫；真正的視覺效果由 `topology.yaml` 的 `event_reactions` 決定。

換句話說：

- `rules/*.py` 決定「有哪些事件、欄位長什麼樣」
- `event_reactions` 決定「這些事件在前端如何被表現」
- `state.py` 又會重用其中的 `add_class` / `remove_class` 來維護 `state_snapshot`

因此 `event-schema` 與 [`../reference/topology-yaml.md`](../reference/topology-yaml.md) 是分工合作，而不是互相取代。
