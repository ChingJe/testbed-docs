# Events And DVR

本文描述 `frontend/events.js` 目前如何管理 session、事件來源、timeline 與 DVR 控制。

## 1. 這層的責任

`events.js` 目前是前端控制平面，負責：

- 啟動時取得 session 與 Grafana 設定
- 決定 live / replay 兩種前端模式
- 維護 `_events` 緩衝、timeline 與 event log
- 控制 `LIVE / PAUSED / SCRUBBING / PLAYING / RESUMING` 狀態切換
- 在 replay 播放期間協調 pseudo-live Grafana session

它不直接解讀拓樸設定；所有拓樸操作都透過 `window.Topology` 完成。

## 2. Bootstrap 順序

前端 bootstrap 會先等待兩類資訊：

- `TopologyReady`
- `GET /api/grafana-config` 與 `GET /api/session-info`

之後依 `session-info.mode` 分兩條路徑：

### live

- 初始 DVR 狀態設為 `LIVE`
- 建立 `/ws` 連線
- timeline 右邊界會隨新事件持續向前推進

### replay

- 初始 DVR 狀態設為 `PAUSED`
- 先透過 `GET /api/events` 把該 session 的 events 全部載入前端記憶體
- 不建立 WebSocket
- timeline 初始位置停在 session 起點

## 3. DVR 狀態機

前端目前使用五個顯式狀態：

| 狀態 | 說明 |
|---|---|
| `LIVE` | 即時顯示新事件 |
| `PAUSED` | 停在某個時間點，不再即時 dispatch |
| `SCRUBBING` | 正在拖曳 timeline |
| `PLAYING` | 依事件時間差重播 |
| `RESUMING` | 正在切換回 live 或重啟 replay pseudo-live |

控制列按鈕的 enabled / disabled 也完全由這個狀態機決定。

可以把常見轉移簡化成：

```text
LIVE --pause--> PAUSED --play--> PLAYING
  ^                ^               |
  |                |               |
  +----go-live-----+-----pause-----+

PAUSED --drag--> SCRUBBING --release--> PAUSED
```

## 4. 事件緩衝與時間軸

所有非 `state_snapshot` 事件都會進 `_events`。前端維護的不是單純 append-only view，而是一個可排序的事件緩衝。

### 緩衝特性

- `_tryAddEvent()` 會先記錄事件，再視情況把 `_eventsSorted` 標成 dirty
- `_ensureSorted()` 在需要時才做排序
- `_timelineMinMs` / `_timelineMaxMs` 會綁定 session 邊界或目前事件範圍

### timeline 來源

- live：左邊界優先用 `session_start`，右邊界取目前最後事件時間與 `Date.now()` 的較大值
- replay：左邊界取 `start_time`，右邊界優先取 `end_time`

時間顯示格式固定為：

```text
HH:MM:SS.mmm
```

### 緩衝事件何時會被消費

live mode 下，即使使用者已離開即時視角，前端仍會持續從 WebSocket 收事件並放進 `_events`。這些資料不是立刻丟棄，而是延後在下列時機使用：

- `LIVE`：新事件一進 `_events` 就立刻 `dispatch(event)`，直接更新 topology 與 event log
- `PAUSED` / `SCRUBBING`：`Topology.renderStaticSnapshot(...)` 會從 `_events` 重建指定時間點的 node class、edge 與 pulse；event log 也會改由 `_renderLogTailAt(...)` 從 `_events` 取該時間點之前的 tail
- `PLAYING`：前端以 `_findFirstIndexAfter(_timelinePosMs)` 從 `_events` 找播放起點，再用 `_tickPlayback()` 逐筆重播
- `Go Live`：若 `/api/state` 成功，前端直接套用 backend 權威 snapshot；若失敗，才退回用 `_events` 在最新時間點做一次靜態重建；此外 recent live log 也是從 `_events` 取回最近一段 tail

因此 `_events` 不只是 replay 專用緩衝，而是 live DVR 的核心資料池。它讓前端能在不中斷後端 live pipeline 的情況下，隨時切到歷史視角再切回即時視角。

## 5. Live 模式的資料來源

live mode 主要依賴 `/ws`。

### 連線建立後

server 會先推一份 `state_snapshot`。前端目前只在下列情況套用它：

- `LIVE`
- `RESUMING`

若使用者正在 scrub 或 paused，收到新的 `state_snapshot` 不會直接覆蓋目前畫面。

### 一般事件

非 `state_snapshot` 事件會：

1. 進入 `_events`
2. 更新 timeline 邊界
3. 僅在目前狀態是 `LIVE` 時才立即 `dispatch(event)`

因此在 paused / playing / scrubbing 期間，live 事件會持續累積，但不會立刻改變使用者正在看的時間點。

### 中斷重連

WebSocket 關閉後，只有 live mode 會自動在 3 秒後重連。replay mode 不使用這條路徑。

## 6. Replay 與 Live DVR 的主要行為

### Pause

按下 Pause 時：

- live / replay 都會進入 `PAUSED`
- replay 若正在 pseudo-live，會先呼叫 `/api/replay/pause`
- 前端會把目前時間點凍結成靜態快照，並重畫最近的 event log tail

### Scrub

拖曳 timeline 時會進入 `SCRUBBING`。前端為了避免每次 pointer move 都完整同步 DOM，會用 `requestAnimationFrame` 批次執行：

- `Topology.renderStaticSnapshot(...)`
- `_renderLogTailAt(targetMs)`

在 live mode，如果目前前端 buffer 不包含目標時間點以前的完整歷史，還會額外向：

```text
GET /api/events?session=<session_id>&from=<session_start>&to=<target>
```

補拉缺失事件。

這條補拉路徑目前還有一個 single-flight 鎖：`_historyFetchPromise`。只要前一次補拉尚未完成，新的 scrub 補抓請求就會直接等待同一個 promise，而不會重複發出第二條 `/api/events` backfill。

### Play

從 `PAUSED` 進入 `PLAYING` 時：

1. 取消 scrub render
2. 清掉暫時性 topology 效果
3. 以 `_findFirstIndexAfter(_timelinePosMs)` 找到播放起點
4. 依相鄰事件時間差與 `_playSpeed` 安排 `setTimeout`

live 與 replay 的差異在於：

- live：播放追到尾端後自動切回 `LIVE`
- replay：播放前必須先成功啟動 pseudo-live，播放到尾端後停在 `PAUSED`

### Go Live

`Go Live` 只在 live mode 出現。它會：

1. 對 `/api/state` 取最新 `state_snapshot`
2. 套用 `Topology.applySnapshot(snapshot)`
3. 把 timeline 跳到最新位置
4. 重畫最近 live log
5. 恢復即時 dispatch

若 `/api/state` 失敗，前端會退回用目前 `_events` 做一次靜態重建。

## 7. Replay Pseudo-Live 協調

replay mode 的 topology 播放與 Grafana 圖表播放不是同一條資料流。

當前端從 `PAUSED` 進入 replay `PLAYING` 時，會先：

```text
POST /api/replay/play
```

取得新的 `pseudo_session`，並把 `_grafanaSessionId` 切到這個 pseudo session。之後：

- 速度變更會呼叫 `POST /api/replay/speed`
- 暫停或 scrub 開始時會呼叫 `POST /api/replay/pause`

這代表 replay 播放中，前端自己負責 topology 與 log 的節奏；Grafana 則另外追一條 pseudo-live metrics stream。

## 8. Event Log 行為

前端 event log 有兩套輸出模式：

- live append：每來一筆事件就 append，最多保留 200 行
- static tail：在 paused / scrub 狀態重畫指定時間點之前最近 50 筆

`state_snapshot` 不會顯示在 event log 中。log 只保留一般事件，讓畫面維持可讀。

## 9. 與 Grafana 的耦合點

`events.js` 內還維護：

- `_chartWindowMin`
- `_grafanaSessionId`
- `_replayGrafanaMode`

這些屬於 DVR 控制列與 iframe 同步邏輯，詳見 [grafana-embed.md](./grafana-embed.md)。

## 10. 目前限制

- live 模式補拉歷史事件時，前端目前只做 append / sort，沒有 dedupe；若補拉區間和既有 `_events` 重疊，可能造成 scrub 或 log tail 出現重複事件
- replay 模式會把整個 session 的 event list 載入記憶體，長時間 session 的前端記憶體成本會跟著上升
- 播放控制完全由單一 `setTimeout` 鏈維持，沒有更細的 scheduler 或 checkpoint 機制
- `state_snapshot` 只在特定狀態套用；若未來 backend 想推送更多種類的權威快照，需要重新定義這層的狀態優先序
