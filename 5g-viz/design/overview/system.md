# 5g-viz 系統概觀

本文描述目前 `5g-viz` 的系統組成、執行模式與主要對外介面。內容以目前程式碼實作為準。

## 1. 系統目的

`5g-viz` 是一個以 FastAPI 為核心的觀測與 replay 介面，負責：

- 透過 SSH 持續讀取 5GC VM 上的 log
- 將原始 log 解析成結構化事件
- 把事件同步送到瀏覽器 topology 與 event log
- 將特定事件轉成 Prometheus metrics，供 Grafana 顯示
- 在 replay 模式下重播歷史 session，並以 historical relative Grafana query 保持播放中的平滑圖表體驗

目前系統同時支援 `live` 與 `replay` 兩種執行模式。

## 2. 主要目錄與模組

目前 runtime 已收斂成下列模組層次：

| 層次 | 主要路徑 | 責任 |
|---|---|---|
| CLI / entrypoint | `run.py` | 啟動 live / replay、session status、Prometheus helper commands |
| backend | `backend/app.py` | FastAPI app、lifespan、HTTP API、WebSocket、Grafana proxy |
| runtime | `runtime/*` | profile loader、runtime context、state |
| services | `services/*` | collector、Prometheus 管理、Grafana setup、replay session status |
| replay | `replay/*` | session I/O、importer、parser-related replay utilities |
| frontend | `frontend/*` | topology、event log、DVR controls、Grafana iframe |
| rules | `rules/*` | parser rules、metric handlers、reaction helper |

## 3. 執行模式

| 模式 | 啟動方式 | 事件來源 | Grafana / metrics 路徑 | 前端主要資料來源 |
|---|---|---|---|---|
| `live` | `uv run run.py live --profile <name>` | 遠端 VM log | 後端即時更新 exporter metrics，由 Prometheus scrape | `/ws` + `/api/state` + `/api/events` |
| `replay` | `uv run run.py replay sessions/<id> --profile <name>` | `events.jsonl` | 啟動時先做 replay backfill；播放時改用原始 session + historical relative query | `/api/session-info` + `/api/events` |

### Live

- `run.py` 會先檢查 Prometheus 是否已在跑，並同步 managed `prometheus.yml`
- `backend/app.py` 在 lifespan 中載入目前 profile 的 `config.yaml` 與 `topology.yaml`
- collector 依 `config.yaml.collector.sources` 建立 SSH tail
- parser / rules 產生事件
- 事件同時驅動：
  - `state_snapshot`
  - `events.jsonl`
  - WebSocket broadcast
  - live metrics exporter

### Replay

- `run.py replay` 會先做 session status 檢查，並依 `--backfill=auto|overwrite|skip` 決定 replay metric policy
- app 啟動時載入 session 內的 `meta.json`、`events.jsonl`、`topology.yaml`
- 若 policy 允許，app 會把歷史 metric event 以原始 timestamp remote write 回 Prometheus
- 前端播放 topology / event log 時只使用本地事件集合；Grafana 播放則維持查原始 session，但使用 historical relative time range

## 4. Session 與持久化

一個 session 代表一次可錄製、可重播的實驗觀測單位。核心檔案是：

- `meta.json`
- `events.jsonl`
- `topology.yaml`

### live session

live 啟動後會建立新的 `sessions/<session_id>/`：

- `meta.json` 記錄 profile、開始時間、Grafana groups 等 metadata
- `events.jsonl` 逐筆附加結構化事件
- `topology.yaml` 複製目前 profile 的拓樸設定快照

### replay session

replay 不建立新的事件檔，而是讀取既有 session：

- `meta.json`：session metadata
- `events.jsonl`：歷史事件
- `topology.yaml`：錄製當下使用的 topology config

因此 replay 畫面優先對齊錄製當下的 session artifact，而不是現在 profile 最新內容。

## 5. Prometheus 與 Grafana

### Prometheus

目前 Prometheus 是：

- 單一 persistent TSDB
- 由 `run.py prom install-user-service` 產生 user service 定義
- 正式建議以 `systemctl --user` 常駐

`run.py live/replay` 不再自行拉起 Prometheus；若 Prometheus 沒開，只會提示使用者先啟動 managed service。

### Grafana

Grafana 由 `services/grafana_setup.py` 在 app 啟動時建立或更新：

- Prometheus datasource
- dashboard panels
- annotations
- `session` variable

前端不直接組 PromQL，只透過 iframe query string 控制：

- `var-session`
- `from`
- `to`
- `refresh`

## 6. 對外介面

### CLI

由 `run.py` 提供：

- `uv run run.py live --profile <name>`
- `uv run run.py replay sessions/<id> --profile <name> --backfill=auto|overwrite|skip`
- `uv run run.py session-status sessions/<id> --profile <name>`
- `uv run run.py prom status --profile <name>`
- `uv run run.py prom delete-session <session_id> --profile <name>`
- `uv run run.py prom install-user-service --profile <name>`

### HTTP / WebSocket

主要介面如下：

| 路徑 | 方法 | 用途 |
|---|---|---|
| `/metrics` | `GET` | Prometheus scrape endpoint |
| `/api/grafana-config` | `GET` | 回傳 Grafana base URL 與 dashboard UID |
| `/api/session-info` | `GET` | 回傳目前模式、session ID、起訖時間 |
| `/api/sessions` | `GET` | 列出本機 session |
| `/api/events` | `GET` | 查 session events |
| `/api/topology-config` | `GET` | 回傳目前 topology config |
| `/api/state` | `GET` | 回傳目前 `state_snapshot` |
| `/ws` | WebSocket | live 模式事件推播 |
| `/grafana/*` | HTTP / WebSocket proxy | 代理 Grafana iframe 與 websocket |

注意：舊的 `/api/replay/*` mutation APIs 已移除。

## 7. 前端資料來源

前端啟動時固定會先讀：

- `/api/grafana-config`
- `/api/session-info`
- `/api/topology-config`

之後依模式分流：

### live

- 建立 `/ws`
- 新事件持續進 `_events`
- `Pause / Scrub / Play / Go Live` 都建立在同一份 `_events` 緩衝上

### replay

- 直接從 `/api/events` 載入 session events
- 不使用一般事件 WebSocket
- topology / event log 的播放由前端本地事件集合驅動
- Grafana 依狀態切換固定歷史窗或 historical relative 窗口

## 8. 系統邊界

`5g-viz` 目前的責任是：

- 讀取既有 log
- 轉成事件與 metrics
- 提供 live / replay 視覺化與查詢介面

它目前不直接負責：

- 啟停 5GC / NWDAF 元件
- 編排整個 testbed lifecycle
- 自行實作圖表引擎
- 把 Prometheus TSDB 當作可攜的錄製格式

因此它更接近「以 session artifact 與 Prometheus metrics 為核心的觀測 / replay 層」，而不是 testbed 控制平面。
