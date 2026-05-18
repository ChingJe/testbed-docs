# Replay

本文描述目前 `5g-viz` 的 replay runtime：如何載入本地 session、如何決定是否 backfill Prometheus、以及前端如何在同一份 session 上完成 topology 與 Grafana 的歷史觀察。

若要先看使用者角度，請讀：

- [`../../guides/ui-workflows/dvr-controls.md`](../../guides/ui-workflows/dvr-controls.md)
- [`../frontend/events-and-dvr.md`](../frontend/events-and-dvr.md)
- [`../grafana/rendering.md`](../grafana/rendering.md)

## 1. 入口

目前 replay 由 `run.py` 啟動：

```bash
uv run run.py replay sessions/<session_id> --profile <profile> --backfill=auto|overwrite|skip
```

其中：

- `--backfill=auto`
  - Prometheus 已有該 session 時直接 reuse
  - 否則 backfill
- `--backfill=overwrite`
  - 先刪掉該 session 的 metrics，再重灌
- `--backfill=skip`
  - 不寫 Prometheus；此時 Grafana 可能沒有圖

## 2. Replay 的前提條件

目前 replay 需要兩種資產：

1. 本地 session artifact
   - `events.jsonl`
   - `meta.json`
   - `topology.yaml`
2. 已常駐且可用的 Prometheus

其中本地 session 仍是 replay 的 source of truth。Prometheus 既有 session 最多只能讓 system 省掉 metrics backfill，不能取代本地 `events.jsonl`。

## 3. 啟動流程

### 1. `run.py` 先做 session / Prometheus 判斷

在真正啟動 app 前，`run.py` 會：

1. 載入 profile `config.yaml`
2. 確認 Prometheus 已常駐並可 reload
3. 透過 `services/replay_session_service.py` 讀出：
   - 本地 session 是否存在
   - event count、時間範圍
   - Prometheus 是否已有該 session
4. 依 `--backfill` policy 決定本次 replay 的 metrics 策略

### 2. Backend 載入 session 並重建 state

`backend.app` 啟動後會：

1. 載入 session `meta.json`
2. 載入 `events.jsonl`
3. 載入 session 目錄中的 `topology.yaml`
4. 依事件重建 `runtime/state.py`
5. 若 policy 需要，執行 replay backfill

replay mode 不會啟動：

- SSH collector
- live queue consumer
- live WebSocket 事件來源

## 4. Replay 的兩個資料面

replay 下仍然要分清楚兩個資料面。

### event / topology

前端會先讀：

- `GET /api/session-info`
- `GET /api/events`

之後把整份事件放進本地 `_events`，所有：

- play
- pause
- scrub
- keyboard arrow step
- residual edge/pulse continuation

都只是在這份本地事件集合上重新取樣與重播。

### metrics / Grafana

Grafana 不再有 pseudo-live session。

目前 replay 下只有一個 canonical session：

- `var-session=<original_session_id>`

差別只有時間窗：

- paused / scrubbed：絕對 `from/to`
- playing：historical relative `from=now-(offset+window)`、`to=now-offset`

因此現在 replay 播放期間，Grafana 看的是「原始 session 的歷史相對視窗」，不是額外映射到現在的新 session。

## 5. Replay backfill

### 1. 角色

replay backfill 的角色很單純：

- 讓這份 session 的 metrics 以原始時間戳存在於 Prometheus
- 讓 paused / scrubbed / playing 三種 replay chart 模式都能查同一個原始 session

### 2. 執行方式

backfill 會把 metric-bearing events 重建成 Prometheus time series，再用 remote write 寫進 TSDB。

這條路徑保留原始時間戳，不會把樣本重映射到 `now`。

### 3. overwrite 的正式語意

`overwrite` 的語意已正式收斂為：

1. 刪除 `session=<session_id>` 的 `nwdaf_*` series
2. 再重新 backfill

這比早期的「每次啟動先清整個 TSDB」更精準，也比較符合持久 Prometheus 的心智。

## 6. 前端 replay 狀態機

目前前端 replay 的主要狀態仍是：

- `PAUSED`
- `PLAYING`
- `SCRUBBING`

但它們現在只影響兩件事：

1. topology / event log 如何從本地 `_events` 取樣
2. Grafana iframe 的 `from/to/refresh`

已不存在的行為包括：

- replay speed
- `/api/replay/play`
- `/api/replay/pause`
- `/api/replay/speed`
- `MetricPlayer`
- pseudo session cleanup

## 7. `skip` policy 的含義

`--backfill=skip` 目前是刻意保留的簡化模式：

- replay 仍可正常載入 session、播放 topology、看 event log
- 但如果 Prometheus 本來沒有這個 session 的 metrics，Grafana chart 會是空的

這條模式主要用於只想看 topology / event path，而暫時不處理 chart 的情境。

## 8. 與 live 的差異

| 面向 | live | replay |
|---|---|---|
| 事件來源 | 遠端 log 即時解析 | 本地 `events.jsonl` |
| 前端事件來源 | `/ws` | `/api/events` 一次載入 |
| chart metric 來源 | `/metrics` scrape | original session backfill |
| 播放中的 chart | `now-window ~ now` 查當前 live session | historical relative 查原始 replay session |
| Prometheus 寫入策略 | 持續 scrape | 啟動時依 policy backfill |

## 9. 目前限制

- replay 仍需要本地 session artifact；Prometheus 既有 session 不能單獨替代它
- `skip` 在 Prometheus 無資料時會讓 chart 為空
- replay 會把整份事件載入前端記憶體；超長 session 的初始載入成本仍與 event count 成正比
- replay chart 與 topology 雖共享同一份 session，但一邊是 Grafana query，一邊是前端本地重播；兩者並不是單一事件 loop 在驅動

若看到 `start.sh`、pseudo-live、`MetricPlayer`、replay speed 或 replay control APIs，應視為 pre-refactor historical runtime，不是目前系統。
