# DVR Mode: Session Recording & Replay

本主題已由單一長文件拆分為多個子文件。子文件保留原始章節編號，方便沿用既有 `§X.Y` 引用。

## 0. 進度註記（2026-04-15）

- [x] Session recording + JSONL：已完成 live session 建立、`meta.json`/`events.jsonl` 寫入、結束時回填 `end_time` 與 `event_count`。
- [x] Prometheus session 隔離：`nwdaf_*` metrics 已加 `session` label，Grafana query 與 variable 已改為 session-aware。
- [x] 後端 API：已完成 `/api/session-info`、`/api/sessions`、`/api/events`、`/api/state`，可供 DVR 前端讀取資料。
- [x] Replay 啟動入口：`start.sh --replay <session_path>` 與後端 replay 載入流程已打通。
- [x] 前端 DVR 控制列與狀態機（Pause/Play/Scrub/Go Live）：已完成基礎狀態機與 live timeline 互動（含 replay 基礎入口）。
- [x] 前端 Grafana 同步：scrub/paused 對齊時間窗、replay 播放時切換 pseudo-live、Go Live 還原即時視窗。
- [x] Replay 的 Prometheus remote write backfill：已補齊 replay 啟動前 backfill、重複 session 檢查與 `--force-backfill`。
- [x] Replay 模式 Grafana「近似 live」核心流程：已完成 `MetricPlayer`、`/api/replay/*`、前端 backfill ↔ pseudo-live 切換，以及 replay 啟動時清理 managed Prometheus TSDB，避免舊樣本污染。
- [x] Chart 時間窗口 spinner、↻ Reset Chart：已完成可調 chart window（1~15 分鐘）、Reset Chart，以及 live/replay 暫停視角統一為 trailing window。
- [ ] 匯出/匯入流程與 E2E 驗收腳本尚未實作。

## 1. 目標

目前 5g-viz 只支援 live tail，畫面永遠顯示最新狀態，錯過的事件無法回頭看。本功能要達成：

1. **Live DVR**：實驗進行中可暫停、慢速回放、拖曳時間軸到任意時間點，也可一鍵跳回 live。
2. **Session recording**：每次實驗自動錄製到磁碟，事後可完整重播。
3. **可攜帶**：錄製檔案可打包給他人，在另一台機器上還原完整的 topology 動畫與 Grafana 曲線。

## 文件索引

- [recording-and-prometheus.md](recording-and-prometheus.md)
  - `§2 核心設計原則`
  - `§3 Session 管理`
  - `§4 Prometheus Session 隔離`
  - `§5 Replay 時的 Prometheus 回填`
- [frontend-and-api.md](frontend-and-api.md)
  - `§6 操作模式`
  - `§7 前端模式識別與 Session 資訊`
  - `§8 前端 DVR 行為`
  - `§9 後端 API`
- [runtime-and-validation.md](runtime-and-validation.md)
  - `§10 導出與匯入`
  - `§11 啟動流程變更`
  - `§12 潛在問題與注意事項`
  - `§13 驗收標準`
- [replay-pseudo-live.md](replay-pseudo-live.md)
  - `§14 Pseudo-Live Pipeline（Replay Grafana 近似 Live 體驗）`
- [roadmap.md](roadmap.md)
  - `§15 實作順序建議`

## 延伸議題（實作中發現）

以下幾項是在 DVR 實作過程中暴露出來的相關問題，但不視為當前 DVR 主流程 blocker，已抽離為獨立設計討論：

- [../grafana/chart-rendering.md](../grafana/chart-rendering.md)
  - Grafana 在小時間窗下的邊界裁切、右邊界缺線、over-fetch / dynamic epsilon 等時間窗渲染語意。
- [replay-pseudo-live-consistency.md](replay-pseudo-live-consistency.md)
  - replay `pause/backfill` 與 `play/pseudo-live` 的圖表一致性問題，以及 pseudo-live timestamp remap 帶來的數值穩定性風險。
- [../architecture/metric-event-modeling.md](../architecture/metric-event-modeling.md)
  - `retrain` annotation 目前沿用 counter 推導事件；後續若要改為獨立 event metric，將在此文件整理 live / replay 共用的 metrics 建模。
