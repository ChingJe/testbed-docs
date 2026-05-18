# DVR Overview

本文從功能層描述目前 `5g-viz` 的 DVR。這裡的 DVR 不只是前端控制列，而是一組跨越 session 錄製、事件重建、Grafana 視窗切換與 replay session management 的整體機制。

## 1. DVR 的目標

目前 DVR 的目標是讓使用者能在同一套 UI 中做三件事：

- 在 live 實驗進行中暫停、拖曳與回看既有事件
- 將一次實驗錄成可重播的 session
- 事後以 replay 模式重看 topology、event log 與 Grafana 曲線

其中：

- topology / event log 以 event stream 為主
- Grafana 曲線以 Prometheus metrics 為主
- session 錄製則以 `events.jsonl` 為核心持久化格式

## 2. 兩種執行模式

### live

live mode 維持 collector -> parser -> backend -> frontend 的即時管線，同時：

- 持續寫入 `events.jsonl`
- 更新 live session `meta.json`
- 持續把 metrics 暴露到 `/metrics`

使用者進入 pause / scrub 時，backend 不會停下來；只是前端暫時停止即時消費最新事件。

### replay

replay mode 則是重新開一個唯讀 runtime，從本地 session artifact 載入：

- `events.jsonl`
- `meta.json`
- `topology.yaml`

接著：

- 由前端本地重播 event / topology
- 由後端視 policy 決定是否 backfill Prometheus

## 3. Live DVR 操作路徑

### Pause

當使用者按下 Pause：

- frontend 狀態從 `LIVE` 進入 `PAUSED`
- backend 仍持續 collect、parse、寫 session、更新 metrics
- 前端仍會把新事件收進 `_events`，但不再即時 dispatch 到 topology / log
- Grafana 從 `now-window ~ now` 切成當前停點對應的絕對時間窗

### Scrub

當使用者拖曳 timeline：

- frontend 進入 `SCRUBBING`
- topology 改由 `Topology.renderStaticSnapshot(...)` 重建指定時間點
- event log 改為顯示該時間點之前的近期尾段
- 若本地 buffer 不夠早，可向 `/api/events` 補拉該 live session 的歷史事件

### Play

當使用者從 live paused 狀態按下 Play：

- frontend 依事件原始時間差重播本地 `_events`
- backend 仍持續向前產生新事件
- 若播放追到 live edge，前端會回到 `LIVE`

### Go Live

當使用者按下 Go Live：

- frontend 先請求最新 `state_snapshot`
- 套用 `Topology.applySnapshot(...)`
- timeline 跳到最新位置
- Grafana 恢復 `now-window ~ now`

## 4. Replay 操作路徑

目前 replay 已收斂成較單純的模型。

### 啟動

`run.py replay` 啟動前會先做：

- local session status 檢查
- Prometheus session status 檢查
- `auto / overwrite / skip` 決策

### 前端資料來源

replay 前端不建立 live WebSocket 事件來源，而是：

1. 抓 `/api/session-info`
2. 抓 `/api/events`
3. 在本地建立 timeline
4. 以本地 `_events` 驅動 pause / scrub / play / resume

### Grafana

目前 replay Grafana 的 canonical model 是：

- 始終查原始 session
- paused / scrubbed 用絕對時間窗
- playing 用 historical relative 時間窗

因此現在 replay 已不再有：

- pseudo-live
- `MetricPlayer`
- replay speed
- replay mutation APIs

## 5. 前端 DVR 狀態

前端目前仍可視為有這幾個主要狀態：

| 狀態 | 主要意義 |
|---|---|
| `LIVE` | 即時顯示最新事件 |
| `PAUSED` | 停在某一時間點 |
| `SCRUBBING` | 使用者正在拖曳 timeline |
| `PLAYING` | 從目前時間點往前播放後續事件 |
| `RESUMING` | 從 paused / scrubbed 回到播放或 live |

這些狀態定義的是前端如何消費事件，不代表 backend pipeline 本身會停下來。

## 6. 三條需要一起協調的資料面

### topology

- 來源是結構化 events
- live 時即時 dispatch
- paused / scrubbing 時以 static snapshot 重建
- playing 時按事件原始時間差重播

### event log

- 與 topology 同源
- live append 與 paused tail render 共用同一份 `_events`

### Grafana

- 來源不是 event 本身，而是 Prometheus 內的 metrics
- live、paused、replay、playing 的差異主要在 query window

## 7. Session artifact 與 Prometheus 的分工

目前 replay 要正常工作，需要分清楚兩類資產：

### session artifact

canonical replay artifact 仍是本地 session 目錄：

- `events.jsonl`
- `meta.json`
- `topology.yaml`

### Prometheus metrics

Prometheus 端已有同名 session 時，只能讓 system 省掉 backfill；它不能替代本地 `events.jsonl`。

因此：

- 本地 session 不存在 => 不能 replay
- 本地 session 存在、Prometheus 無資料 => 可 replay，但 chart 視 policy 可能為空或需要 backfill

## 8. 目前邊界

目前 DVR 已完成的範圍包括：

- live session 錄製
- replay 載入
- `/api/sessions`、`/api/events`、`/api/state`
- timeline pause / scrub / play / go-live
- replay session status / backfill policy
- replay chart 的 historical relative 播放

目前不再屬於系統一部分的舊模型包括：

- pseudo-live metrics stream
- replay speed
- `MetricPlayer`
- replay chart window change 需重啟 pseudo-live

## 9. 閱讀順序

若要沿著 DVR 主線閱讀，建議順序是：

1. [session.md](./session.md)
2. [replay.md](./replay.md)
3. [../frontend/events-and-dvr.md](../frontend/events-and-dvr.md)
4. [../frontend/grafana-embed.md](../frontend/grafana-embed.md)
