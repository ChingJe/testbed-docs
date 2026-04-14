# DVR Mode: Replay Pseudo-Live Pipeline

本文件保留原始章節編號 `§14`。

## 14. Pseudo-Live Pipeline（Replay Grafana 近似 Live 體驗）

### 14.1 問題與目標

Replay PLAYING 期間若透過更新 Grafana iframe 的 `from`/`to` 來追蹤播放位置，每次更新都觸發 iframe reload（1~3 秒），造成閃爍（見 §12.6）。目標是讓 replay 播放中的 Grafana 曲線體感等同 live——平滑、不閃爍、持續向右延伸。

PAUSED / SCRUBBING 則維持現行邏輯（backfill 資料 + 固定 from/to）。

### 14.2 核心機制

Live 模式下 Grafana 之所以順暢，是因為：

1. 後端即時 set Prometheus gauge 值。
2. Prometheus 每 5s scrape `/metrics`，以 scrape 當下的時間（= now）記錄。
3. Grafana 用 `now-Nm ~ now` + `refresh=5s` 自動追蹤，不需要改 iframe src。

Replay pseudo-live 不再走 `/metrics` exporter，而是把歷史 metric event **重新映射到當前時間軸** 後，直接透過 Prometheus remote write 寫入 TSDB。Grafana 仍然用相同的 `now-Nm ~ now` + `refresh=5s` 相對時間視窗顯示，因此播放期間不需要更新 iframe src。

Prometheus TSDB 中會存在兩組不衝突的資料：

```
① backfill:     session="20260413T063000123"        timestamp = 原始時間
                 └─ 用途：PAUSED / SCRUBBING 時 Grafana 顯示

② pseudo-live:  session="_live_20260413T063000123__20260414T101530456Z"   timestamp = 映射後的 now-relative 時間
                 └─ 用途：PLAYING 時 Grafana now-Nm ~ now 顯示（每次 Play 皆為新的 pseudo_session）
```

### 14.3 後端 MetricPlayer

新增 `MetricPlayer` class（建議放在獨立的 `metric_player.py`），管理 replay 播放期間的 pseudo-live metric 發射。

**職責**：

1. **Pre-seed**：play 開始時，從 scrub 位置往前取一段 metric events，透過 remote write 寫入 Prometheus，timestamp 映射到 now 的過去，確保 Grafana 切換到 `now-Nm ~ now` 時立刻看到完整曲線（見 §14.4）。
2. **Async emit loop**：play 開始後，從 scrub 位置依序等待 `event 間隔 / speed`，將到期的 metric events 重新映射到目前 wall clock 時間，再透過 remote write 寫入 Prometheus。
3. **暫停 / 變速**：回應前端的 playback 控制 API（見 §14.5）。

**資料路徑**：

- **Replay pseudo-live 只走 remote write**。`MetricPlayer` 不再把 pseudo-live 資料寫進 `/metrics` exporter，避免同一個 `pseudo_session` 同時被 scrape / remote write 兩次。
- **每次 Play 都產生新的 `pseudo_session`**：例如 `_live_<session_id>__<play_token>`。這樣同一輪 replay 中多次 Play / Pause 互不干擾，Grafana 只需查當前的 `pseudo_session`。
- **Replay 啟動時清空 managed TSDB**：`start.sh --replay` 會先清 `~/prometheus/data` 再啟動 Prometheus，確保不會讀到先前版本遺留的 replay / scrape 樣本。

**初始化**：play 從任意 scrub 位置開始時，先遍歷 `from_time` 之前的 metric events，計算各 metric 在該時間點的累積狀態，再把這段歷史映射為 `now` 左側的 pre-seed remote-write samples。之後 async emit loop 只負責把 `from_time` 之後的新事件持續寫進同一個 `pseudo_session`。

**Async emit loop**：

```python
async def _emit_loop(self):
    while self._playing and self._cursor < len(self._metric_events):
        event = self._metric_events[self._cursor]
        dt = event_time - prev_event_time
        await asyncio.sleep(dt / self._speed)
        self._remote_write(event)       # remote write with mapped wall-clock timestamp
        self._cursor += 1
    # 播放到結尾 → 停止
    self._playing = False
```

**生命週期**：

- Replay 模式啟動時建立 MetricPlayer 實例（持有所有 metric events，尚不開始播放）。
- 收到 play 請求 → 產生新的 `play_token` / `pseudo_session`，執行 pre-seed + 啟動 async loop。
- 收到 pause 請求或 event 播放完畢 → 停止 loop，不再為該 `pseudo_session` 產生新的 remote-write sample。
- 5g-viz 結束時清理。

### 14.4 Pre-seed（播放前預填）

#### 目的

Grafana 切換到 `now-Nm ~ now` 時，若 Prometheus 裡無資料，圖表會一片空白，要等數分鐘才填滿。Pre-seed 在 play 開始前一次性補入歷史資料，確保 Grafana 立刻顯示完整曲線。

#### 機制

Play 開始時，後端從 scrub 位置往前取 `W × S` 的 session 時間範圍內的 metric events（W = Grafana 視窗寬度，S = 播放速度），將 timestamp 映射到真實時間的過去，透過 remote write 寫入 Prometheus：

```
scrub 位置 = T_scrub
Grafana 視窗 = W（分鐘；目前固定 3，後續由前端 Chart 時間窗口 spinner 控制）
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

Pre-seed 最後一筆 timestamp ≈ now。MetricPlayer 接著沿相同 `pseudo_session` 持續 remote write 後續 sample；Grafana 只需維持 `now-Nm ~ now` + auto-refresh，即可看到曲線向右延伸，不需要再靠 scrape 或 iframe reload 銜接。

### 14.5 Playback 控制 API

前端透過 REST API 通知後端播放狀態變更。這些 endpoint 僅在 replay 模式下可用，live 模式呼叫回傳 404。

| Method | Path | Body | 說明 |
|--------|------|------|------|
| POST | `/api/replay/play` | `{ from_time, speed, window }` | 產生新的 `pseudo_session`，執行 pre-seed + 啟動 emit loop |
| POST | `/api/replay/pause` | `{ pseudo_session }` | 停止指定 `pseudo_session` 的 emit loop |
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
3. 執行 pre-seed remote write（見 §14.4）。
4. 啟動 async emit loop。
5. 回傳 `pseudo_session`。前端收到後再切換 Grafana iframe，確保 Grafana reload 時 Prometheus 裡已有資料。

#### `/api/replay/pause`

**參數**：

| 參數 | 類型 | 說明 |
|------|------|------|
| `pseudo_session` | string | 本次播放專用的 pseudo-live session token |

**行為**：

1. 僅當 `pseudo_session` 等於目前 active 的 pseudo-live stream 時才處理；若收到舊 token，直接忽略或回傳 204，避免 race condition。
2. 取消 emit loop 的 `asyncio.sleep`，停止為該 `pseudo_session` 產生新的 remote-write sample。
3. 前端切回原始 session 的 backfill 視窗；舊的 `pseudo_session` 樣本留在當前 replay run 的 TSDB 中，但不再被 Grafana 查詢。

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

### 14.6 前端與後端的同步模型

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
│ + log     │     POST /replay/speed    │ remote write  │
│ (JS timer)│ ───────────────────────▶  │       │       │
└──────────┘                            └───────┼───────┘
      │                                         │
      │ dispatch events                         ▼
      ▼                                  remote write batches
  Cytoscape                                     │
  + event log                            Prometheus TSDB
                                                │
                                                ▼
                                          Grafana iframe
                                          now-Nm ~ now
                                          refresh=5s
                                          (不需 reload)
```

### 14.7 前端 Play / Pause / Speed 流程

#### Play

1. 呼叫 `POST /api/replay/play { from_time, speed, window }`。
2. 等待 response，取得本次播放專用的 `pseudo_session`（後端也已完成 pre-seed）。
3. 切換 Grafana iframe：`var-session=<pseudo_session>`，`from=now-{N}m&to=now&refresh=5s`。iframe reload 一次。
4. 狀態 → PLAYING，啟動 `_tickPlayback()`。

播放期間 Grafana 完全不需要更新——auto-refresh 自行追蹤最新資料。

#### Pause

1. 取消 `_tickPlayback()` timer，記錄暫停位置。
2. `POST /api/replay/pause { pseudo_session }`（fire-and-forget）。後端停止該 pseudo-live stream 的 emit loop（見 §14.5）。
3. 切換 Grafana iframe 回 backfill：`var-session=<orig_id>`，trailing from/to 停在暫停位置（顯示「到該時刻為止」的過去 N 分鐘）。iframe reload 一次。
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

### 14.8 DVR 控制列新增元件

本節是後續 UI 增補，**目前尚未實作**。當前版本仍使用固定 3 分鐘視窗，且沒有 ↻ Reset Chart 按鈕。

在 §8.1 的 DVR 控制列右側新增兩個元件：

**Chart 時間窗口 spinner**：`<input type="number">`，設定 Grafana 顯示的時間窗口寬度（分鐘）。預設 3，最小 1，最大 30。可直接輸入數值或用上下按鈕 ±1 調整。

前端維護 `_grafanaWindowMin` 變數。變更時：

- PLAYING 期間：更新 iframe src 的 `from=now-{N}m`（reload 一次），後續繼續 auto-refresh。
- PAUSED / SCRUBBING 期間：更新 trailing window 寬度（`from = scrub_time - _grafanaWindowMin * 60 * 1000`，`to = scrub_time`）。
- 下次 play 時：`window` 參數傳入 `/api/replay/play`，影響 pre-seed 範圍。

**↻ Reset Chart**：重設 Grafana iframe 回到追蹤播放的視角。

- PLAYING 期間按下：重設 iframe src 為 `from=now-{N}m&to=now&refresh=5s`。
- PAUSED 期間按下：重設 iframe src 為 trailing from/to。
- 用途：使用者在 Grafana 圖表上手動放大觀察某段後，按此回歸正常追蹤視角。

### 14.9 注意事項

#### 多次 Play / Pause 循環

每次 play 都會產生新的 `pseudo_session`，pause / play end 只會停止該 stream 的 emit loop，不再為它產生新的 remote-write sample。Grafana `now-Nm ~ now` 只查當前 `pseudo_session`，因此同一輪 replay 中先前播放留下的 pseudo-live 樣本不會混入目前畫面。

為了避免「跨 replay 啟動」讀到舊版本遺留的 replay / scrape 樣本，`start.sh --replay` 目前會先清空 managed Prometheus TSDB（`~/prometheus/data`）再啟動。這讓 replay 驗證保持可重現，也解掉了先前同一個 model 出現雙線的污染問題。

#### Prometheus 空間回收

Pseudo-live 每次 Play 都會產生新的 `pseudo_session`，因此 Prometheus TSDB 中的 series / samples 會隨播放次數累積。這不會影響查詢正確性，因為 Grafana 只查當前 `pseudo_session`；但若使用者頻繁 play / pause / scrub，磁碟佔用會逐步增加。

**目前的回收機制**：

- Replay run 之間：`start.sh --replay` 會先清空 managed TSDB，因此上一輪 replay 的 pseudo-live 資料不會帶到下一輪。
- 同一輪 replay 之內：舊的 `pseudo_session` 樣本仍保留在 TSDB，但 Grafana 只查當前 `pseudo_session`，不影響畫面正確性。

**建議的加強方案**：

1. **明確設定 TSDB retention**：在 `start.sh` 加 `--storage.tsdb.retention.time=<duration>`，避免 pseudo-live 累積太久。若系統仍需要保留較久的一般 replay/live session，應保守設定（例如 `3d`、`7d`），或優先搭配第 2 項只刪 pseudo-live series，而不要直接把全域 retention 壓到過短。
2. **主動刪除 pseudo-live series**：在 `pause` / `play end` 後，透過 Prometheus admin API 呼叫 `delete_series` 刪除 `session="<pseudo_session>"` 的所有 `nwdaf_*` series，之後再視需要呼叫 `clean_tombstones`。這能讓 pseudo-live 歷史資料更快從查詢結果中消失，但實際磁碟空間回收仍取決於 Prometheus compaction / tombstone 清理節奏，不是立即釋放。

**本 phase 現況**：

- 已採用「唯一 `pseudo_session` + replay 啟動清 TSDB」的保守策略，優先確保 replay 驗證結果乾淨、可重現。
- 若未來希望保留 replay run 之間的 Prometheus 歷史，再重新評估 retention 或 `delete_series` / `clean_tombstones`。

#### 播放速度對 Grafana 曲線密度的影響

倍速在後端是有效的：MetricPlayer 會用 `event 間隔 / speed` 調整 emit loop 的等待時間，並以當下 wall clock 時間寫入新的 pseudo-live sample。

但 Grafana 上的體感不一定明顯，原因有兩個：

1. 圖表時間軸固定是 `now-Nm ~ now`，不會像播放器一樣「整張圖變快」。
2. Grafana 仍以 `refresh=5s` 更新，且 query step 可能高於 5 秒；因此倍速變化會被圖表刷新週期與下採樣稀釋。

因此目前倍速主要影響 topology / log 與 pseudo-live 資料推進節奏；Grafana 上僅呈現「資料更快進入目前視窗」，而非強烈的播放器式加速感。這屬於 Grafana 的工具限制，不視為本 phase blocker。

#### 播放中切換速度不需要重做 pre-seed

速度切換只影響 MetricPlayer emit loop 的 `asyncio.sleep` 間隔（見 §14.5 `/api/replay/speed`），不影響已寫入 Prometheus 的資料。原因：

1. **Pre-seed 資料的 timestamp 已固定**：pre-seed 寫入時 timestamp 已映射為真實時間（`now - offset`），與播放速度無關。速度改變後，這些資料點仍在 TSDB 中，Grafana `now-Nm ~ now` 視窗仍正常顯示。
2. **新速度的 emit 資料自然接續**：MetricPlayer 以新速度繼續 remote write 新 sample，新的 pseudo-live 資料點自然接在已有資料後面。
3. **舊速度的資料自然淡出**：假設視窗寬度 3 分鐘，切速後最多 3 分鐘內舊速度產生的資料就會滾出視窗左邊界。這段過渡期曲線密度逐漸從舊速度切換到新速度，視覺上不會有跳變或閃爍。

因此速度切換的處理非常輕量：前端 fire-and-forget 呼叫 `/api/replay/speed`，後端更新 sleep 間隔，雙方繼續推進。不需要 reset chart、不需要 reload iframe、不需要重做 pre-seed。

#### Pre-seed remote write 失敗

若 remote write 失敗，log warning 後仍然啟動 MetricPlayer emit loop 並回傳 200。Grafana 會顯示空白圖表，但隨著 emit loop 持續 remote write 新資料，曲線會逐漸出現。體感類似 live 模式剛啟動時的「曲線逐漸長出」——不理想但可用。使用者可按 Pause 再重新 Play 重試 pre-seed。

---

