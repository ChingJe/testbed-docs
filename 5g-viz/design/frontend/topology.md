# Topology

本文描述 `frontend/topology.js` 目前如何把 `topology.yaml` 轉成前端可互動的 Cytoscape 拓樸。

## 1. 前端拓樸層的責任

`topology.js` 目前負責四件事：

- 向 `GET /api/topology-config` 載入 profile / session 對應的拓樸設定
- 初始化 Cytoscape node、style、tooltip 與 panel 大小
- 建立 node / edge filter sidebar
- 將 event 或 `state_snapshot` 轉成可見的 node class、pulse 與 edge flash

相對地，`events.js` 不直接操作 Cytoscape 細節，而是透過 `window.Topology` 暴露的 API 來驅動拓樸。

## 2. 啟動順序

`topology.js` 在載入時立即建立一個全域 promise：

```text
window.TopologyReady
```

這個 promise 會：

1. `fetch('/api/topology-config')`
2. 建立 `NODE_ID`、`LABEL_SHORT`、`EDGE_STYLE`、`EVENT_REACTIONS`
3. 套用 `panels` 版面比例
4. 初始化 Cytoscape
5. 建立 filter sidebar

只有在這些步驟完成後，`events.js` 才會進入 bootstrap。也就是說，前端把「先有拓樸設定，再開始收事件」視為固定契約。

## 3. `topology-config` 被怎麼使用

`topology.yaml` 在前端主要被拆成下列用途：

| 欄位 | 用途 |
|---|---|
| `nodes` | 建立 Cytoscape nodes，含 `id`、`label`、`parent`、`position` |
| `nf_aliases` | 將 parser / event 中的 NF 名稱轉成前端 node ID |
| `edge_styles` | 決定邊的顏色、寬度、動畫時間與短標籤 |
| `event_reactions` | 決定 event type 對應哪些 `flash_edge` / `pulse` / `add_class` / `remove_class` |
| `layout` | 決定初始 zoom 與 pan offset |
| `panels` | 決定 topology、chart、event log 的初始版面比例 |
| `_profile` | replay 時顯示目前載入的 profile badge |

其中 `event_reactions` 不是純 UI 裝飾，而是前端重建靜態快照時的主要語意來源。

## 4. Cytoscape 模型

目前拓樸畫面採用單一 Cytoscape instance，初始化後掛在：

```text
window._cy
```

現況特性：

- 一般 NF 用圓角矩形 node 表示
- `NWDAF` 這類 compound parent 用 Cytoscape 的 parent node 表示
- edge 以動態新增 / 移除的方式表達訊號流，不是預先把所有連線常駐在圖上
- edge label 預設顯示 short label，hover 時用 tooltip 顯示 full label

目前 style 規則內建在 `topology.js`，包含：

- node 基本顏色
- `up`、`active`、`retraining` 等 class 的視覺效果
- loop edge、弧線 edge 與 hover edge 樣式

這一層沒有再把 Cytoscape style 外移到設定檔。

## 5. Filter Sidebar

filter sidebar 由 `topology.js` 依目前 config 動態生成，不再維護硬編碼清單。

### Node Filter

- 每個 top-level node 有一個 checkbox
- child node 依 `parent` 關係縮排顯示
- 若 parent 被關閉，child 會被一起隱藏，且 child checkbox 進入 disabled

### Edge Filter

- 每個 `edge_styles` key 對應一個 checkbox
- 顯示名稱優先使用 `short`，完整 label 放在 hover title

### 套用方式

`_applyFilter()` 會：

- 先依 `_nodeFilter` 決定哪些 node 顯示
- 再依 `_edgeFilter` 與 source / target 是否可見決定哪些 edge 顯示

這表示 edge filter 只控制顯示，不改變 event 本身是否存在於前端 buffer。

## 6. Event Reaction 執行模型

`window.Topology.react(event)` 是 live dispatch 與 DVR 播放共用的入口。它的流程是：

1. 依 `event.type` 查 `EVENT_REACTIONS`
2. 對每個 action 做 placeholder 展開，例如 `{from}`、`{to}`
3. 用 `nf_aliases` 把名稱解析成 node ID
4. 執行對應效果

目前支援的 action 類型：

- `flash_edge`
- `pulse`
- `add_class`
- `remove_class`

### `flash_edge`

`flashEdge()` 不是更新既有 edge，而是暫時新增一條 edge，並在 duration 後自動移除。

現況還有兩個附加行為：

- 會依 `label` 的 base 名稱回查 `edge_styles`
- 相同 key 的 edge 若短時間內過多，超過閾值後會折疊成 `...(N more)` summary edge

### `pulse`

`pulse()` 會暫時把 node 加上 `active` 效果，並做一次 border 動畫。播放速度變更時，動畫 duration 會依 `_playbackSpeed` 縮放。

### `add_class` / `remove_class`

這兩類 action 既用於 live 顯示，也用於 DVR 靜態重建。`topology.js` 會先從所有 reaction 中收集 `_managedClasses`，之後在套用 snapshot 時只清除這批由 reaction 管理的 class。

## 7. `state_snapshot` 與靜態快照

前端有兩條不同的狀態重建路徑。

### `Topology.applySnapshot(snapshot)`

這條路徑用於：

- WebSocket 連線建立後收到的初始 `state_snapshot`
- live DVR 按下 Go Live 時，由 `GET /api/state` 拉回的最新狀態

它只做兩件事：

- 清空暫時性 edge / pulse 效果
- 套用 `snapshot.node_classes`

### `Topology.renderStaticSnapshot(events, targetMs, edgeWindowMs, options)`

這條路徑用於 scrub 與 paused view。它會：

1. 從頭掃描 `events <= targetMs`
2. 依 `add_class` / `remove_class` 重建各 node 的累積 class
3. 取 `targetMs ± edgeWindowMs` 內的 `flash_edge` 與 `pulse` 作靜態呈現
4. 視需要保留既有 active node 樣式

這也是前端在不依賴 backend state 的情況下，重建任意時間點拓樸畫面的核心機制。

## 8. 對外暴露的 Topology API

`topology.js` 目前暴露下列 API 給 `events.js` 使用：

- `react(event)`：依 `event_reactions` 做即時反應
- `applySnapshot(snapshot)`：套用 backend 提供的權威狀態
- `renderStaticSnapshot(events, targetMs, edgeWindowMs, options)`：重建任意時間點的靜態畫面
- `clearTransientEdges()`：清除目前所有暫時性視覺效果
- `captureActiveNodes()`：保存目前 active node 樣式
- `setPlaybackSpeed(speed)`：讓 pulse / edge duration 隨播放速度縮放

其中 `clearTransientEdges()` 名稱較窄，但目前實作其實也會清除 transient node effect。

## 9. 目前限制

- Cytoscape style 與 reaction execution 仍全寫在 `topology.js`，尚未拆成更細的前端模組
- 靜態快照重建採用「從頭 replay 到 targetMs」策略；事件量再高一個數量級時可能需要 checkpoint
- filter 狀態只存在前端記憶體，重新整理頁面就會重置
- `_managedClasses` 只管理由 `event_reactions` 宣告過的 class；若未來直接在別處增減 class，snapshot 路徑不一定能完整重建
