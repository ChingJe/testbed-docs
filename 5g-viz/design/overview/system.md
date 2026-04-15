# 5g-viz 系統概觀

本文描述目前 `5g-viz` 的系統組成、執行模式與主要對外介面。內容以目前程式碼實作為準。

## 1. 系統目的

`5g-viz` 是一個以 FastAPI 為核心的即時視覺化服務，負責：

- 透過 SSH 持續讀取 5GC VM 上的 log 檔
- 將原始 log 解析成結構化事件
- 把事件同步送到瀏覽器拓樸圖與事件列表
- 將特定事件轉成 Prometheus metrics，供 Grafana 呈現
- 在 replay 模式下重播歷史 session，並以 pseudo-live 方式重建圖表體驗

目前系統同時支援 `live` 與 `replay` 兩種執行模式。

## 2. 執行模式

| 模式 | 啟動方式 | 事件來源 | 圖表資料來源 | 前端取得資料的方式 |
|---|---|---|---|---|
| `live` | `./start.sh` | 透過 SSH tail 遠端 log | 後端在收到事件時即時更新 `/metrics`，由 Prometheus scrape | WebSocket `/ws` + 補抓 `/api/state` / `/api/events` |
| `replay` | `./start.sh --replay sessions/<session_id>` | 從磁碟上的 `events.jsonl` 載入 | 啟動時先做 replay backfill；播放時再由 pseudo-live remote write 送入 Prometheus | 啟動時直接抓 `/api/events` 與 `/api/session-info`，不建立 WebSocket |

### live 模式

- `start.sh` 會啟動 Prometheus，然後用 `uvicorn` 啟動 FastAPI。
- `main.py` 在 lifespan 期間讀取 `profiles/<profile>/topology.yaml`，建立新的 session 目錄，並啟動：
  - `_process_queue()`：消費 collector 推入的原始 log
  - `collector.start()`：對遠端 VM 建立 SSH tail
  - `_setup_grafana()`：建立或更新 Grafana dashboard

### replay 模式

- `start.sh --replay` 會先清空本機 Prometheus data dir，再重新啟動 Prometheus。
- `main.py` 會讀取指定 session 目錄中的 `meta.json`、`events.jsonl` 與 `topology.yaml`。
- 啟動時會先把歷史 metrics 做一次 remote write backfill，讓 Grafana 能查到原始 session 資料。
- 若使用者按下播放，`MetricPlayer` 會建立一個新的 `pseudo_session`，把歷史 metric 重新映射到「現在時間」並持續 remote write。

## 3. 主要組件

| 組件 | 主要檔案 | 責任 |
|---|---|---|
| 啟動腳本 | `start.sh` | 設定 `PROFILE` / `SESSION_MODE` / `SESSION_PATH`，管理 Prometheus 啟停，啟動 `uvicorn` |
| 設定載入 | `config.py` | 讀取 `profiles/<profile>/.env`，提供 Grafana 與 UPF mapping 設定 |
| 主應用 | `main.py` | FastAPI app、lifespan、session 管理、事件記錄、API、WebSocket、replay backfill |
| 遠端收集器 | `collector.py` | 依 `topology.yaml` 的 `ssh_sources` 設定連線到 VM，對 log 檔執行 `tail -F` |
| 解析器 | `parser.py` | 將 logrus 格式 log 解析成 base dict，依 `rules/` 規則產生事件 |
| 規則與 metrics | `rules/__init__.py`、`rules/*.py` | 匯整 parser rules 與 metric handlers，並注入目前 session ID |
| 狀態模型 | `state.py` | 根據 topology config 初始化 node 狀態，從事件與 `event_reactions` 推導 `state_snapshot` |
| Replay metric player | `metric_player.py` | replay 模式下產生 pseudo-live metric stream，支援 play / pause / speed update |
| Grafana 設定器 | `grafana_setup.py` | 建立或更新 Prometheus datasource 與 dashboard |
| 前端 | `frontend/index.html`、`frontend/events.js`、`frontend/topology.js` | 初始化 topology、建立 WebSocket、渲染事件、控制 DVR 與 Grafana iframe |

## 4. Session 與持久化

### live session

live 模式啟動時，`main.py` 會建立新的 `sessions/<session_id>/` 目錄，並寫入：

- `meta.json`
- `events.jsonl`
- `topology.yaml` 的 session copy

其中：

- `session_id` 由 UTC 時間戳產生
- `events.jsonl` 逐筆追加 parser 產生的事件
- 結束時會回填 `meta.json` 的 `end_time` 與 `event_count`

### replay session

replay 模式不會建立新的事件檔，而是讀取既有 session：

- `meta.json`：session 基本資訊
- `events.jsonl`：歷史事件
- `topology.yaml`：當時錄製時使用的 topology config

這讓 replay 模式可以使用錄製當時的拓樸設定，而不是目前 profile 下的最新設定。

## 5. 對外介面

### HTTP / API

| 路徑 | 方法 | 用途 |
|---|---|---|
| `/metrics` | `GET` | 輸出 Prometheus scrape endpoint |
| `/api/grafana-config` | `GET` | 回傳 Grafana base URL 與 dashboard UID |
| `/api/session-info` | `GET` | 回傳目前模式、session ID、起訖時間 |
| `/api/sessions` | `GET` | 列出本機 `sessions/` 目錄中的 session |
| `/api/events` | `GET` | 依 session、時間範圍、offset/limit 查事件 |
| `/api/topology-config` | `GET` | 回傳目前載入的 topology config |
| `/api/state` | `GET` | 回傳目前 `state_snapshot` |
| `/api/replay/play` | `POST` | replay 模式下啟動 pseudo-live metric stream |
| `/api/replay/pause` | `POST` | replay 模式下停止指定 pseudo session |
| `/api/replay/speed` | `POST` | replay 模式下更新 pseudo-live 播放速度 |

### WebSocket

| 路徑 | 用途 |
|---|---|
| `/ws` | live 模式下的事件推播通道；新連線會先收到一份 `state_snapshot` |

### 靜態前端

| 路徑 | 用途 |
|---|---|
| `/` | 掛載 `frontend/` 靜態檔案，提供主畫面 |

## 6. 前端初始化方式

前端啟動時有兩條固定初始化流程：

1. `topology.js` 先抓 `/api/topology-config`，用來建立 Cytoscape 拓樸與 filter sidebar。
2. `events.js` 再抓：
   - `/api/grafana-config`
   - `/api/session-info`

之後依模式分流：

- `live`：建立 WebSocket 連線到 `/ws`
- `replay`：直接從 `/api/events` 載入整份 session 事件，再用本地狀態機控制播放

## 7. 與外部系統的關係

### 5GC VM

`collector.py` 依 `topology.yaml` 的 `ssh_sources` 設定，對遠端 VM tail 兩種 log：

- `free5gc.log`
- `nwdaf.log`

`free5gc.log` 透過「先找最新子目錄，再 tail 該檔案」的方式追蹤；`nwdaf.log` 則直接 tail 指定路徑。

### Prometheus

- 由 `start.sh` 啟動
- scrape `5g-viz` 的 `/metrics`
- replay 模式下接受 remote write backfill 與 pseudo-live emit

### Grafana

- 由 `grafana_setup.py` 自動建立 datasource 與 dashboard
- 前端以 iframe 嵌入 dashboard
- dashboard 透過 `session` variable 切換不同 session 的 metrics

## 8. 系統邊界

目前 `5g-viz` 的責任範圍是「讀取既有 log、轉成事件與 metrics，並提供 live / replay 視覺化介面」。

它目前**不**直接負責：

- 啟動、停止或編排 5GC / NWDAF 元件
- 修改 NF 本身的行為或訓練流程
- 直接實作自有圖表引擎；圖表仍委由 Grafana 呈現
- 將 Prometheus TSDB 當作長期錄製格式；可攜的錄製資料仍以 session 目錄中的 `meta.json`、`events.jsonl`、`topology.yaml` 為主

因此，`5g-viz` 比較接近一個「以 log 為來源的觀測與重播層」，而不是 5G testbed 的控制平面。
