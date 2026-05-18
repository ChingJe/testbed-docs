# Session

本文描述 `5g-viz` 目前如何建立、保存、列舉與載入 DVR session。

## 1. Session 的角色

在目前架構中，一個 session 代表一次可錄製、可重播的實驗執行個體。

session 保存的不是畫面快照，而是三個核心檔案：

- `meta.json`
- `events.jsonl`
- `topology.yaml`

其中：

- `events.jsonl` 是 replay 的核心事件來源
- `meta.json` 提供 session metadata
- `topology.yaml` 提供當時的 topology / reaction snapshot

## 2. 目錄結構

目前 session 目錄長成：

```text
sessions/<session_id>/
  meta.json
  events.jsonl
  topology.yaml
```

### `session_id`

session id 由 UTC 時間產生，格式近似：

```text
YYYYMMDDTHHMMSSmmm
```

用來避免同秒啟動撞名。

## 3. Live 錄製流程

live mode 啟動後，backend 會：

1. 載入目前 profile 的 `topology.yaml`
2. 建立新的 session 目錄
3. 複製 `topology.yaml` 到 session 內
4. 寫入初始 `meta.json`
5. 以 append mode 開啟 `events.jsonl`

之後每成功 parse 一筆 event，就會依序：

1. append 到記憶體 `_events`
2. append 到 `events.jsonl`
3. 更新 metrics
4. broadcast 到 WebSocket clients

也就是說，錄製寫檔發生在 metrics 更新與前端廣播之前。

## 4. `meta.json` 目前保存什麼

session 建立時，`meta.json` 至少包含：

- `session_id`
- `profile`
- `grafana_groups`
- `start_time`

live 正常結束時，backend 會再補上：

- `end_time`
- `event_count`

若錄製期間發生 JSONL 寫入失敗，也會標記：

- `corrupted: true`

目前 `/api/sessions` 與 replay session status 都會利用這些欄位。

## 5. `events.jsonl` 的語意

`events.jsonl` 每行是一個 JSON object，內容與前端一般事件 payload 對齊。

它的特性是：

- append-only
- 每筆事件都 `write + flush`
- 保持 live parse 成功的先後順序

這份檔案目前被視為 replay 的唯一權威事件來源。Prometheus replay backfill 與 topology replay 都是從它衍生，不另外依賴 live 時的 Prometheus TSDB。

## 6. 為什麼 `topology.yaml` 也要保存進 session

replay mode 並不優先讀現在 profile 下的 `topology.yaml`，而是優先讀 session 目錄內保存的那一份。

原因是：

- `event_reactions` 會直接影響拓樸重建
- `nodes` / `nf_aliases` / `edge_styles` 也會影響 replay 畫面

若只保留事件而不保留當時的 topology config，舊 session 在未來 profile 演變後可能無法被忠實重播。

## 7. 錄製失敗時的降級

目前 JSONL 寫入採保守策略：

- 每筆事件有限次重試
- 第一次明確失敗時就把 session 標成 `corrupted`
- 若連續失敗過多，會停用後續錄製

但即使錄製停用：

- `_events` 記憶體緩衝仍持續累積
- WebSocket broadcast 繼續
- metrics 更新繼續

也就是說，live 體驗優先於錄製完整性。代價是這個 session 之後的 replay 不再保證完整。

## 8. Replay 載入流程

replay mode 啟動時，backend 會：

1. 解析 session 目錄
2. 驗證 `events.jsonl` 存在
3. 載入 `meta.json`
4. 載入 `events.jsonl`
5. 載入 session 自帶的 `topology.yaml`
6. 推導 `session_id`、`start_time`、`end_time`

其中：

- `session_id` 優先取 `meta.json["session_id"]`
- `start_time` / `end_time` 若缺失，會用第一筆 / 最後一筆 event 補

所以即使 live 非正常中止，只要 `events.jsonl` 還在，多數情況下仍可 replay。

## 9. Session 查詢與狀態

### `GET /api/sessions`

`/api/sessions` 會掃描 `sessions/` 目錄，回傳可見 session 的 metadata。

### `GET /api/events`

`/api/events` 是前端 timeline、scrub 與 replay 初始化的主要資料來源。

它支援：

- `session`
- `from`
- `to`
- `limit`
- `offset`

目前直接回傳完整 event objects，不做額外投影。

### CLI `session-status`

目前更正式的 diagnostics 入口是：

```bash
uv run run.py session-status <session_ref> --profile <profile>
```

它會同時檢查：

- 本地 session 是否存在
- 是否 replayable
- event count / time range
- Prometheus 中是否已有這個 session

## 10. 可攜性現況

從實作上看，一個 session 目錄已具備基本可攜性，因為它同時保存：

- event log
- topology snapshot
- session metadata

目前 repo 也已提供：

- `replay/import_logs_cli.py`

可從原始 log 匯入生成新 session。

## 11. 目前限制

- `events.jsonl` 仍以 append 與全量載入為主，沒有索引檔
- `_session_events_cache` 沒有 eviction 機制
- `corrupted` 只標示「此 session 可能不完整」，不標出從哪一筆開始不可信
- session artifact 與 Prometheus metrics 仍是兩份不同資產；Prometheus 既有資料不能單獨取代本地 session directory
