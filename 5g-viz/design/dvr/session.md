# Session

本文描述 `5g-viz` 目前如何建立、保存、列舉與載入 DVR session。

## 1. Session 的角色

在目前架構中，一個 session 代表一次可錄製、可重播的實驗執行個體。

session 的保存內容不是畫面快照，而是三個檔案：

- `meta.json`
- `events.jsonl`
- `topology.yaml`

其中 `events.jsonl` 是 replay 的核心來源；另外兩個檔案則提供 metadata 與渲染所需的配置快照。

## 2. 目錄結構

live mode 啟動後，`main.py` 會在：

```text
5g-viz/sessions/<session_id>/
```

建立新的 session 目錄。實際內容如下：

```text
sessions/<session_id>/
  meta.json
  events.jsonl
  topology.yaml
```

### `session_id`

session ID 由 `_new_session_id_utc()` 產生，格式是：

```text
YYYYMMDDTHHMMSSmmm
```

使用 UTC 時間與毫秒尾碼，避免同秒啟動撞名。

## 3. Live 錄製流程

live mode 進入 lifespan 後，`main.py` 會：

1. 載入目前 profile 的 `topology.yaml`
2. 建立新的 session 目錄
3. 複製 `topology.yaml` 到 session 內
4. 寫入初始 `meta.json`
5. 以 append mode 開啟 `events.jsonl`

之後 `_process_queue()` 每成功 parse 一筆 event，就會依序：

1. append 到記憶體 `_events`
2. append 到 `events.jsonl`
3. 更新 metrics
4. broadcast 到 WebSocket clients

這表示檔案寫入發生在 metrics 更新與 WebSocket broadcast 之前。

## 4. `meta.json` 目前保存什麼

session 建立時，`meta.json` 至少包含：

- `session_id`
- `profile`
- `grafana_groups`
- `start_time`

live 正常結束時，`_finalize_live_session()` 會再補上：

- `end_time`
- `event_count`

若錄製期間發生 JSONL 寫入失敗，也會補上：

- `corrupted: true`

這些欄位不是 schema 驗證過的正式契約，但目前 `/api/sessions` 與 replay 初始化都依賴它們。

## 5. `events.jsonl` 的語意

`events.jsonl` 每行是一個 JSON object，內容與前端一般事件 payload 對齊。

特性：

- 使用 append-only 寫入
- 每筆事件都 `write + flush`
- `ensure_ascii=False`，保留原始 UTF-8 輸出
- 保持 `_process_queue()` 成功 parse 的先後順序，不做額外排序

這個檔案目前被視為 replay 的唯一權威事件來源。Prometheus replay backfill 與 topology replay 都是從這份事件流衍生，不另外依賴 live 時的 Prometheus TSDB。

也就是說，`events.jsonl` 的順序語意是：

- live 錄製時：先寫檔，再更新 metrics / broadcast
- replay 載入時：先依檔案行序讀回事件

前端若要依時間戳做 scrub 或播放，會在自己的 `_events` 緩衝中再做排序；session 檔本身不會在載入時重排。

## 6. `topology.yaml` 為什麼也要複製進 session

replay mode 並不讀現在 profile 下的 `topology.yaml`，而是優先讀 session 目錄內保存的那一份。

原因是：

- `event_reactions` 會直接影響拓樸重建
- `nodes` / `nf_aliases` / `edge_styles` 也會影響 replay 畫面

若只保留事件而不保留當時的 topology config，舊 session 在未來 profile 演變後可能無法被忠實重播。

## 7. 錄製失敗時的降級

`_append_event_jsonl()` 目前採保守策略：

- 每筆事件最多重試 2 次
- 失敗就增加 `_write_fail_streak`
- 第一次失敗時立刻把 session 標成 `corrupted`
- 連續失敗超過 10 次後，直接停用錄製

但即使錄製停用：

- `_events` 記憶體緩衝仍持續累積
- WebSocket broadcast 繼續
- metrics 更新繼續

也就是說，live 體驗優先於錄製完整性。代價是這個 session 之後的 replay 不再保證完整。

## 8. Replay 載入流程

replay mode 啟動時，`_init_replay_session()` 會：

1. 解析 `SESSION_PATH`
2. 驗證目錄存在
3. 驗證 `events.jsonl` 存在
4. 載入 `meta.json`
5. 載入 `events.jsonl`
6. 推導 `session_id`、`start_time`、`end_time`

之後 lifespan 還會再呼叫 `_load_replay_topo_config()`，驗證 session 目錄中的 `topology.yaml` 也存在。缺少這份檔案時，replay 會直接失敗，因為前端與 state 重建都需要它。

其中：

- `session_id` 優先取 `meta.json["session_id"]`，否則退回目錄名
- `start_time` / `end_time` 若缺失，會用第一筆 / 最後一筆 event 的 `time` 補

這也是為什麼非正常中止的 live session 仍可被 replay。

## 9. Session 查詢 API

### `GET /api/sessions`

`/api/sessions` 會掃描 `sessions/` 目錄，回傳所有可見 session 的 metadata。

特性：

- live mode 下，正在錄製中的 current session 會直接從記憶體狀態生成最新回傳值
- 舊 session 則讀 `meta.json`，必要時回推 `start_time`、`end_time`、`event_count`
- 已載入的事件會放進 `_session_events_cache`

### `GET /api/events`

`/api/events` 是前端 timeline、scrub 與 replay 初始化的主要資料來源。

特性：

- `session` 省略時預設取 current / active session
- 支援 `from`、`to`、`limit`、`offset`
- 上限固定 `50000`
- 若 session 尚未快取，會從 `events.jsonl` 載入

目前這條 API 直接回傳完整 event objects，不做額外投影。

## 10. 可攜性現況

從實作上看，一個 session 目錄已經具備基本可攜性，因為它同時保存：

- event log
- topology config snapshot
- grafana groups 與 profile metadata

但目前 repo 內還沒有正式的 export / import CLI。現況仍是手動複製或打包 `sessions/<id>/` 目錄，再用：

```text
./start.sh --replay sessions/<id>
```

讀取。

## 11. 目前限制

- `events.jsonl` 只做 append 與全量載入，沒有索引檔；長 session 的隨機區間查詢效率有限
- `_session_events_cache` 目前沒有 eviction 機制，查過的 session 會留在程序記憶體中
- 錄製失敗時雖會標記 `corrupted`，但沒有更細緻地標出從哪一筆事件開始不可信
- export / import 仍停留在「手動複製 session 目錄」階段，沒有額外驗證或版本檢查
