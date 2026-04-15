# DVR Mode: Roadmap

本文件保留原始章節編號 `§15`。

## 15. 實作順序建議

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
11. [x] **Pseudo-live pipeline — 後端 MetricPlayer**：每次 Play 產生唯一 `pseudo_session`、pre-seed remote write、async emit loop，且播放中的新點也走 remote write。
12. [x] **Pseudo-live pipeline — Playback 控制 API**：`/api/replay/play`、`/api/replay/pause`、`/api/replay/speed`。
13. [x] **Pseudo-live pipeline — 前端整合核心**：Play / Pause 時切換 Grafana 模式（backfill ↔ pseudo-live）、pause/scrub 使用 trailing backfill 視窗。
14. [x] **Chart 時間窗口 spinner、↻ Reset Chart 按鈕**。
