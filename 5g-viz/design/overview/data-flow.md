# 5g-viz 資料流

> Historical note: this document still uses the older module names (`main.py`, root-level services) and replay model. The current runtime keeps the same broad live/replay split, but the module boundaries and replay chart path have changed.

本文描述 `5g-viz` 目前兩條主要資料路徑：

- live：從遠端 VM log 即時進入瀏覽器與 Grafana
- replay：從磁碟上的 session 還原事件與 metrics

## 1. Live 路徑總覽

```text
5GC VM log
  -> collector.py (SSH + tail -F)
  -> asyncio.Queue
  -> parser.py + rules/*
  -> event dict
  -> main.py
     -> _events / events.jsonl
     -> state.apply_event()
     -> metric handlers -> /metrics
     -> WebSocket /ws
  -> frontend/events.js + topology.js

/metrics
  -> Prometheus scrape
  -> Grafana dashboard
  -> frontend iframe
```

## 2. Live：從 log 到 event

### Step 1. collector 連到遠端 VM

`collector.start()` 會根據 `topology.yaml` 的 `ssh_sources` 設定，同時對每個 source 啟動 `_connect_and_tail()`。

對於每個 log source：

- 若設了 `latest_subdir: true`，先用 `ls -t <dir>/ | head -1` 找出最新子目錄
- 再對目標檔案執行 `tail -F`
- 每讀到一行，就把資料推進共用 `asyncio.Queue`

推入 queue 的資料格式是：

```json
{
  "source": "free5gc" | "nwdaf",
  "line": "<原始 log 行>"
}
```

### Step 2. queue consumer 做 parser 與 side effects

`main.py` 的 `_process_queue()` 是 live 模式下的單一 consumer loop：

1. 從 queue 取出一行 log
2. 呼叫 `parser.parse_line(line, source=...)`
3. 若 parser 回傳事件：
   - append 到記憶體中的 `_events`
   - 追加寫入 `events.jsonl`
   - 呼叫 `_update_metrics(event)`
   - 呼叫 `_broadcast(event)`

### Step 3. parser 依 `rules/` 產生事件

`parser.py` 先用固定 regex 解析 logrus 風格欄位：

- `time`
- `level`
- `msg`
- `CAT`
- `NF`
- 其他 key/value

之後依 `rules.ALL_RULES` 順序比對：

- `nf`
- `cat`
- `source`
- `msg` regex

第一條匹配成功的 rule 會產生 event dict。這些事件會成為整個系統的共同資料模型。

現況上，`rules/__init__.py` 是用 `pkgutil.iter_modules()` 掃描 `rules/` 目錄；以目前 repo 檔名與執行環境，模組載入順序是：

1. `nwdaf.py`
2. `nwdaf_sub.py`
3. `smf.py`

再加上 parser 採 first-match-wins，因此 rule 的先後順序會直接影響哪一條規則先匹配到某筆 log。

一個簡化後的 `aggregated_slot` 範例如下：

```text
raw log
  -> BASE regex 取出 time / level / msg / CAT / NF
  -> match `rules/nwdaf.py` 的 aggregated_slot 規則
  -> event = {type, time, sub_id, target, ts, ul_vol, dl_vol, ul_thr, dl_thr}
```

## 3. Live：從 event 到前端

### Step 4. 後端更新 state snapshot

`_broadcast(event)` 在送出 WebSocket 前，會先呼叫 `state.apply_event(event)`。

`state.py` 目前會做兩類事情：

- 若事件是 `nf_up`，更新 `nf_status`
- 若事件對應的 `event_reactions` 包含 `add_class` / `remove_class`，更新 `node_classes`

這讓後端能維持一份可重建目前拓樸狀態的 `state_snapshot`。

### Step 5. WebSocket 推送事件

在 live 模式下，所有 parser 產生的事件都會經過 `/ws` 推給前端。

新 WebSocket 連線建立時，後端會先送一份：

```json
{
  "type": "state_snapshot",
  "nf_status": { ... },
  "node_classes": { ... }
}
```

之後才開始持續送 parser 事件。

### Step 6. 前端套用事件

`events.js` 在 live 模式下會：

1. 建立到 `/ws` 的 WebSocket 連線
2. 收到 `state_snapshot` 時呼叫 `Topology.applySnapshot(...)`
3. 收到一般事件時：
   - 加入本地 `_events`
   - 更新 timeline 邊界
   - 在 `DVR.LIVE` 狀態下即時 `dispatch(event)`

`dispatch(event)` 會：

- 呼叫 `Topology.react(event)` 更新 Cytoscape 視覺效果
- 將事件 append 到 event log
- 若事件是 `aggregated_slot`，確保 Grafana iframe 已建立

## 4. Live：從 event 到 metrics

### Step 7. metric handlers 更新 `/metrics`

`_update_metrics(event)` 會呼叫 `rules.ALL_METRIC_HANDLERS` 中對應 event type 的 handler。

目前會寫入 Prometheus metrics 的事件包括：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- `retrain_trigger`
- `model_swap`

這些 handler 定義在 `rules/nwdaf.py`。

對照如下：

| Event Type | 寫入 Metrics | Labels |
|---|---|---|
| `aggregated_slot` | `nwdaf_ground_truth_ul_bytes`、`nwdaf_ground_truth_dl_bytes` | `session`、`sub_id`、`target` |
| `ml_inference` | `nwdaf_predicted_ul_bytes`、`nwdaf_predicted_dl_bytes` | `session`、`sub_id`、`target` |
| `accuracy` | `nwdaf_deviation` | `session`、`model` |
| `retrain_trigger` | `nwdaf_retrain_total` | `session` |
| `model_swap` | live 路徑會刪除舊 deviation series | `session`、`model` |
| 其他事件 | 不進 Prometheus，只保留在 event log / topology | — |

### Step 8. Prometheus scrape，Grafana 查詢

- `/metrics` 由 `prometheus_client.generate_latest()` 產生
- Prometheus 以固定 scrape 週期抓取 `/metrics`
- Grafana dashboard 查詢這些 metrics，並用 `session` variable 區分不同 session

前端不直接讀 Prometheus；而是嵌入 Grafana iframe。

## 5. Replay 路徑總覽

```text
sessions/<session_id>/
  -> meta.json + events.jsonl + topology.yaml
  -> main.py _init_replay_session()
  -> _events + state rebuild
  -> replay backfill (remote write)
  -> frontend /api/events + /api/session-info
  -> DVR timeline / paused view

play
  -> /api/replay/play
  -> MetricPlayer
  -> pseudo_session remote write
  -> Grafana now-window
```

## 6. Replay：啟動階段

### Step 1. 載入 session 檔案

replay 模式下，`main.py` 會：

1. 驗證 `SESSION_PATH`
2. 載入 `meta.json`
3. 載入 `events.jsonl`
4. 載入 session 目錄中的 `topology.yaml`

接著：

- 把事件放進記憶體 `_events`
- 以事件重播的方式重建 `state`
- 設定目前 session ID，供 replay backfill 使用

### Step 2. replay backfill 到 Prometheus

lifespan 啟動時會呼叫 `_run_replay_backfill()`：

1. 從 `_events` 篩出 metric 相關事件
2. 依事件時間重建 Prometheus time series
3. 編碼成 remote write protobuf payload
4. 用 `snappy` 壓縮 payload
5. 寫入本機 Prometheus `/api/v1/write`

這一步讓 replay 模式即使不播放，也能直接查詢「原始 session 時間軸」上的圖表。

若 `FORCE_BACKFILL` 未開，backend 在真正寫入前還會先 query：

```promql
count(nwdaf_ground_truth_ul_bytes{session="<session_id>"})
```

只有在查不到既有樣本時才繼續 backfill；`FORCE_BACKFILL=1` 則會略過這個 pre-check。

## 7. Replay：前端資料來源

replay 模式下前端不建立 WebSocket，而是：

1. 先抓 `/api/session-info`
2. 再用 `/api/events?session=<id>` 載入整份事件
3. 以本地 `_events` 建立 timeline
4. 用 `Topology.renderStaticSnapshot(...)` 或 `Topology.applySnapshot(...)` 重建指定時間點的畫面

此時 Grafana iframe 預設查的是 replay session 的 backfill 資料。

## 8. Replay：pseudo-live 播放

### Step 1. 使用者按下播放

前端會呼叫：

```text
POST /api/replay/play
```

參數包含：

- `from_time`
- `speed`
- `window`

### Step 2. `MetricPlayer` 建立 pseudo session

`MetricPlayer.play()` 會：

1. 先停止既有的 active pseudo stream
2. 建立新的 `pseudo_session`
3. 依目前 playhead 位置，先做一段 pre-seed
4. 啟動 `_emit_loop()`，把後續 metric event 依播放速度映射到當前 wall clock

### Step 3. Grafana 切到 now-window

當前端進入 replay `PLAYING` 狀態，且有 active `pseudo_session` 時：

- Grafana iframe 會切到 `now-<window>m ~ now`
- query 的 `var-session` 改成 `pseudo_session`

因此畫面上看起來像是「歷史資料正在即時播放」。

## 9. Live 與 Replay 的分界

兩種模式在資料流上有幾個重要差異：

| 面向 | live | replay |
|---|---|---|
| 事件來源 | 遠端 log 即時解析 | 磁碟上的 `events.jsonl` |
| 拓樸更新 | WebSocket push | 前端本地用事件重建 |
| 圖表基礎資料 | scrape `/metrics` | 啟動時 replay backfill |
| 播放中的圖表 | 不需要 pseudo-live | 由 `MetricPlayer` remote write 產生 `pseudo_session` |
| session 設定 | 啟動時新建 | 讀取既有 session |

## 10. 資料流上的關鍵分界

- `event_reactions` 同時影響前端動畫與後端 `state_snapshot`
- `session` label 不一定存在於事件 payload 中，而是由 metric handlers / replay writer 注入到 Prometheus metrics
- replay 模式的圖表至少有兩條路徑：
  - 原始 replay session backfill
  - pseudo-live `pseudo_session`

這些分界直接決定同一批事件在 live 與 replay 模式下，如何被呈現為拓樸狀態與圖表資料。
