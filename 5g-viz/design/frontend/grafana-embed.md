# Grafana Embed

> Historical note: this document still describes the old iframe switching model with `orig_session` and `pseudo_session`. The current runtime keeps a single original session in replay and changes only the query time semantics during playback.

本文描述 `frontend/events.js` 目前如何把 Grafana dashboard 嵌進前端畫面，並隨 live / replay / DVR 狀態同步時間窗口。

## 1. 嵌入層的責任

前端 Grafana 嵌入層目前負責：

- 取得 dashboard base URL 與 UID
- 建立單一 iframe
- 根據目前 session、timeline 與播放狀態重算 iframe URL
- 在 replay 播放期間切換 `orig_session` 與 `pseudo_session`

圖表內容本身、Prometheus query 與 dashboard 結構不在這一層處理。

## 2. 設定來源

前端啟動時會先讀：

- `GET /api/grafana-config`
- `GET /api/session-info`

之後維護三個和 iframe 有關的識別值：

- `_grafanaBase`
- `_dashboardUid`
- `_grafanaSessionId`

`_grafanaSessionId` 預設等於目前 session 的 `session_id`，只有 replay 播放期間才會暫時切到 `pseudo_session`。

## 3. Iframe 生命週期

圖表容器是 `#charts`。`ensureChart()` 只會建立一次 iframe：

- 若 `_dashboardUid` 為空，不建立
- 若 `_chartMounted` 已為 `true`，不重複建立

這代表前端不會在每次狀態切換時重建 DOM，而是盡量只更新現有 iframe 的 `src`。

## 4. URL 組合方式

Grafana URL 由 `_grafanaUrl(from, to, refresh)` 組出，固定包含：

- `/d/<dashboardUid>`
- `orgId=1`
- `kiosk`
- `theme=dark`
- `var-session=<session>`

其中：

- `var-session` 由 `_grafanaSessionId` 決定
- `refresh=5s` 只在需要「追著 now 跑」的模式下打開
- `from` / `to` 可能是相對時間，也可能是絕對毫秒時間戳

主要參數可整理成：

| 參數 | 目前值 / 來源 | 用途 |
|---|---|---|
| 路徑 | `/d/<dashboardUid>` | 指向既有 dashboard UID |
| `orgId` | 固定 `1` | 指向 Grafana 組織 |
| `kiosk` | 固定帶上 | 隱藏 Grafana 頂部導覽列，讓 iframe 更像內嵌圖表 |
| `theme` | 固定 `dark` | 使用深色主題 |
| `var-session` | `_grafanaSessionId` | 指定 dashboard template variable `session` |
| `refresh` | `5s` 或省略 | 只在 live now-window / replay pseudo-live 下啟用自動刷新 |
| `from` | 相對或絕對時間 | 查詢起點 |
| `to` | 相對或絕對時間 | 查詢終點 |

## 5. 四種主要顯示模式

### Live 即時模式

當前端處於：

- `sessionMode = live`
- `dvrState = LIVE`

iframe 會使用：

```text
from=now-<window>m
to=now
refresh=5s
var-session=<orig_session>
```

### Live DVR 靜態模式

當 live 使用者已 paused / scrubbed 離開即時點時，iframe 改成 trailing absolute range：

```text
from=<timelinePos - window>
to=<timelinePos>
refresh=off
var-session=<orig_session>
```

### Replay Backfill 模式

replay 預設也是 trailing absolute range，但資料來源仍是原始錄製 session：

```text
from=<timelinePos - window>
to=<timelinePos>
refresh=off
var-session=<orig_session>
```

若 timeline 還沒建立出可用 trailing window，前端才退回整段 session range。

### Replay Pseudo-Live 模式

當 replay 進入播放狀態，且 pseudo-live 已啟動後，iframe 會切成：

```text
from=now-<window>m
to=now
refresh=5s
var-session=<pseudo_session>
```

從前端體感來看，這一段和 live Grafana 行為相同，但實際資料是 backend 重新 emit 的 pseudo-live metrics。

## 6. Chart Window 控制

前端目前把圖表窗口寬度視為 DVR 控制列的一部分：

- 預設 `3` 分鐘
- 最小 `1` 分鐘
- 最大 `15` 分鐘

### 一般變更

若只是 live 或 paused 狀態改變窗口寬度，前端直接重算 iframe URL。

### Replay 播放中變更

若 replay 正在 `PLAYING`，改變窗口寬度不只是 reload iframe，而是會：

1. 暫時進入 `RESUMING`
2. 停掉目前 pseudo-live
3. 以目前 playhead 與新 window 重新呼叫 `/api/replay/play`
4. 切回 `PLAYING`

這是因為 pseudo-live metrics 需要根據新的 trailing window 重新 pre-seed。

### Reset Chart

`Reset Chart` 只是把 window 重設為預設值 `3`，並強制同步一次目前模式對應的 iframe URL。

## 7. Reload 抑制

前端用 `_lastGrafanaUrl` 避免不必要的 iframe reload。

只有在：

- URL 確實改變
- 或呼叫端明確要求 `force = true`

時，才會真正改寫 `iframe.src`。

這對下列情境特別重要：

- live mode 持續收到事件時，不必每筆 event 都重設 iframe
- replay play / pause / chart reset 時，可以在真正切模式時才 reload

## 8. 目前限制

- iframe URL 目前固定寫死 `theme=dark` 與 `/d/<uid>` 路徑，沒有 per-profile 或 per-dashboard 的前端覆寫層
- 模式切換本質上仍靠 iframe reload，Grafana panel 內部互動狀態不會被保留
- `var-session` 被視為 dashboard 已存在的 template variable；若 dashboard schema 改變，前端不會自動檢查相容性
- 這層只控制時間窗與 session，不理解 panel 級別的縮放或 legend 狀態，因此 Grafana 內部互動不會回寫到 5g-viz 控制列
