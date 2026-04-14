# DVR Mode: Session Recording & Replay

## 0. 進度註記（2026-04-13）

- [x] Session recording + JSONL：已完成 live session 建立、`meta.json`/`events.jsonl` 寫入、結束時回填 `end_time` 與 `event_count`。
- [x] Prometheus session 隔離：`nwdaf_*` metrics 已加 `session` label，Grafana query 與 variable 已改為 session-aware。
- [x] 後端 API：已完成 `/api/session-info`、`/api/sessions`、`/api/events`、`/api/state`，可供 DVR 前端讀取資料。
- [x] Replay 啟動入口：`start.sh --replay <session_path>` 與後端 replay 載入流程已打通。
- [x] 前端 DVR 控制列與狀態機（Pause/Play/Scrub/Go Live）：已完成基礎狀態機與 live timeline 互動（含 replay 基礎入口）。
- [x] 前端 Grafana 同步：scrub/paused 對齊時間窗、play 期間節流更新、Go Live 還原即時視窗。
- [x] Replay 的 Prometheus remote write backfill：已補齊 replay 啟動前 backfill、重複 session 檢查與 `--force-backfill`。
- [ ] Replay 模式 Grafana「近似 live」渲染體驗：pseudo-live pipeline（見 §15）。
- [ ] 匯出/匯入流程與 E2E 驗收清單尚未實作。

## 1. 目標

目前 5g-viz 只支援 live tail——畫面永遠顯示最新狀態，錯過的事件無法回頭看。本功能要達成：

1. **Live DVR**：實驗進行中可暫停、慢速回放、拖曳時間軸到任意時間點，也可一鍵跳回 live。
2. **Session recording**：每次實驗自動錄製到磁碟，事後可完整重播。
3. **可攜帶**：錄製檔案可打包給他人，在另一台機器上還原完整的 topology 動畫與 Grafana 曲線。

---

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

- 新增 template variable：`session`，query = `label_values(nwdaf_ground_truth_ul_bytes, session)`。
- 所有 panel 的 PromQL 加 `session="$session"` filter。
- 預設選取最新的 session。

### 注意事項

- `nwdaf_retrain_total` 是 Counter 類型。`prometheus_client` 每次 process 啟動 Counter 從 0 開始，Prometheus 會視為 counter reset。因為每個 session 有獨立 label，reset 只發生在 session 開頭，不影響 `idelta()` 判斷。
- Session label 會增加 Prometheus 的 label cardinality，但 session 數量有限（一天通常個位數），不構成問題。

---

## 5. Replay 時的 Prometheus 回填

### 目的

Replay 舊 session（或在別人的機器上開啟導入的 session）時，Prometheus 裡沒有對應的 metrics 資料。需要從 JSONL 重建。

### 方法：Remote Write

Prometheus 支援 remote write receiver（啟動時加 `--web.enable-remote-write-receiver`）。5g-viz replay 模式在開始播放前，從 JSONL 中提取所有 metric event，以 **原始 timestamp** 透過 remote write API 批量寫入 Prometheus。

流程：

1. **檢查是否已回填**：向 Prometheus 查詢 `count(nwdaf_ground_truth_ul_bytes{session="<id>"})`。若有結果，代表該 session 已回填過，跳過步驟 2~4（log 提示 "session already backfilled, skipping"）。這避免重複回填的等待時間。
2. 讀取 `events.jsonl`，篩選出 metric 相關的 event types（`aggregated_slot`、`ml_inference`、`accuracy`、`retrain_trigger`、`model_swap`）。
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

## 6. 操作模式

### 6.1 Live 模式（預設）

正常的實驗即時監控，與目前行為一致，額外加上 JSONL recording。

```
啟動：./start.sh [-p profile]
行為：collector tail → parse → JSONL + WebSocket + Prometheus
前端：即時渲染 topology + log + Grafana（from=now-Nm, to=now）
DVR 控制列：顯示，Go Live 按鈕 disabled（已在 live）
```

### 6.2 Live DVR 模式

在 live 模式中，使用者按下 Pause 或拖曳時間軸，進入 DVR 模式。

```
觸發：按 Pause 或拖曳 timeline slider
行為：
  - 後端不變，繼續 collect / parse / 寫 JSONL / 更新 Prometheus
  - WebSocket 照常推送 event 到前端
  - 前端收到新 event 後存入 buffer，但不 dispatch 到 topology / log
Scrub：拖曳時顯示靜態快照（見 §8.2）
Play：從 scrub 位置按選定速度播放動畫（見 §8.3）
Go Live：一鍵跳回即時模式
  - 重建 topology 到最新狀態：
    方式一（簡單）：向 GET /api/state 拉取 server 端最新的 state snapshot，套用到 topology。
    此方式要求後端 state.py 擴充——除了 nf_status，也追蹤所有 node CSS class（retraining 等）。
    方式二（純前端）：用 buffer 中累積的 pending events 做一次快速 replay（只跑 add_class / remove_class，跳過動畫），算出最新 node state。
    建議使用方式一，因為 server state 永遠是權威的，不受前端 buffer 遺漏影響。
  - 清空 DVR 期間的 event log，用 buffer 中最近的 events 填入 log 面板
  - 恢復即時 dispatch（新 WebSocket event 直接 dispatch，不再 buffer）
  - Grafana 恢復 from=now-Nm, to=now, refresh=5s
```

### 6.3 Replay 模式

從磁碟載入過去的 session（本機或別人導出的）進行回放。

```
啟動：./start.sh --replay sessions/20260413T063000123
行為：
  - 不啟動 SSH collector
  - 讀取 events.jsonl 到記憶體
  - 回填 Prometheus（remote write）
  - 啟動 web server（FastAPI），提供 REST API（/api/events 等）
  - 不使用 WebSocket 推送（前端透過 /api/events fetch + 自行排程播放，見 §9「前端驅動」）
前端：
  - 自動進入 DVR paused 狀態（§8.0 狀態機中 LIVE / RESUMING 不可達）
  - 不建立 WebSocket 連線
  - Grafana from/to = session 時間範圍
  - 可 scrub、play、調速
```

---

## 7. 前端模式識別與 Session 資訊

前端需要知道目前處於哪種模式（live / replay），以及當前 session 的基本資訊。

### `/api/session-info` Endpoint

新增一個 endpoint，前端啟動時 fetch：

```json
// Live 模式
{
  "mode": "live",
  "session_id": "20260413T063000123",
  "start_time": "2026-04-13T14:30:00+08:00",
  "end_time": null
}

// Replay 模式
{
  "mode": "replay",
  "session_id": "20260413T063000123",
  "start_time": "2026-04-13T14:30:00+08:00",
  "end_time": "2026-04-13T15:45:12+08:00"
}
```

前端根據 `mode` 決定行為：

| 行為 | live | replay |
|------|------|--------|
| 建立 WebSocket 連線 | 是 | 否（改用 `/api/events` fetch） |
| DVR 控制列的 Go Live 按鈕 | enabled | disabled / hidden |
| Timeline slider max | 持續增長（以最新 event 時間更新） | 固定為 `end_time` |
| Grafana iframe URL | 含 `&var-session=<session_id>` | 同左 |
| 初始狀態 | live，收到第一個 event 後 slider 出現 | 自動進入 DVR paused 狀態 |

### Grafana Session Variable 傳遞

Grafana iframe URL 加上 `&var-session=<session_id>`，讓 dashboard 的 `$session` template variable 自動選到正確的 session：

```
${_grafanaBase}/d/${_dashboardUid}?orgId=1&kiosk&theme=dark&var-session=${sessionId}&from=...&to=...
```

### Replay 模式下的 Topology Config

Replay 模式下，`/api/topology-config` 改為讀取 session 目錄內的 `topology.yaml`（而非 `profiles/<profile>/`）。`main.py` 中 `_load_topo_config()` 需根據 `SESSION_MODE` 切換來源。

---

## 8. 前端 DVR 行為

### 8.0 前端狀態機

前端 DVR 有五個狀態，所有 UI 行為和 event dispatch 邏輯由當前狀態決定。

**狀態定義**：

| 狀態 | 說明 | WebSocket events | Topology 渲染 |
|------|------|-----------------|---------------|
| `LIVE` | 即時模式，正常顯示 | 直接 dispatch | 即時動畫 |
| `PAUSED` | 暫停在某時間點 | 存入 buffer，不 dispatch | 靜態快照 |
| `SCRUBBING` | 正在拖曳 slider | 存入 buffer，不 dispatch | 靜態快照（隨 slider 更新） |
| `PLAYING` | DVR 播放中 | 存入 buffer，不 dispatch | 按速度播放動畫 |
| `RESUMING` | Go Live 過渡中（拉 /api/state → 套用 → 切回 LIVE） | 存入 buffer | 等待 state 載入 |

**狀態轉移表**：

```
觸發事件                       當前狀態 → 新狀態
─────────────────────────────────────────────────
按 Pause                      LIVE → PAUSED
                              PLAYING → PAUSED（停在當前 play 位置）
拖曳 slider 開始               LIVE → SCRUBBING
                              PAUSED → SCRUBBING
                              PLAYING → SCRUBBING（先取消播放 timer）
拖曳 slider 放開               SCRUBBING → PAUSED（停在放開位置）
按 Play                       PAUSED → PLAYING（從 scrub 位置開始）
播放到 event list 末尾          PLAYING → PAUSED（停在最後位置）
Live 模式下播放追上 live 位置    PLAYING → LIVE
按 Go Live                    PAUSED → RESUMING → LIVE
                              PLAYING → RESUMING → LIVE
                              SCRUBBING → RESUMING → LIVE
改變速度                       任何狀態 → 不改變狀態（PLAYING 中改速度立即生效）
```

**競態處理規則**：

- **快速連點 Play/Pause**：每次狀態轉移前先取消上一個播放 timer（`clearTimeout` / `cancelAnimationFrame`），確保不會同時有兩個播放迴圈。
- **拖曳中按 Play**：拖曳 mousedown 會進入 SCRUBBING，mouseup 會回到 PAUSED。Play 只在 PAUSED 狀態下有效，SCRUBBING 中按 Play 無反應。
- **RESUMING 中的操作**：RESUMING 是短暫過渡（等 `/api/state` 回應，通常 < 100ms）。期間所有按鈕 disabled，防止競態。
- **Replay 模式**：初始狀態為 `PAUSED`（slider 在 session 起始位置）。`LIVE` 和 `RESUMING` 狀態不可達。

### 8.1 DVR 控制列

位於 header 下方，新增一列 UI：

```
[⏸ Pause] [▶ Play] [⏭ Live]    ◀■■■■■■■■■■■■■■■□──────────▶    14:32:05 / 14:45:12    [0.25x ▾]    Chart: [▼ 3 ▲] min  [↻]
                                  ↑ scrub slider                   ↑ 目前 / 結束時間       ↑ 速度選單   ↑ 時間窗口 spinner  ↑ 重設
```

元件：

| 元件 | 功能 |
|------|------|
| Pause | 暫停 topology / log 更新，開始 buffer 新 event |
| Play | 從 scrub 位置按選定速度播放 |
| Live | 跳回即時模式（replay 模式下 disabled / hidden） |
| Timeline slider | 拖曳定位到任意時間點 |
| 時間顯示 | 左邊 = scrub 目前位置，右邊 = session 結束時間（live 模式為「now」） |
| 速度選單 | 0.25x / 0.5x / 1x / 2x / 4x |
| Chart 時間窗口 | `<input type="number">` spinner，設定 Grafana 顯示的時間窗口寬度（分鐘）。預設 3，最小 1，最大 30。可直接輸入或用上下按鈕 ±1 調整 |
| ↻ Reset Chart | 重設 Grafana iframe 回到追蹤播放的視角。使用者在 Grafana 內手動放大觀察某段後，按此回歸 |

### 與現有 `↺ Live` 按鈕的關係

Header 目前有 `#live-controls`（minutes input + `↺ Live` 按鈕），用於重設 Grafana iframe 時間範圍。此功能被 DVR 控制列的 Grafana 同步和 Go Live 按鈕完全取代。**移除 `#live-controls`，改由 DVR 控制列統一管理。**

### Timeline slider 範圍

- **Live 模式**：`min` = session start time，`max` = 最新 event 時間。每收到新 WebSocket event 時更新 `max`。在 live 狀態（非 DVR）下 slider 自動跟隨 max（類似 YouTube 直播的紅色進度條）。
- **Replay 模式**：`min` = session start time，`max` = session end time。固定不變。

### 8.2 Scrub（拖曳）行為

使用者拖曳 timeline slider 時，前端需要快速顯示該時間點的 topology 狀態。

**渲染方式：靜態快照**

1. 從 event buffer（或 REST API）取得從 session 開始到 scrub 時間點的所有 events。
2. 重建 topology 狀態：
   - **Node class**：從頭 replay 所有影響 node state 的 events，算出 scrub 時間點的累積狀態。注意：node state 變更是由 `topology.yaml` 的 `event_reactions` 驅動的（`add_class`、`remove_class` action），不是硬寫在前端裡。**重建邏輯必須重用 `event_reactions` config**，而非另外寫一套 if/else。具體做法：遍歷 events，對每個 event 查 `EVENT_REACTIONS`，只執行 `add_class` / `remove_class` action（跳過 `flash_edge` 和 `pulse`），即可算出累積 node state。
   - **Edge**：取 scrub 時間點前後一個短窗口（例如 ±1 秒）內的 `flash_edge` reaction events，以靜態箭頭顯示（不做動畫，不設 setTimeout 移除），表示「這個時刻附近有這些信號在流動」。箭頭樣式（顏色、寬度）沿用 `EDGE_STYLE` config。
3. 不執行任何動畫（不呼叫 `pulse()`、不觸發 `flashEdge` 的 setTimeout），確保拖曳過程流暢。

**效能考量**：

- Node state 數量少（< 10 個 NF），從頭 replay 到任意時間點的運算成本很低。
- 可選優化：每 N 秒建一個 state checkpoint（snapshot），scrub 時只需從最近的 checkpoint replay，而非永遠從頭開始。初期可不做，觀察效能後決定是否需要。

### 前端 Event 來源與 Fallback

前端持有的 event 來源取決於模式和時機：

- **Live 模式**：前端從 WebSocket 連線起開始累積 events。如果使用者中途重新整理頁面，buffer 只有重連之後的 events。
- **Replay 模式**：前端啟動時一次 fetch 完整 event list。

**Scrub 時的 fallback 策略**：如果前端 buffer 不包含 scrub 目標時間範圍的 events（例如 live 模式下 buffer 起點晚於 scrub 目標），前端向 `GET /api/events?from=<session_start>&to=<scrub_time>` 補拉缺失的 events，合併到 buffer 中。補拉一次後 buffer 就完整了，後續 scrub 不需重複請求。

### 8.3 Play（播放）行為

按下 Play 後，從 scrub 位置開始，按照選定速度依序播放 events。

**播放邏輯**：

1. 以 scrub 位置為起點，取出後續的 events。
2. 計算相鄰 event 的時間差 `dt`。
3. 實際等待時間 = `dt / speed`（例如 0.25x 時 5 秒間隔變成 20 秒，讓使用者仔細觀察）。
4. 每個 event 正常 dispatch 到 `Topology.react()` 和 `appendLog()`，產生完整的動畫效果（edge flash、node pulse 等）。
5. Timeline slider 同步前進。

**速度選項**：

| 速度 | 等待倍率 | 適用場景 |
|------|----------|----------|
| 0.25x | ×4 | 慢動作，逐一觀察每個信號 |
| 0.5x | ×2 | 稍慢，抓重點 |
| 1x | ×1 | 原始速度 |
| 2x | ×0.5 | 快速瀏覽 |
| 4x | ×0.25 | 快轉尋找特定事件 |

### 8.4 Event Log 面板

DVR 模式下，event log 面板的行為：

- **Scrub 時**：清空目前 log，顯示 scrub 時間點之前最近 N 筆 events（例如最近 50 筆），讓使用者知道「剛剛發生了什麼」。
- **Play 時**：隨播放進度 append events（與目前 live 行為一致）。
- **Go Live 時**：清空 DVR 期間的 log，從 live buffer 中的最近事件接續。

### 8.5 Grafana 同步

前端維護 `_grafanaWindowMin` 變數（預設 3，最小 1，最大 30），由 DVR 控制列的 Chart 時間窗口 spinner 控制。Grafana iframe 維持 `&kiosk`（full kiosk），所有時間控制由 5g-viz DVR 控制列統一管理。

| 動作 | Grafana 行為 |
|------|-------------|
| 拖曳 slider 中 | 不更新（避免頻繁 iframe reload） |
| 拖曳結束 / Play 暫停 | 更新 iframe `from`/`to` 為以 scrub 時間為中心的窗口（半寬 = `_grafanaWindowMin` / 2），`var-session=<orig_id>` |
| Replay Play 開始 | 呼叫 `/api/replay/play` 取得本次播放專用的 `pseudo_session`，再切換到 pseudo-live 模式：`var-session=<pseudo_session>`，`from=now-{N}m&to=now&refresh=5s`（N = `_grafanaWindowMin`）。iframe reload 一次，之後不再更新（見 §15） |
| Replay Play 進行中 | 不更新 iframe——Grafana auto-refresh 自行追蹤（等同 live 體感） |
| Replay Play 暫停 | 切回 backfill 資料：`var-session=<orig_id>`，centered from/to 在暫停位置。iframe reload 一次 |
| Go Live（live 模式） | 恢復 `from=now-Nm&to=now&refresh=5s`，`var-session=<orig_id>` |
| Chart 時間窗口變更 | PLAYING 中：更新 iframe `from=now-{N}m`（reload 一次）。PAUSED / SCRUBBING 中：更新 centered 半寬 |
| ↻ Reset Chart | 重設 iframe src 為當前模式對應的 URL（使用者在 iframe 內手動放大後恢復） |

---

## 9. 後端 API

### 新增 Endpoints

| Method | Path | 用途 |
|--------|------|------|
| GET | `/api/session-info` | 回傳當前模式（live/replay）、session ID、start/end time（見 §7） |
| GET | `/api/sessions` | 列出所有可用 session（每個 session 的 meta.json 內容） |
| GET | `/api/events?session=<id>&from=<ts>&to=<ts>` | 取得指定 session 和時間範圍的 events |
| GET | `/api/state` | 回傳 server 端最新的完整狀態 snapshot（用於 Go Live 重建） |
| POST | `/api/replay/play` | 啟動 pseudo-live（pre-seed + emit loop），並回傳本次播放專用的 `pseudo_session`。Body：`{ from_time, speed, window }`（見 §15.5） |
| POST | `/api/replay/pause` | 停止指定 `pseudo_session` 的 pseudo-live emit loop。Body：`{ pseudo_session }`（見 §15.5） |
| POST | `/api/replay/speed` | 變速。Body：`{ pseudo_session, speed, current_time }`（見 §15.5） |

以上三個 `/api/replay/*` endpoint 僅在 replay 模式下可用，live 模式呼叫回傳 404。

### `/api/events` 說明

**參數**：

| 參數 | 必要 | 說明 |
|------|------|------|
| `session` | 否 | Session ID。Live 模式下省略 = 當前 session；replay 模式下省略 = 載入的 session。指定不存在的 session 回傳 404。 |
| `from` | 否 | ISO 8601 timestamp，篩選起點（inclusive）。省略 = session 起始。 |
| `to` | 否 | ISO 8601 timestamp，篩選終點（inclusive）。省略 = session 最新。 |
| `limit` | 否 | 最大回傳筆數，預設 50000。上限 50000（超過以 50000 計）。 |
| `offset` | 否 | 跳過前 N 筆，用於分頁。預設 0。 |

**回傳**：

```json
{
  "events": [ ... ],
  "total": 8234,
  "has_more": true
}
```

- `events`：JSON array，按 event `time` 升序排列，保證穩定排序（同 timestamp 的 events 保持 JSONL 中的原始順序）。
- `total`：符合 from/to 條件的 event 總數（不受 limit/offset 影響）。
- `has_more`：是否還有更多 events（`offset + len(events) < total`）。

**典型使用場景**：

- 前端初次 scrub 補拉：`GET /api/events`（無 from/to，拉全部）。一次實驗通常 < 50000 筆，一次拉完。
- 超長 session：先 `GET /api/events?limit=50000` 取前 50000 筆 + `total`，按需分頁。

**資料來源**：Live 模式下從記憶體 buffer 回應（快速）；replay 模式下從載入的 event list 回應。不直接讀 JSONL 檔案。

### `/api/state` 說明

回傳 server 端維護的完整狀態，用於 Go Live 時一次性重建 topology。目前 `state.py` 只追蹤 `nf_status`，需擴充為同時追蹤所有 node CSS class。

**Key 一律使用 topology.yaml 中的 canonical node ID**（例如 `nwdaf_mtlf`、`smf`），不使用 NF 名稱（`MTLF`、`SMF`）。這與前端 `window._cy.$('#nwdaf_mtlf')` 直接對應，不需要額外的 name → ID 轉換。

```json
{
  "type": "state_snapshot",
  "nf_status": {"smf": "up", "nwdaf": "up"},
  "node_classes": {
    "nwdaf_mtlf": ["retraining"],
    "nwdaf_anlf": [],
    "smf": ["up"],
    "consumer": [],
    "upf_ees": [],
    "upf_ees2": [],
    "adrf": []
  }
}
```

`node_classes` 包含**所有** topology.yaml 中定義的 node（不只是有 class 的）；沒有任何 managed class 的 node 對應空 array `[]`。

`state.apply_event()` 需要參照 `topology.yaml` 的 `event_reactions` 來更新 `node_classes`——跟前端 scrub 重建的邏輯一致，但在 server 端執行。注意 `event_reactions` 中的 `add_class` / `remove_class` action 可能使用 `nf_aliases`（例如 `{nf}` → `SMF` → `smf`），server 端需載入 `nf_aliases` 做同樣的解析。

**前端套用 snapshot 的流程（full replace）**：

1. 移除所有暫時 edge（由 `flashEdge` 產生的、帶 setTimeout 的 edge 全部 `cy.remove()`）。
2. 對每個 node，先清除所有 managed CSS class（`up`、`active`、`retraining` 等），再根據 `node_classes` 加回應有的 class。
3. 這確保從任何 DVR 狀態跳回 LIVE 時，topology 是乾淨且正確的。

### 記憶體 Buffer

- Live 模式下，除了寫 JSONL，也在記憶體中維護一份 event list。
- 用途：前端 scrub 時透過 `/api/events` 查詢，直接從記憶體回應，不需讀磁碟。
- 容量：無上限（一次實驗通常幾千到幾萬筆，記憶體消耗在合理範圍）。若日後發現記憶體壓力，可改為固定容量的 ring buffer，超過的部分 fallback 讀 JSONL。

### WebSocket 協議變更

目前 WebSocket 是純 server→client 單向推送。DVR 模式不需要改變 WebSocket 協議——前端的 pause / buffer / scrub 全在 client 端處理，後端無需知道前端是否在 DVR 模式。

### Replay 模式的前端驅動 vs. 後端驅動

**方案 A — 前端驅動（推薦）：**
- 前端啟動時一次性 fetch 所有 events（`/api/events?session=<id>`）。
- Scrub、play、speed 全在前端用 JavaScript timer 控制。
- 後端只需提供 REST API，不需要 replay 排程邏輯。
- 優點：簡單、前端控制力強、不需要 WebSocket 雙向通訊。
- 缺點：session 很大時一次 fetch 全部可能慢。可分段 fetch 緩解。

**方案 B — 後端驅動：**
- 前端透過 WebSocket 發送 play/pause/seek/speed 指令。
- 後端維護 playback state，按速度排程推送 event。
- 優點：前端邏輯輕。
- 缺點：需要雙向 WebSocket 協議、後端狀態管理複雜、多 client 時需獨立 playback state。

**選擇方案 A**：replay 和 live DVR 的 scrub/play 邏輯統一放在前端。Live DVR 時前端已經有記憶體中的 event buffer（WebSocket 收到的），scrub 時補查 `/api/events`；replay 時前端持有完整 event list。兩種模式可共用同一套 DVR UI 邏輯。

---

## 10. 導出與匯入

### 導出

```bash
# 打包 session
tar czf session-20260413T063000123.tar.gz -C 5g-viz/sessions 20260413T063000123/

# 檔案大小估計：
#   一次 30 分鐘實驗 ≈ 5,000~20,000 events ≈ 1~5 MB（gzip 後更小）
```

### 匯入

```bash
# 解壓到 sessions 目錄
tar xzf session-20260413T063000123.tar.gz -C 5g-viz/sessions/

# 以 replay 模式啟動
./start.sh --replay sessions/20260413T063000123
```

### 對方的環境需求

- 安裝好 5g-viz 及其 Python 依賴。
- Prometheus 運行中（需 `--web.enable-remote-write-receiver`）。
- Grafana 運行中（dashboard 會自動建立，因 `grafana_setup.py` 在啟動時執行）。
- **不需要** 5GC VM 或 SSH 連線（replay 模式不啟動 collector）。
- **不需要** 相同的 topology.yaml profile（replay 的 event 不依賴 topology config 做 re-parse；但前端渲染 topology 仍需對應的 config。可考慮在 meta.json 中記錄 profile 名稱，或把 topology.yaml 一起打包）。

### 注意事項：topology.yaml 相容性

Replay 的 events 是已 parse 過的結構化資料，不依賴 parser rules。但前端 `Topology.react()` 是由 `topology.yaml` 中的 `event_reactions` 驅動的——如果匯入方的 topology.yaml 與錄製方不同（例如缺少某些 event_reactions），部分 event 可能不會有 topology 動畫。

**建議**：導出時將 `profiles/<profile>/topology.yaml` 一起打包進 session 目錄。Replay 模式優先使用 session 內的 topology.yaml。

```
sessions/20260413T063000123/
  meta.json            ← 含 grafana_groups
  events.jsonl
  topology.yaml        ← 從 profiles/<profile>/ 複製
```

### Grafana Dashboard 可攜帶性

`meta.json` 記錄了 `grafana_groups`（來自錄製時的 `GRAFANA_GROUPS` 環境變數）。Replay 模式下，`grafana_setup.py` 使用 `meta.json` 中的 `grafana_groups` 來生成 dashboard panels，而非讀取匯入方的 `.env`。這確保即使匯入方的環境設定不同，Grafana dashboard 的 panel 數量和 group 分組都與原始錄製一致。

具體行為：
- **Live 模式**：`grafana_setup.setup()` 照舊從 `config.GRAFANA_GROUPS`（.env）讀取 groups。
- **Replay 模式**：`grafana_setup.setup()` 接受 groups 參數，由 `main.py` 從 `meta.json["grafana_groups"]` 傳入。

---

## 11. 啟動流程變更

### start.sh

```
./start.sh [-p profile]                           # live 模式（預設）
./start.sh --replay sessions/20260413T063000123      # replay 模式
```

**Live 模式**：

1. 啟動 Prometheus（加 `--web.enable-remote-write-receiver`）
2. 等 Prometheus ready
3. 啟動 5g-viz：`PROFILE=<profile> SESSION_MODE=live uv run uvicorn main:app ...`
4. `main.py` lifespan：
   - 產生 session ID
   - 建立 `sessions/<id>/` 目錄，寫 `meta.json`（start_time）
   - 複製 `topology.yaml` 到 session 目錄
   - 啟動 collector、queue processor、Grafana setup
   - SIGINT/SIGTERM 時更新 `meta.json`（end_time, event_count）

**Replay 模式**：

1. 啟動 Prometheus（同上）
2. 等 Prometheus ready
3. 啟動 5g-viz：`SESSION_MODE=replay SESSION_PATH=<path> uv run uvicorn main:app ...`
4. `main.py` lifespan：
   - 讀取 session 的 `meta.json` 和 `events.jsonl`
   - 執行 Prometheus remote write 回填
   - 執行 Grafana setup（確保 dashboard 存在）
   - 不啟動 collector
   - 將 events 載入記憶體，供 `/api/events` 查詢

### 移除 clear_metrics()

`grafana_setup.clear_metrics()` 不再呼叫。`main.py` lifespan 中移除相關程式碼。

---

## 12. 潛在問題與注意事項

### 12.1 JSONL 寫入效能

每個 event 執行一次 `write` + `flush`。5g-viz 的 event 頻率通常在每秒幾筆到幾十筆，I/O 不構成瓶頸。但 `flush` 確保 crash-safe，有少量 I/O overhead。若未來 event 頻率大幅增加，可改為定時 flush（例如每 100ms）或使用 buffered writer。

### 12.2 Prometheus remote write 依賴

Remote write payload 使用 protobuf + snappy 壓縮，這是 Prometheus 官方規格。Python 端需要：

- `protobuf`（或 `prometheus-remote-write` 套件）
- `python-snappy`（snappy 壓縮）

這些是額外依賴。替代方案：評估是否有輕量的 Python library 能處理 remote write。或者，如果只需要支援本機 Prometheus，可考慮直接操作 TSDB WAL，但這更複雜且脆弱。

另一個替代方案是 replay 時不走 remote write，而是啟動一個臨時的 `/metrics` endpoint，把 JSONL 中的 metric 值 expose 出來，讓 Prometheus 正常 scrape。但這樣 Prometheus 會以 scrape 當下的時間記錄，而非原始時間，導致 Grafana 曲線出現在錯誤的時間軸位置。因此 **remote write 是必要的**。

### 12.3 大型 Session 的記憶體消耗

Replay 模式一次載入整個 JSONL 到記憶體。一筆 event 約 200~500 bytes，10,000 筆 ≈ 2~5 MB，100,000 筆 ≈ 20~50 MB，在合理範圍。

若日後出現超長 session（例如跑一整天、百萬筆 event），需考慮：

- 前端分段 fetch（pagination），而非一次全拉。
- 後端使用 memory-mapped file 或 streaming 讀取。
- 這些優化初期不需要，可在遇到實際需求時再加。

### 12.4 時間戳一致性

目前 event 的 `time` 欄位來自 VM 上的 log 時間（logrus 的 `time="..."`）。這個時間取決於 VM 的系統時鐘。

- 如果 VM 和 host 的時鐘有偏差，JSONL 中的時間和 Prometheus scrape 時間會不一致。
- Live 模式下不影響（Prometheus 以自己的 scrape 時間記錄，Grafana 看的是 Prometheus 時間）。
- Replay 模式下，remote write 使用 JSONL 中的 event 時間（= VM 時間），Grafana 的 `from`/`to` 也設為 event 時間範圍，所以自洽。
- 潛在問題：如果 VM 時鐘在實驗過程中跳動（NTP sync），可能導致 event 時間不連續。這會影響 DVR timeline 的顯示，但發生機率低，暫不處理。

### 12.5 Scrub 時的 Edge 窗口

靜態快照模式下，需要決定 edge 顯示的時間窗口大小。窗口太小可能漏掉重要的 edge；窗口太大會同時顯示太多 edge 造成雜亂。

建議初始值為 ±1 秒，之後根據實際使用體驗調整。也可考慮讓使用者調整此參數。

### 12.6 Grafana iframe reload 延遲

更新 Grafana iframe 的 `src` 會觸發整個 iframe 重新載入，通常需要 1~3 秒。在 play 期間頻繁更新會造成閃爍。

緩解方式：

- Play 期間限制 Grafana 更新頻率（例如每 5 秒真實時間最多更新一次）。
- 考慮使用 Grafana 的 `postMessage` API 來更新時間範圍（而非替換 `src`），避免 full reload。需確認在 anonymous/kiosk 模式下是否可用。

#### 12.6.1 當前結論（2026-04-14 更新）

在目前「嵌入 Grafana iframe，透過更新 from/to 追時間」架構下，**無法完全重現 live 模式的連續渲染體感**。原因是只要變更時間範圍，Grafana 就必須重新 query/重繪；若透過重設 `iframe.src`，還會觸發整頁 reload。

解決方案：**Pseudo-Live Pipeline**（§15）。Replay 播放期間，後端按播放速度重新 set Prometheus gauge 值，Prometheus 以 scrape 時間（= now）記錄，Grafana 用 `now-Nm ~ now` + `refresh=5s` 運作，與 live 模式體感一致。PAUSED / SCRUBBING 時維持現行的 backfill 資料 + 固定 from/to。Play ↔ Pause 切換時各有一次 iframe reload（時間模式從固定切到 now-relative），但播放期間完全不動 iframe。

### 12.7 `state_snapshot` 在 DVR 模式下的行為

目前新 WebSocket client 連線時收到 `state_snapshot`（只有 `nf_status`）。DVR 模式下，這個 snapshot 反映的是 live 的最新狀態，但前端可能正在顯示歷史時間點的狀態。

處理方式：前端在 DVR 模式下忽略 `state_snapshot`（因為狀態由 event replay 重建），只在 Go Live 時才套用。

### 12.8 前端 Event Buffer 與記憶體

Live DVR 模式下，前端在 pause 期間持續收 WebSocket events 存入 buffer。如果 pause 很久（例如使用者離開去做別的事），buffer 會持續增長。

緩解方式：buffer 設定上限（例如 50,000 筆），超過時丟棄最舊的。或者 Go Live 時不 replay buffer 內容，而是向 `/api/events` 拉取最近的 state 重建。

### 12.9 失敗降級

| 失敗情境 | 影響範圍 | 降級行為 |
|----------|----------|----------|
| **Prometheus 未啟動或不可用** | Grafana 曲線無資料 | Live 模式：JSONL recording + topology 動畫正常運作，Grafana 區域顯示空圖。啟動時 log warning "Prometheus unreachable, metrics will not be available"。不 block 啟動流程。 |
| **Remote write 回填失敗**（replay） | Grafana 曲線缺失 | log warning "remote write failed: {error}, Grafana charts unavailable for this session"。繼續進入 replay 模式，topology + event log 回放正常。使用者可稍後重試 `--force-backfill`。 |
| **Session 目錄或 events.jsonl 損毀 / 缺失**（replay） | 無法啟動 replay | 啟動時印出明確錯誤訊息並 exit（例如 "events.jsonl not found in session directory"）。不 fallback 到 live 模式，避免混淆。 |
| **JSONL 寫入失敗**（live，磁碟滿等） | 本次 session 錄製不完整 | log error "JSONL write failed: {error}"。**將 `meta.json` 標記 `"corrupted": true`**，並在前端 DVR 控制列顯示警告（例如黃色標記 "recording incomplete"）。WebSocket broadcast + Prometheus metrics 繼續運作，確保當前實驗的 live 體驗不中斷。但此 session 的 replay / export 不再保證完整——DVR scrub 只能看到已成功寫入的部分，導出時打包的 events.jsonl 可能缺尾。每次寫入失敗都嘗試重試一次；若連續失敗超過 10 次，停止寫入（避免大量 error log），但 live pipeline 繼續。 |
| **/api/state 請求失敗**（Go Live） | 無法重建最新狀態 | Fallback 到方式二（從前端 buffer replay），log warning。若 buffer 也不完整，直接切回 LIVE 狀態並接受可能的短暫狀態不一致——下一個到來的 event 會逐步修正。 |

---

## 13. 驗收標準

### 效能目標

| 指標 | 目標 | 量測方式 |
|------|------|----------|
| Scrub 延遲（拖曳 slider 到 topology 更新） | < 200ms（10,000 events 以內的 session） | browser devtools performance profiling |
| Replay backfill 完成時間 | < 5s（10,000 metric events） | server log 計時 |
| Go Live 恢復延遲（RESUMING → LIVE） | < 500ms | 從按下 Go Live 到 topology 恢復即時更新 |
| JSONL 寫入延遲（per event） | < 1ms | 不影響 event broadcast 速度 |
| 前端 event fetch（/api/events 全量） | < 1s（10,000 events） | network tab response time |

### 最小 E2E 驗證流程

**Live DVR 流程**：

1. 啟動 `./start.sh`，確認 topology 正常顯示、Grafana 有曲線。
2. 等待至少 30 秒（累積足夠 events）。
3. 按 Pause → 確認 topology 停止更新，新 events 不出現在 log。
4. 拖曳 slider 到 10 秒前 → 確認 topology 顯示該時間點的 node state + 靜態 edge。
5. 按 Play（0.25x）→ 確認 edge 動畫以慢速播放，slider 緩慢前進。
6. 改速度到 2x → 確認加速。
7. 按 Go Live → 確認 topology 恢復即時、Grafana 恢復 live 時間範圍。
8. Ctrl+C → 確認 `sessions/<id>/meta.json` 有 `end_time`、`events.jsonl` 有內容。

**Replay 流程**：

1. `./start.sh --replay sessions/<id>`，確認前端自動進入 paused 狀態。
2. Grafana 顯示該 session 時間範圍的曲線（backfill 成功）。
3. 按 Play → 確認 topology 動畫正常。
4. Scrub 到任意時間點 → 確認靜態快照正確。
5. Go Live 按鈕 disabled / hidden。

**Replay Pseudo-Live 流程**：

1. `./start.sh --replay sessions/<id>`，等 backfill 完成。
2. Scrub 到 session 中間某位置，按 Play。
3. 確認 Grafana 立刻顯示完整曲線（pre-seed 成功），且曲線持續向右延伸（auto-refresh，無 iframe reload）。
4. 播放 30 秒後改速度（例如 1x → 4x）→ 確認 topology 和 Grafana 曲線都加速，無中斷。
5. 按 Pause → 確認 Grafana 切回 backfill 視角（原始時間軸，centered 在暫停位置）。
6. 再次按 Play → 確認 Grafana 再次以 pseudo-live 模式啟動，曲線平滑。
7. 調整 Chart 時間窗口（例如 3 → 1 分鐘）→ 確認 Grafana 視窗寬度變更。
8. 在 Grafana 圖表上手動放大某段 → 按 ↻ Reset Chart → 確認恢復追蹤播放視角。

**導出 / 匯入流程**：

1. 打包 session：`tar czf` session 目錄。
2. 在另一台機器（或清空 Prometheus 後）解壓。
3. `./start.sh --replay` → 確認 topology + Grafana 曲線都正常。

**崩潰復原驗證**：

1. Live 模式中 `kill -9` 強制終止 5g-viz process。
2. 確認 `events.jsonl` 存在且可讀（最多遺失最後一筆）。
3. 確認 `meta.json` 缺少 `end_time`（非正常關閉）。
4. `./start.sh --replay sessions/<id>` → 確認 replay 正常運作（end_time 從 JSONL 最後一筆推斷）。

---

## 14. 實作順序建議

1. [x] **Session recording + JSONL 寫入**：最基礎的改動，不影響現有功能。
2. [x] **Prometheus session label**：改 metrics 宣告、Grafana dashboard query。移除 `clear_metrics()`。
3. [x] **後端 API**（`/api/events`、`/api/sessions`）+ 記憶體 buffer。
4. [x] **前端 DVR 控制列 UI**：HTML/CSS layout。
5. [x] **前端 pause + buffer 邏輯**。
6. [x] **前端 scrub（靜態快照重建）**。
7. [x] **前端 play + 變速播放邏輯**。
8. [x] **前端 Grafana 同步**：scrub/paused 位置對齊視窗、play 期間節流同步、Go Live 回到即時視窗。
9. [x] **Replay 模式**：`start.sh --replay`、後端 replay 載入、前端 replay 入口與 Prometheus remote write 回填已完成（支援 `--force-backfill`）。
10. [ ] **導出打包**（含 topology.yaml）。
11. [ ] **Pseudo-live pipeline — 後端 MetricPlayer**：重用既有 metric family、每次 Play 產生唯一 `pseudo_session`、pre-seed remote write、async emit loop。
12. [ ] **Pseudo-live pipeline — Playback 控制 API**：`/api/replay/play`、`/api/replay/pause`、`/api/replay/speed`。
13. [ ] **Pseudo-live pipeline — 前端整合**：Play / Pause 時切換 Grafana 模式（backfill ↔ pseudo-live）、Chart 時間窗口 spinner、↻ Reset Chart 按鈕。

---

## 15. Pseudo-Live Pipeline（Replay Grafana 近似 Live 體驗）

### 15.1 問題與目標

Replay PLAYING 期間若透過更新 Grafana iframe 的 `from`/`to` 來追蹤播放位置，每次更新都觸發 iframe reload（1~3 秒），造成閃爍（見 §12.6）。目標是讓 replay 播放中的 Grafana 曲線體感等同 live——平滑、不閃爍、持續向右延伸。

PAUSED / SCRUBBING 則維持現行邏輯（backfill 資料 + 固定 from/to）。

### 15.2 核心機制

Live 模式下 Grafana 之所以順暢，是因為：

1. 後端即時 set Prometheus gauge 值。
2. Prometheus 每 5s scrape `/metrics`，以 scrape 當下的時間（= now）記錄。
3. Grafana 用 `now-Nm ~ now` + `refresh=5s` 自動追蹤，不需要改 iframe src。

Pseudo-live 將同樣的機制複製到 replay：播放時後端按播放速度 set gauge 值，Prometheus scrape 記錄為 now，Grafana 用相同的 `now-Nm ~ now` + `refresh=5s` 運作。

Prometheus TSDB 中會存在兩組不衝突的資料：

```
① backfill:     session="20260413T063000123"        timestamp = 原始時間
                 └─ 用途：PAUSED / SCRUBBING 時 Grafana 顯示

② pseudo-live:  session="_live_20260413T063000123__20260414T101530456Z"   timestamp = 播放當下的 now
                 └─ 用途：PLAYING 時 Grafana now-Nm ~ now 顯示（每次 Play 皆為新的 pseudo_session）
```

### 15.3 後端 MetricPlayer

新增 `MetricPlayer` class（建議放在獨立的 `metric_player.py`），管理 replay 播放期間的 metric 發射。

**職責**：

1. **Pre-seed**：play 開始時，從 scrub 位置往前取一段 metric events，透過 remote write 寫入 Prometheus，timestamp 映射到 now 的過去，確保 Grafana 切換到 `now-Nm ~ now` 時立刻看到完整曲線（見 §15.4）。
2. **Async emit loop**：play 開始後，從 scrub 位置依序 set gauge 值，等待時間 = event 間隔 / speed。
3. **暫停 / 變速**：回應前端的 playback 控制 API（見 §15.5）。

**Gauge 管理**：

- **重用現有的 Gauge / Counter family**（`rules/nwdaf.py` 中已註冊的 `_ground_truth_ul` 等）。`prometheus_client` 不允許同名 metric 重複註冊，所以不能建新的 Gauge 實例。Pseudo-live 只是對同一個 Gauge family 使用不同的 session label 值（`pseudo_session = "_live_<session_id>__<play_token>"`），這在 `prometheus_client` 中完全合法——同一個 Gauge family 可以有任意多組 label 值，每組都是獨立的 series。
- **`nwdaf_retrain_total`（Counter）**：同樣重用現有的 Counter family。Counter 對新 label set（`session="<pseudo_session>"`）從 0 開始。初始化時呼叫 `.inc()` N 次即可設到正確的累積值。播放期間遇到 `retrain_trigger` event 再 `.inc()`。不需要改成 Gauge，也不需要另建 metric name。
- **每次 Play 都產生新的 `pseudo_session`**：不要重用固定 `_live_<session_id>`。`.remove()` 只能阻止之後的 scrape，不會刪掉 Prometheus TSDB 中已經 pre-seed 的資料；因此要靠唯一 `pseudo_session` 把每次播放的資料隔離開來，避免多次 Play / Pause 互相污染。

**初始化**：play 從任意 scrub 位置開始時，先遍歷該時間前所有 metric events 算出累積狀態（各 gauge 的值、retrain counter 的累積次數），一次 set / inc 到對應的 label set 上：

```python
# Gauge：直接 .set()
_ground_truth_ul.labels(session="_live_abc__20260414T101530456Z", sub_id="imsi-001", target="group=1").set(1234.0)

# Counter：新 label set 從 0 開始，.inc() N 次
for _ in range(retrain_count_at_scrub_position):
    _retrain_total.labels(session="_live_abc__20260414T101530456Z").inc()
```

這確保 Prometheus 下一次 scrape 就能拿到正確值。

**Async emit loop**：

```python
async def _emit_loop(self):
    while self._playing and self._cursor < len(self._metric_events):
        event = self._metric_events[self._cursor]
        dt = event_time - prev_event_time
        await asyncio.sleep(dt / self._speed)
        self._set_gauge(event)          # set prometheus_client Gauge
        self._cursor += 1
    # 播放到結尾 → 停止
    self._playing = False
```

**生命週期**：

- Replay 模式啟動時建立 MetricPlayer 實例（持有所有 metric events，尚不開始播放）。
- 收到 play 請求 → 產生新的 `play_token` / `pseudo_session`，執行 pre-seed + 啟動 async loop。
- 收到 pause 請求或 event 播放完畢 → 停止 loop，並清除該 `pseudo_session` 對應的 exporter label set。
- 5g-viz 結束時清理。

### 15.4 Pre-seed（播放前預填）

#### 目的

Grafana 切換到 `now-Nm ~ now` 時，若 Prometheus 裡無資料，圖表會一片空白，要等數分鐘才填滿。Pre-seed 在 play 開始前一次性補入歷史資料，確保 Grafana 立刻顯示完整曲線。

#### 機制

Play 開始時，後端從 scrub 位置往前取 `W × S` 的 session 時間範圍內的 metric events（W = Grafana 視窗寬度，S = 播放速度），將 timestamp 映射到真實時間的過去，透過 remote write 寫入 Prometheus：

```
scrub 位置 = T_scrub
Grafana 視窗 = W（分鐘，由前端 Chart 時間窗口 spinner 控制）
播放速度 = S

pre-seed 範圍（session 時間）：T_scrub − W×S  ~  T_scrub
timestamp 映射：mapped_ts = now − (T_scrub − event_time) / S

例（W=3min, S=1x）：
  event at T_scrub − 2min  →  remote write at now − 2min
  event at T_scrub − 1min  →  remote write at now − 1min
  event at T_scrub         →  remote write at now
```

所有 pre-seed sample 使用本次播放專用的 `pseudo_session`。

#### 資料量

以每 5 秒一筆 metric event、3 分鐘視窗為例：約 36 × metric 種類 ≈ 幾百筆 sample。可重用 §5 的 remote write 編碼邏輯（`_encode_remote_write` / `_iter_series_batches`），應在 1 秒內完成。

#### 播放速度對 pre-seed 範圍的影響

| 速度 | 視窗 3min | pre-seed 的 session 時間範圍 |
|------|-----------|------------------------------|
| 0.25x | 3min | 45 秒 |
| 1x | 3min | 3 分鐘 |
| 4x | 3min | 12 分鐘 |

高速播放時涵蓋更多 session 時間，但 sample 數量仍可控（event 頻率固定，只是映射到更密的真實時間間隔）。

#### Retrain counter 的 pre-seed 策略

Grafana 的 retrain annotation query 使用 `idelta(nwdaf_retrain_total{...}[15s]) > 0`，需要 15s 視窗內有**至少 2 個資料點**且值有變化才會觸發。若 pre-seed 只在 `retrain_trigger` event 發生的時間點寫入 counter 值，兩次 retrain 之間可能超過 15s 沒有資料點，`idelta` 無法偵測到變化。

解法：pre-seed 時在**每個 timestamp**（不只是 retrain event）都寫入 retrain counter 的當前累積值。這樣 `idelta` 在沒有 retrain 的區間看到連續相同值（delta = 0，不觸發 annotation），在 retrain 發生的時間點看到值跳變（delta > 0，觸發 annotation）。

```
timestamp    retrain_total    idelta    annotation
mapped -60s       2              0        —
mapped -55s       2              0        —
mapped -50s       3              1        ✓ retrain
mapped -45s       3              0        —
mapped -40s       3              0        —
```

資料量影響可控：retrain counter 只有 1 個 label set（`session`），每個 timestamp 多 1 筆 sample。

#### Pre-seed 與 Prometheus scrape 的銜接

Pre-seed 最後一筆 timestamp ≈ now。MetricPlayer 接著 set gauge，Prometheus 下一次 scrape 在 0~5s 後記錄新值。兩者之間最多 5 秒間隙，在 Grafana 的分鐘級視窗中不可察覺。

### 15.5 Playback 控制 API

前端透過 REST API 通知後端播放狀態變更。這些 endpoint 僅在 replay 模式下可用，live 模式呼叫回傳 404。

| Method | Path | Body | 說明 |
|--------|------|------|------|
| POST | `/api/replay/play` | `{ from_time, speed, window }` | 產生新的 `pseudo_session`，執行 pre-seed + 啟動 emit loop |
| POST | `/api/replay/pause` | `{ pseudo_session }` | 停止指定 `pseudo_session` 的 emit loop，並清除對應 label set |
| POST | `/api/replay/speed` | `{ pseudo_session, speed, current_time }` | 變速 |

#### `/api/replay/play`

**參數**：

| 參數 | 類型 | 說明 |
|------|------|------|
| `from_time` | ISO 8601 | 播放起點（= 前端 scrub 位置） |
| `speed` | float | 播放速度（0.25 / 0.5 / 1 / 2 / 4） |
| `window` | int | Grafana 視窗寬度（分鐘），用於決定 pre-seed 範圍 |

**回傳**：

```json
{ "pseudo_session": "_live_20260413T063000123__20260414T101530456Z" }
```

**行為**：

1. 產生新的 `play_token`，組出本次播放專用的 `pseudo_session = "_live_<session_id>__<play_token>"`。
1. 從 `from_time` 定位到 metric events 中對應的 index。
2. 初始化 gauge 到 `from_time` 時的累積狀態。
3. 執行 pre-seed remote write（見 §15.4）。
4. 啟動 async emit loop。
5. 回傳 `pseudo_session`。前端收到後再切換 Grafana iframe，確保 Grafana reload 時 Prometheus 裡已有資料。

#### `/api/replay/pause`

**參數**：

| 參數 | 類型 | 說明 |
|------|------|------|
| `pseudo_session` | string | 本次播放專用的 pseudo-live session token |

**行為**：

1. 僅當 `pseudo_session` 等於目前 active 的 pseudo-live stream 時才處理；若收到舊 token，直接忽略或回傳 204，避免 race condition。
2. 取消 emit loop 的 `asyncio.sleep`，停止 gauge 發射。
3. **清除 pseudo-live label set**：對所有曾 set 過的 Gauge 呼叫 `.remove()`，從 `/metrics` 輸出中移除 `session="<pseudo_session>"` 的 label set。

```python
_ground_truth_ul.remove("_live_abc__20260414T101530456Z", "imsi-001", "group=1")
_ground_truth_dl.remove("_live_abc__20260414T101530456Z", "imsi-001", "group=1")
# … 對所有使用中的 (sub_id, target) 組合重複
_retrain_total.remove("_live_abc__20260414T101530456Z")
```

**為什麼需要 `.remove()`**：暫停後 Prometheus 仍持續每 5s scrape `/metrics`。若不清除，Prometheus 會以暫停瞬間的 gauge 值持續記錄 → 圖表上出現**水平直線**。呼叫 `.remove()` 後，Prometheus 在暫停期間 scrape 不到該 label set，TSDB 中不會產生新資料點。注意：`.remove()` **不會刪掉** TSDB 中已經 pre-seed 的舊資料，這也是每次 Play 都必須產生新 `pseudo_session` 的原因。

#### `/api/replay/speed`

**參數**：

| 參數 | 類型 | 說明 |
|------|------|------|
| `pseudo_session` | string | 本次播放專用的 pseudo-live session token |
| `speed` | float | 新的播放速度 |
| `current_time` | ISO 8601 | 前端當前播放位置（用於定位 event index） |

**行為**：

1. 僅當 `pseudo_session` 等於目前 active 的 pseudo-live stream 時才處理；若收到舊 token，直接忽略或回傳 204，避免上一輪 play 的延遲請求影響當前播放。
2. 更新 `self._speed`。
3. Cancel 目前正在 await 的 `asyncio.sleep`。
4. 從 `current_time` 定位到對應的 event index。
5. 計算到下一筆 event 的剩餘 session 時間差 `dt_remain`。
6. 新的等待時間 = `dt_remain / new_speed`。
7. 繼續 async loop。

前端呼叫時不等 response（fire-and-forget）——前端和後端獨立推進，微小 drift 不影響體感。不需要重做 pre-seed：已寫入 Prometheus 的資料是固定的真實時間 timestamp，速度改變只影響後續 emit 節奏。

### 15.6 前端與後端的同步模型

前端驅動 topology / log（`_tickPlayback()`），後端驅動 Prometheus metric（MetricPlayer），兩者從同一起點以同一速度**獨立推進**，不需要嚴格同步。

理由：

- Topology 動畫和 Grafana 曲線本來就是不同的渲染管道。
- Grafana 有 5s refresh 週期 + 分鐘級視窗，秒級的 drift 不影響體感。
- 避免前端和後端之間建立雙向即時通訊的複雜度。

```
┌──────────┐     POST /replay/play      ┌──────────────┐
│  Frontend │ ───────────────────────▶  │ MetricPlayer  │
│           │     POST /replay/pause    │  (async loop) │
│ topology  │ ───────────────────────▶  │               │
│ + log     │     POST /replay/speed    │  set gauge()  │
│ (JS timer)│ ───────────────────────▶  │       │       │
└──────────┘                            └───────┼───────┘
      │                                         │
      │ dispatch events                         ▼
      ▼                                  /metrics endpoint
  Cytoscape                                     │
  + event log                            Prometheus scrape
                                          (every 5s, ts=now)
                                                │
                                                ▼
                                          Grafana iframe
                                          now-Nm ~ now
                                          refresh=5s
                                          (不需 reload)
```

### 15.7 前端 Play / Pause / Speed 流程

#### Play

1. 呼叫 `POST /api/replay/play { from_time, speed, window }`。
2. 等待 response，取得本次播放專用的 `pseudo_session`（後端也已完成 pre-seed）。
3. 切換 Grafana iframe：`var-session=<pseudo_session>`，`from=now-{N}m&to=now&refresh=5s`。iframe reload 一次。
4. 狀態 → PLAYING，啟動 `_tickPlayback()`。

播放期間 Grafana 完全不需要更新——auto-refresh 自行追蹤最新資料。

#### Pause

1. 取消 `_tickPlayback()` timer，記錄暫停位置。
2. `POST /api/replay/pause { pseudo_session }`（fire-and-forget）。後端停止該 pseudo-live stream 的 emit loop，並呼叫 `.remove()` 清除對應 label set（見 §15.5）。
3. 切換 Grafana iframe 回 backfill：`var-session=<orig_id>`，centered from/to 在暫停位置。iframe reload 一次。
4. 狀態 → PAUSED。

#### 變速（PLAYING 中）

1. 前端更新 `_playSpeed`，取消並重啟 `_tickPlayback()`，更新 `Topology.setPlaybackSpeed()`。
2. `POST /api/replay/speed { pseudo_session, speed, current_time }`（fire-and-forget）。
3. 後端 MetricPlayer cancel 當前 sleep，用新速度繼續。
4. 不重做 pre-seed，不更新 Grafana iframe。

#### 播放到 event 結尾

1. 後端 MetricPlayer emit 完最後一筆 → 停止。
2. 前端 `_tickPlayback()` 的 `_playCursor >= _events.length` → 狀態 → PAUSED。
3. `POST /api/replay/pause { pseudo_session }` → Grafana 切回 backfill。

### 15.8 DVR 控制列新增元件

在 §8.1 的 DVR 控制列右側新增兩個元件：

**Chart 時間窗口 spinner**：`<input type="number">`，設定 Grafana 顯示的時間窗口寬度（分鐘）。預設 3，最小 1，最大 30。可直接輸入數值或用上下按鈕 ±1 調整。

前端維護 `_grafanaWindowMin` 變數。變更時：

- PLAYING 期間：更新 iframe src 的 `from=now-{N}m`（reload 一次），後續繼續 auto-refresh。
- PAUSED / SCRUBBING 期間：更新 centered window 半寬（`DVR_GRAFANA_HALF_WINDOW_MS = _grafanaWindowMin * 60 * 1000 / 2`）。
- 下次 play 時：`window` 參數傳入 `/api/replay/play`，影響 pre-seed 範圍。

**↻ Reset Chart**：重設 Grafana iframe 回到追蹤播放的視角。

- PLAYING 期間按下：重設 iframe src 為 `from=now-{N}m&to=now&refresh=5s`。
- PAUSED 期間按下：重設 iframe src 為 centered from/to。
- 用途：使用者在 Grafana 圖表上手動放大觀察某段後，按此回歸正常追蹤視角。

### 15.9 注意事項

#### 多次 Play / Pause 循環

每次 pause 時後端呼叫 `.remove()` 清除目前 active `pseudo_session` 的 exporter label set（見 §15.5），確保暫停期間 Prometheus 不會 scrape 到 stale gauge 值。下次 play 時重新產生新的 `pseudo_session`、重新初始化 gauge（`.set()` / `.inc()`）、重新 pre-seed、啟動新的 emit loop，Grafana 切到 `now-Nm ~ now` 後看到的是乾淨的新曲線。

關鍵點是：`.remove()` 只能阻止**之後**的 scrape，不會刪掉 Prometheus TSDB 中已經 pre-seed / scrape 的舊資料。因此每次 play 都必須使用新的 `pseudo_session`，把不同播放嘗試的資料隔離開來。Grafana `now-Nm ~ now` 只查當前 `pseudo_session`，舊的播放資料即使仍留在 TSDB 中，也完全不會混入目前畫面；retention 到期後自動清除即可。

#### Prometheus 空間回收

Pseudo-live 每次 Play 都會產生新的 `pseudo_session`，因此 Prometheus TSDB 中的 series / samples 會隨播放次數累積。這不會影響查詢正確性，因為 Grafana 只查當前 `pseudo_session`；但若使用者頻繁 play / pause / scrub，磁碟佔用會逐步增加。

**目前的回收機制**：

- Exporter 層：`pause` / `play end` 時呼叫 `.remove()`，只負責把當前 `pseudo_session` 從 `/metrics` 輸出中移除，避免後續 scrape 持續寫入假水平線。
- TSDB 層：已經寫入的 pseudo-live 資料仍會保留，依 Prometheus retention 自動過期。現有系統對一般 session 也採同樣策略（舊 session 留在 TSDB，預設 retention 15 天；見 §4）。

**建議的加強方案**：

1. **明確設定 TSDB retention**：在 `start.sh` 加 `--storage.tsdb.retention.time=<duration>`，避免 pseudo-live 累積太久。若系統仍需要保留較久的一般 replay/live session，應保守設定（例如 `3d`、`7d`），或優先搭配第 2 項只刪 pseudo-live series，而不要直接把全域 retention 壓到過短。
2. **主動刪除 pseudo-live series**：在 `pause` / `play end` 後，透過 Prometheus admin API 呼叫 `delete_series` 刪除 `session="<pseudo_session>"` 的所有 `nwdaf_*` series，之後再視需要呼叫 `clean_tombstones`。這能讓 pseudo-live 歷史資料更快從查詢結果中消失，但實際磁碟空間回收仍取決於 Prometheus compaction / tombstone 清理節奏，不是立即釋放。

**本 phase 建議**：

- 先實作唯一 `pseudo_session` + `.remove()`，確保正確性。
- 同步把 retention 明確設成可接受的值；若仍需要保留一般 session 查詢能力，優先不要用過短的全域 retention。
- `delete_series` / `clean_tombstones` 可列為進一步優化；若實測 pseudo-live 使用量不高，也可以先不做。

#### 播放速度對 Grafana 曲線密度的影響

Prometheus 每 5s scrape 一次，不受播放速度影響。

| 速度 | 5s 內經過的 session 時間 | Grafana 上的效果 |
|------|------------------------|-----------------|
| 0.25x | 1.25s | 曲線變化慢，幾乎每個 scrape 點看起來一樣 |
| 1x | 5s | 與 live 一致 |
| 4x | 20s | 多筆 event 壓在同一個 scrape 點，gauge 取最後一個值 |

高速播放時部分中間值會被「吞掉」（gauge 在兩次 scrape 之間被 set 多次，只有最後一次被記錄）。這與 live 模式下 event burst 的行為一致，是 Prometheus pull model 的固有特性，不影響趨勢判讀。

#### 播放中切換速度不需要重做 pre-seed

速度切換只影響 MetricPlayer emit loop 的 `asyncio.sleep` 間隔（見 §15.5 `/api/replay/speed`），不影響已寫入 Prometheus 的資料。原因：

1. **Pre-seed 資料的 timestamp 已固定**：pre-seed 寫入時 timestamp 已映射為真實時間（`now - offset`），與播放速度無關。速度改變後，這些資料點仍在 TSDB 中，Grafana `now-Nm ~ now` 視窗仍正常顯示。
2. **新速度的 emit 資料自然接續**：MetricPlayer 以新速度繼續 set gauge → Prometheus 以 5s 間隔 scrape → 新資料點自然接在已有資料後面。
3. **舊速度的資料自然淡出**：假設視窗寬度 3 分鐘，切速後最多 3 分鐘內舊速度產生的資料就會滾出視窗左邊界。這段過渡期曲線密度逐漸從舊速度切換到新速度，視覺上不會有跳變或閃爍。

因此速度切換的處理非常輕量：前端 fire-and-forget 呼叫 `/api/replay/speed`，後端更新 sleep 間隔，雙方繼續推進。不需要 reset chart、不需要 reload iframe、不需要重做 pre-seed。

#### Pre-seed remote write 失敗

若 remote write 失敗，log warning 後仍然啟動 MetricPlayer emit loop 並回傳 200。Grafana 會顯示空白圖表，但隨著 Prometheus scrape 累積新資料，曲線會逐漸出現。體感類似 live 模式剛啟動時的「曲線逐漸長出」——不理想但可用。使用者可按 Pause 再重新 Play 重試 pre-seed。
