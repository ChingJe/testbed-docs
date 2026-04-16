# DVR Mode: Recording, Sessions, and Prometheus

本文件保留原始章節編號 `§2 ~ §5`。

## 2. 核心設計原則

### JSONL 是唯一真實來源（single source of truth）

目前 event 經 parser 產生後，同時送往 WebSocket（topology）和 Prometheus（Grafana 曲線）。兩者各自獨立，沒有共同的持久紀錄。

新架構在資料流中加入一層 **JSONL event log**，作為所有資料的權威來源：

```
collector → parse → event
                      ├─→ append events.jsonl      （持久紀錄，唯一真實來源）
                      ├─→ broadcast WebSocket       （即時 topology）
                      └─→ update Prometheus metrics  （即時 Grafana 曲線）
```

Prometheus 是 JSONL 的「衍生視圖」——從 JSONL 中的 metric event（`aggregated_slot`、`ml_inference`、`accuracy`、`retrain_trigger` 等）即可完整重建所有 Prometheus 時間序列。這代表：

- **錄製**只需要寫 JSONL，不需要額外備份 Prometheus 資料。
- **Replay** 時從 JSONL 回填 Prometheus，Grafana 曲線自動出現。
- **導出**只需要打包 JSONL，幾 MB 就是一次完整實驗。

**邊界條件**：JSONL 是 replay / export 的唯一來源，但 live 體驗不應因 JSONL 寫入失敗而中斷。若寫入失敗，session 標記為 `corrupted`，live 的 WebSocket + Prometheus 繼續運作，但該 session 的 replay / export 不再保證完整（見 §12.9）。

---

## 3. Session 管理

### Session 定義

每次啟動 5g-viz（live 模式）即產生一個 session。

### Session ID

格式：`YYYYMMDD'T'HHMMSSfff`，**一律使用 UTC**。`fff` 為毫秒，避免同秒啟動撞名。

範例：`20260413T063000123`（UTC 06:30:00.123 = 本地 14:30:00.123 +08:00）。

Python 產生方式：`datetime.now(timezone.utc).strftime('%Y%m%dT%H%M%S') + f'{ms:03d}'`。

選擇 UTC 的原因：session 檔案可能在不同時區的機器之間傳遞（導出/匯入），UTC 確保 ID 排序 = 時間排序，不因時區不同而亂序。`meta.json` 中的 `start_time` / `end_time` 則帶完整 timezone offset，方便人類閱讀。

### 目錄結構

```
5g-viz/
  sessions/
    20260413T063000123/
      meta.json
      events.jsonl
    20260414T013000456/
      meta.json
      events.jsonl
```

### meta.json

```json
{
  "session_id": "20260413T063000123",
  "profile": "default",
  "grafana_groups": ["1", "2"],
  "start_time": "2026-04-13T14:30:00.123+08:00",
  "end_time": "2026-04-13T15:45:12.000+08:00",
  "event_count": 8234
}
```

- `start_time`：session 建立時寫入。
- `end_time`、`event_count`：5g-viz 正常結束（SIGINT / SIGTERM）時寫入。若非正常關閉，這兩個欄位可能缺失；replay 時從 JSONL 最後一筆 event 的 timestamp 推斷。

### events.jsonl

每行一個 JSON object，與目前 WebSocket broadcast 的 event 格式完全一致：

```jsonl
{"type":"nf_up","time":"2026-04-13T14:30:05.123+08:00","nf":"SMF"}
{"type":"aggregated_slot","time":"2026-04-13T14:30:10.456+08:00","sub_id":"imsi-001","target":"group=1","ul_vol":1234.0,"dl_vol":5678.0,"ts":"...","ul_thr":100.0,"dl_thr":200.0}
{"type":"ml_inference","time":"2026-04-13T14:30:10.789+08:00","sub_id":"imsi-001","target":"group=1","ul_vol":1100,"dl_vol":5500,"confidence":95}
```

寫入使用 append mode（`open(..., 'a')`），每筆 event 一次 `write` + `flush`，確保 crash 時最多遺失最後一筆。

---

## 4. Prometheus Session 隔離

### 現有問題

目前 `main.py` 每次啟動都呼叫 `clear_metrics()` 刪除所有 `nwdaf_*` series。原因是快速重啟時，Prometheus TSDB 殘留的舊 Gauge 值會被 Grafana 顯示為「延續到新 session」的假資料。

### 解決方式：Session label

所有 `nwdaf_*` metrics 加上 `session` label：

```
nwdaf_ground_truth_ul_bytes{session="20260413T063000123", sub_id="imsi-001", target="group=1"}
```

效果：

- 不同 session 的 metrics 天然隔離，**不再需要 `clear_metrics()`**。
- 舊 session 資料留在 Prometheus TSDB，查閱期限取決於 Prometheus retention 設定；若未特別調整，通常沿用 Prometheus 的預設 retention。
- Grafana dashboard 加 `session` template variable，使用者透過下拉選單切換 session。

### Grafana Dashboard 變更

- 新增 template variable：`session`，query =
  `query_result(count by (session) (count_over_time(nwdaf_retrain_total[365d])))`，
  並以 regex 抽出 `session="..."` label。
- 所有 panel 的 PromQL 加 `session="$session"` filter。
- 預設選取最新的 session。

### 注意事項

- `nwdaf_retrain_total` 仍是 Counter 類型，但 retrain annotation 已改由
  `nwdaf_retrain_start_event` / `nwdaf_retrain_done_event` pulse gauge 處理，不再依賴 `idelta()`。
- Session label 會增加 Prometheus 的 label cardinality，但 session 數量有限（一天通常個位數），不構成問題。

---

## 5. Replay 時的 Prometheus 回填

### 目的

Replay 舊 session（或在別人的機器上開啟導入的 session）時，Prometheus 裡沒有對應的 metrics 資料。需要從 JSONL 重建。

### 方法：Remote Write

Prometheus 支援 remote write receiver（啟動時加 `--web.enable-remote-write-receiver`）。5g-viz replay 模式在開始播放前，從 JSONL 中提取所有 metric event，以 **原始 timestamp** 透過 remote write API 批量寫入 Prometheus。

流程：

1. **檢查是否已回填**：向 Prometheus 查詢 `count(nwdaf_ground_truth_ul_bytes{session="<id>"})`。若有結果，代表該 session 已回填過，跳過步驟 2~4（log 提示 "session already backfilled, skipping"）。這避免重複回填的等待時間。
2. 讀取 `events.jsonl`，篩選出 metric 相關的 event types（`aggregated_slot`、`ml_inference`、`accuracy`、`retrain_trigger`、`retrain_done`、`model_swap`）。
3. 將每筆 event 的數值轉為 Prometheus sample（timestamp + value + labels，包含 `session` label）。
4. 以 batch 方式 POST 到 `http://localhost:9090/api/v1/write`（Prometheus remote write endpoint）。
5. 回填完成後，Grafana 即可查詢該 session 的完整曲線。

若需要強制重新回填（例如 events.jsonl 被修正過），可在 replay 啟動時加 `--force-backfill` flag，跳過步驟 1 的檢查。Prometheus 對相同 timestamp + 相同值的重複寫入是 idempotent 的，強制重寫不會造成資料異常。

### 注意事項

- **start.sh 需加 `--web.enable-remote-write-receiver` flag**。此 flag 開啟一個 HTTP endpoint 接收 remote write，預設關閉。
- Remote write payload 格式為 protobuf（Prometheus remote write spec）。Python 端可用 `prometheus-remote-write` 或手動組裝 snappy-compressed protobuf。需評估是否值得引入額外依賴，或自行實作精簡版（payload 格式固定且簡單）。
- 批量寫入大量歷史資料（幾千筆 sample）應該在幾秒內完成，不影響使用體驗。但若 session 非常長（數萬筆 metric event），需注意 batch size 和記憶體用量。
- 若 Prometheus 裡已經有該 session 的資料（例如 live 模式錄製的），remote write 會產生重複 sample。Prometheus TSDB 對相同 timestamp + 相同值的重複寫入是 idempotent 的，不會造成問題；但若值不同（理論上不應該發生）會以後寫入的為準。

---
