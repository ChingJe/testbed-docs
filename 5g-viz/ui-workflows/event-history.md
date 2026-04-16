# Event History

## 這份文件在說什麼

本文件說明主畫面中的 event 何時進到前端、前端會保留哪些 event、這些 event 會在什麼操作裡被重新使用，以及哪些情況下前端會再向後端補抓較早的歷史。

這一層的重點不是 parser 或 state machine 細節，而是回答幾個直接影響操作理解的問題：

- live event 什麼時候到前端
- replay event 什麼時候載入前端
- 前端本地記憶體裡保留的是什麼
- timeline 拖回過去時，畫面用的是本地事件還是重新抓一次
- event log 顯示筆數，和前端實際保留的 event 筆數，為什麼不是同一件事

## Event 何時到前端

### Live

Live 模式下，前端主要透過 `/ws` 收事件。

連線建立後，最先可能收到的是：

- `state_snapshot`

之後持續收到的是一般事件，例如：

- `aggregated_slot`
- `ml_inference`
- `accuracy`
- 各種只影響 Topology 的 UI event

`state_snapshot` 和一般事件在前端的用途不同：

- `state_snapshot` 主要用來同步目前權威狀態
- 一般事件主要用來推進 Topology、Event Log、DVR playback 與 scrub 重建

## 為什麼 live 新連線不能只靠「從現在開始收 event」

只從目前連線之後開始收 event，無法回答「現在這一刻畫面應該長什麼樣」。

原因是 live event stream 提供的主要是：

- 新發生的變化
- 瞬時互動
- 狀態增減的過程

但新分頁剛打開時，畫面需要先知道的是：

- 哪些持久 class 目前已經成立
- 例如某個 node 是否仍處於 `retraining`
- 哪些狀態是在這個連線建立之前就已經累積完成

如果只從現在開始收 event，前端只會知道「從現在開始又新增了什麼變化」，卻不知道：

- 目前已經存在的持久狀態
- 先前事件已經留下的畫面結果
- 某些短暫事件是否早就發生過且已經結束

這也是為什麼單靠開始收新 event，並不等於拿到最新狀態。

## `state_snapshot` 真正解決的是什麼

`state_snapshot` 解決的是「新連線或回到 live 時，前端需要立刻拿到目前權威畫面狀態」這件事。

它的用途不是取代 event stream，而是補上 event stream 不負責的那一段：

- event stream：描述接下來又發生了哪些變化
- `state_snapshot`：描述此刻已經成立的狀態結果

可以把兩者的分工簡化成：

```text
state_snapshot = 現在畫面應該站在哪個狀態
event stream    = 接下來又有哪些新變化發生
```

因此 live 新分頁的典型順序是：

1. 先用 `state_snapshot` 把 Topology 立刻放到目前狀態
2. 再開始收新的 live event，讓畫面繼續往前推進

沒有這一步時，新分頁必須等待未來事件再次出現，才可能慢慢拼回目前狀態；有些狀態甚至永遠不會自己補回來，因為對應事件早就在更早之前發生完畢。

### Replay

Replay 模式下，前端不靠 WebSocket 持續收新事件。

前端啟動時會先呼叫：

```text
GET /api/events?session=<session_id>
```

把該 replay session 的事件載入前端記憶體，之後 pause、scrub、play 都建立在這份既有事件集合上。

## 前端實際保存的是什麼

前端會把一般事件放進本地 `_events` 緩衝。  
這個緩衝是 DVR 的核心資料池，不只是 Event Log 的資料來源。

目前的重要特性是：

- `state_snapshot` 不會進 `_events`
- 一般事件會持續累積在 `_events`
- `_events` 目前沒有固定上限，也沒有定期清理
- 事件在需要時才排序，不是每收到一筆就完整重排

這代表兩件事：

1. 前端保存事件的時間長度，等同於目前頁面生命週期內已收進來或已補抓進來的歷史範圍。
2. 長時間 live session 會讓前端記憶體中的 `_events` 持續成長。

## Event Log 顯示筆數，不等於前端保存筆數

畫面上的 Event Log 有顯示上限，但前端事件緩衝沒有同樣的上限。

目前有兩種常見顯示方式：

- live append：畫面最多保留 200 行
- paused / scrubbed static tail：畫面重畫指定時間點之前最近 50 筆

這只是 UI 顯示裁切，不代表前端只保留 200 或 50 筆事件。

前端真正用來支撐 replay、scrub、playback 的，是 `_events` 這份本地事件集合。

## 這些 event 會在什麼時候被拿來用

### Live 視角

當前端處於 `LIVE` 狀態時，新事件一進前端就會立刻拿來：

- 更新 Topology
- append 到 Event Log
- 推進 timeline 右邊界

### Pause / Scrub

當畫面進入 `PAUSED` 或 `SCRUBBING`，前端不再只看「最新一筆事件」，而是改用本地 `_events` 重建指定時間點的畫面。

此時會發生的事情是：

- Topology 從 `_events` 重建該時間點的靜態畫面
- Event Log 從 `_events` 取該時間點之前最近一段 tail
- Grafana 另外切到對應時間窗，但它讀的是 metrics 路徑，不直接讀 `_events`

### Play

播放時，前端也不是向後端逐筆要事件，而是：

- 在本地 `_events` 內找到目前 playhead 之後的第一筆事件
- 依事件原始時間差與 playback speed 逐筆往前推進

### Go Live

`Go Live` 會優先向後端取最新權威狀態：

```text
GET /api/state
```

若這條路徑失敗，前端才退回用本地 `_events` 在最新時間點做一次靜態重建。

這表示 `Go Live` 不是單純把本地播放指標拉到最右邊，而是優先回到 backend 提供的目前狀態。

## 拖曳回過去時間時，用的是本地事件還是重新抓一次

答案是：兩種都有，但優先使用本地事件。

### Replay

Replay 幾乎完全建立在前端已載入的 session events 上。

一般情況下，拖曳 replay timeline 時：

- 不會再次向後端補抓整段事件
- 主要直接使用前端已載入的 `_events`

### Live

Live 下拖曳 timeline 時，前端會先檢查本地 `_events` 是否已涵蓋目標時間點以前的歷史。

若本地最早事件已經早於目標時間點，前端直接使用本地 `_events`。

若本地緩衝還不夠早，前端才會補抓：

```text
GET /api/events?session=<session_id>&from=<session_start>&to=<target_time>
```

補抓回來的事件會合併進 `_events`，之後 scrub 重建與 log tail 就改用這份擴充後的本地集合。

## Live 補抓歷史的邊界

這條補抓路徑目前有幾個重要特性：

- 補抓只發生在 live scrub history 不夠早時
- 目前有 single-flight 保護，同一時間只會有一條 backfill request
- 補抓後事件會併進 `_events`
- 目前前端沒有 dedupe；若補抓區間與既有事件重疊，理論上可能出現重複事件

這也是為什麼 live DVR 的歷史視角，不是完全只靠前端或完全只靠後端，而是「先用前端 buffer，不夠時再補拉」。

## `state_snapshot` 在這層的位置

`state_snapshot` 很重要，但它和一般事件不是同一種用途。

它主要出現在兩個場景：

- live 新連線時的目前狀態同步
- `Go Live` 時回到最新權威狀態

它通常不出現在 Event Log，也不是 replay / scrub / play 主要依賴的事件集合。

它比較像：

- live 畫面的初始落點
- `Go Live` 時的重新對齊依據
- backend 對「目前狀態」的權威答案

這也是為什麼新開 live 分頁時，常見現象是：

- 先看到目前狀態
- 不會先補演整段歷史動畫

## 這份前端 event history 解釋了哪些常見現象

### 新分頁只看到當前狀態

因為 live 新連線優先同步的是 `state_snapshot`，不是整段歷史回放。

這不是少資料，而是先把畫面放到目前狀態，再從這個狀態往後收新事件。

### Pause 之後畫面不動，但 live runtime 仍在前進

因為新事件仍會透過 WebSocket 進前端並累積到 `_events`，只是前端暫時停在某個歷史視角，不立即 dispatch。

### 拖回較早時間時，畫面有時會停一下

因為本地事件緩衝可能還不夠早，需要向 `/api/events` 補抓歷史。

### Event Log 只看到幾十或幾百筆，不代表更早事件不存在

因為 Event Log 顯示筆數有限制，但 `_events` 緩衝不是同一個上限。

## 對照閱讀

- Event Log：[`./event-log.md`](./event-log.md)
- DVR Controls：[`./dvr-controls.md`](./dvr-controls.md)
- deeper reference：[`../design/frontend/events-and-dvr.md`](../design/frontend/events-and-dvr.md)
