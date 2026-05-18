# Grafana Embed

本文描述 `frontend/events.js` 目前如何把 Grafana dashboard 嵌進前端畫面，並隨 live / replay / DVR 狀態同步時間窗口。

## 1. 嵌入層的責任

前端 Grafana 嵌入層目前負責：

- 取得 dashboard base URL 與 UID
- 建立單一 iframe
- 根據目前 session、timeline 與 DVR 狀態重算 iframe URL
- 在 live、replay paused、replay playing 之間切換不同的 query 時間語意

它不負責：

- 生成 PromQL
- 建 dashboard schema
- 決定 replay backfill policy

## 2. 設定來源

前端啟動時會讀：

- `GET /api/grafana-config`
- `GET /api/session-info`

之後維護幾個關鍵值：

- `_grafanaBase`
- `_dashboardUid`
- `_grafanaSessionId`
- `_replayGrafanaMode`
- `_chartWindowMin`

注意：`_grafanaSessionId` 在目前系統中通常始終等於原始 session。replay 播放不再切到 `_live_*` pseudo session。

## 3. URL 組合方式

Grafana URL 由 `_grafanaUrl(from, to, refresh)` 組出，固定包含：

- `/d/<dashboardUid>`
- `orgId=1`
- `kiosk`
- `theme=dark`
- `var-session=<session>`

主要參數意義如下：

| 參數 | 來源 | 用途 |
|---|---|---|
| `var-session` | `_grafanaSessionId` | 指定 dashboard template variable `session` |
| `from` / `to` | 模式相關 | 絕對時間或 relative time range |
| `refresh` | `cfg.grafana.refresh` 或省略 | 是否自動刷新 |

## 4. 主要顯示模式

### Live 即時模式

條件：

- `sessionMode = live`
- `dvrState = LIVE`

iframe 會用：

```text
from=now-<window>m
to=now
refresh=on
var-session=<orig_session>
```

### Live DVR 靜態模式

live pause / scrub 時，iframe 會切成：

```text
from=<timelinePos - window>
to=<timelinePos>
refresh=off
var-session=<orig_session>
```

### Replay backfill 模式

replay paused / scrubbed 時，iframe 同樣使用：

```text
from=<timelinePos - window>
to=<timelinePos>
refresh=off
var-session=<orig_session>
```

### Replay historical-relative play 模式

replay `PLAYING` 時，iframe 會改成：

```text
from=now-<offset+window>
to=now-<offset>
refresh=on
var-session=<orig_session>
```

也就是：

- session 不變
- query time range 改成隨 `now` 滑動的歷史相對窗口

## 5. `Chart Window` 控制

前端把圖表窗口寬度視為 DVR 控制列的一部分：

- 預設 `3` 分鐘
- 最小 `1`
- 最大 `15`

一般情況下，改動 `Chart Window` 就是重算 iframe URL。

在 replay `PLAYING` 時，它的效果是：

- 重新計算 historical relative `from/to`
- 讓 iframe 重新同步到新的相對寬度

這仍可能導致可見的 reload，但不再需要重建 pseudo-live metric session。

## 6. Reload 抑制

前端用 `_lastGrafanaUrl` 避免不必要的 iframe reload。

只有在：

- URL 真的改變
- 或呼叫端明確要求 `force`

時，才會真正改寫 `iframe.src`。

這對下列情境很重要：

- live 持續收事件時，不需要每筆事件都重設 iframe
- replay `PLAYING` 時，只有 query 視窗真正改變才更新

## 7. 目前限制

- `theme=dark` 與 `/d/<uid>` 目前仍是固定組法
- iframe reload 仍會丟失 Grafana 內部互動狀態
- 這層只控制 session 與時間窗，不理解 panel 級別互動狀態
