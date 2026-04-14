# DVR Mode: Runtime, Risks, and Validation

本文件保留原始章節編號 `§10 ~ §13`。

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

1. 清空 managed Prometheus TSDB 目錄（`~/prometheus/data`），避免舊 replay / scrape 樣本污染本次驗證。
2. 啟動 Prometheus（同上）
3. 等 Prometheus ready
4. 啟動 5g-viz：`SESSION_MODE=replay SESSION_PATH=<path> uv run uvicorn main:app ...`
4. `main.py` lifespan：
   - 讀取 session 的 `meta.json` 和 `events.jsonl`
   - 執行 Prometheus remote write 回填
   - 建立 `MetricPlayer`（負責 replay pseudo-live 的 pre-seed / emit loop）
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

解決方案：**Pseudo-Live Pipeline**（§14）。Replay 播放期間，後端將歷史 metric event 的 timestamp 映射到 now，透過 Prometheus remote write 寫入本次播放專用的 `pseudo_session`；Grafana 固定用 `now-Nm ~ now` + `refresh=5s` 查這組 pseudo-live 資料。PAUSED / SCRUBBING 時維持原始 session 的 backfill 資料 + 固定 from/to。Play ↔ Pause 切換時各有一次 iframe reload（時間模式從固定切到 now-relative），但播放期間完全不動 iframe。

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
4. 播放 30 秒後改速度（例如 1x → 4x）→ 確認 topology 播放速度改變，Grafana 持續更新且不中斷（倍速體感不一定明顯，見 §14.9）。
5. 按 Pause → 確認 Grafana 切回 backfill 視角（原始時間軸，trailing 到暫停位置）。
6. 再次按 Play → 確認 Grafana 再次以 pseudo-live 模式啟動，曲線平滑。
7. `Chart 時間窗口 spinner` 與 `↻ Reset Chart` 目前尚未實作，留待後續 UI 增補驗收。

**導出 / 匯入流程**：

1. 打包 session：`tar czf` session 目錄。
2. 在另一台機器（或清空 Prometheus 後）解壓。
3. `./start.sh --replay` → 確認 topology + Grafana 曲線都正常。

**崩潰復原驗證**：

1. Live 模式中 `kill -9` 強制終止 5g-viz process。
2. 確認 `events.jsonl` 存在且可讀（最多遺失最後一筆）。
3. 確認 `meta.json` 缺少 `end_time`（非正常關閉）。
4. `./start.sh --replay sessions/<id>` → 確認 replay 正常運作（end_time 從 JSONL 最後一筆推斷）。

