# DVR Mode: Frontend Modes and API

本文件保留原始章節編號 `§6 ~ §9`。

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
[⏸ Pause] [▶ Play] [⏭ Live]    ◀■■■■■■■■■■■■■■■□──────────▶    14:32:05 / 14:45:12    [0.25x ▾]    [Chart: 3 min] [↻ Reset Chart]
                                  ↑ scrub slider                   ↑ 目前 / 結束時間       ↑ 速度選單   ↑ 已實作的圖表控制元件
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
| Chart 時間窗口 | `<input type="number">` spinner，設定 Grafana 顯示的時間窗口寬度（分鐘）。預設 3，最小 1，最大 15 |
| ↻ Reset Chart | 將 Chart window 恢復為預設 3 分鐘，並強制重新同步目前的 Grafana 視角 |

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

目前前端已支援可調 Chart window（1~15 分鐘）與 `↻ Reset Chart`。Grafana iframe 維持 `&kiosk`（full kiosk），所有時間控制由 5g-viz DVR 控制列統一管理。

| 動作 | Grafana 行為 |
|------|-------------|
| 拖曳 slider 中 | 不更新（避免頻繁 iframe reload） |
| 拖曳結束 / Play 暫停 | 更新 iframe `from`/`to` 為以 scrub 時間為右邊界的 trailing window，`var-session=<orig_id>` |
| Replay Play 開始 | 呼叫 `/api/replay/play` 取得本次播放專用的 `pseudo_session`，再切換到 pseudo-live 模式：`var-session=<pseudo_session>`，`from=now-{N}m&to=now&refresh=5s`（N = `_grafanaWindowMin`）。iframe reload 一次，之後不再更新（見 §14） |
| Replay Play 進行中 | 不更新 iframe——Grafana auto-refresh 自行追蹤（等同 live 體感） |
| Replay Play 暫停 | 切回 backfill 資料：`var-session=<orig_id>`，trailing from/to 在暫停位置。iframe reload 一次 |
| Go Live（live 模式） | 恢復 `from=now-Nm&to=now&refresh=5s`，`var-session=<orig_id>` |
| Chart 時間窗口變更 | live 模式下立即重算 `now-Nm ~ now` 或 trailing window；replay `PLAYING` 中會以目前 playhead 重新啟動 pseudo-live，讓 pre-seed 視窗與目前 window 一致 |
| ↻ Reset Chart | 將 Chart window 恢復為 3 分鐘，並強制重同步目前 session / mode 的 Grafana iframe |

---

## 9. 後端 API

### 新增 Endpoints

| Method | Path | 用途 |
|--------|------|------|
| GET | `/api/session-info` | 回傳當前模式（live/replay）、session ID、start/end time（見 §7） |
| GET | `/api/sessions` | 列出所有可用 session（每個 session 的 meta.json 內容） |
| GET | `/api/events?session=<id>&from=<ts>&to=<ts>` | 取得指定 session 和時間範圍的 events |
| GET | `/api/state` | 回傳 server 端最新的完整狀態 snapshot（用於 Go Live 重建） |
| POST | `/api/replay/play` | 啟動 pseudo-live（pre-seed + emit loop），並回傳本次播放專用的 `pseudo_session`。Body：`{ from_time, speed, window }`（見 §14.5） |
| POST | `/api/replay/pause` | 停止指定 `pseudo_session` 的 pseudo-live emit loop。Body：`{ pseudo_session }`（見 §14.5） |
| POST | `/api/replay/speed` | 變速。Body：`{ pseudo_session, speed, current_time }`（見 §14.5） |

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
