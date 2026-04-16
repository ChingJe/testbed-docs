# Topology

## 這個區塊在做什麼

Topology 是主畫面中負責顯示互動、流程與狀態投影的區塊。

它不是整個系統的原樣重建，而是把結構化事件轉成：

- node 狀態
- node class
- edge 動畫
- pulse 效果
- 某個時間點的靜態重建畫面

從畫面用途來看，Topology 適合回答的問題是：

- 哪些 NF 目前正在互動
- 哪些流程剛剛發生
- 哪些節點目前處於某種持久狀態
- 某個時間點的畫面長什麼樣

## Topology 上會出現哪些內容

### 1. 持久狀態

這一類內容不會立刻消失，會持續留在節點上，直到之後有對應事件把狀態移除。

目前最重要的例子包括：

- `retraining`
- 其他由 event reaction 累積出的持久 class

這類內容通常代表：

- 某個節點已經進入某種目前狀態
- 畫面需要在新分頁、Go Live、replay 初始化時仍然可以重建這個狀態

### 2. 瞬時效果

這一類內容主要用來表達「剛剛有事件發生」，而不是長期狀態。

目前最常見的是：

- edge flash
- node pulse

這些效果適合拿來看：

- 某次 SBI call 的流向
- 某次 internal flow 的方向
- 某個 node 是否剛剛被事件觸發

### 3. 靜態重建結果

當畫面進入 pause 或 scrub 時，Topology 不再只依賴新事件往前推，而是根據既有事件重建指定時間點的畫面。

這時畫面上的內容是：

- 截至某個時間點為止的持久狀態
- 指定時間窗附近的 edge / pulse 效果

所以 pause / scrub 狀態下的 Topology，比較接近「某個時間點的狀態投影」，不是持續更新中的 live canvas。

## Topology 與設定檔的關係

Topology 的主要設定來源是 `topology.yaml`。

對畫面最有直接影響的欄位包括：

- `nodes`：節點與位置
- `nf_aliases`：事件名稱到畫面節點 ID 的對應
- `edge_styles`：不同邊類型的顏色、寬度、持續時間、短標籤
- `event_reactions`：不同 event type 觸發哪些視覺效果或持久 class
- `layout`：初始 zoom / pan
- `panels`：畫面比例

這代表 Topology 不是靠大量硬編碼規則定義，而是高度依賴 topology config。

## Filter Sidebar 的用途

Filter Sidebar 是 Topology 的可視化過濾器。

它主要分成兩類：

- node filter
- edge filter

### Node filter

Node filter 適合處理下列情境：

- 暫時只看某些 NF
- 暫時隱藏 compound parent 與其子節點
- 降低畫面擁擠程度

若某個 parent node 被關閉，子節點也會一起隱藏。

### Edge filter

Edge filter 適合處理下列情境：

- 只看某種 SBI call
- 暫時隱藏某些噪音較高的邊類型
- 專注在某條 feature flow

Filter 只改變畫面上的可見性，不改變事件是否存在於前端緩衝或 session。

## Live 與 Replay 下的 Topology 差異

### Live

Live 模式下，Topology 主要跟著新事件往前更新。

主要行為包括：

- 新事件到達後直接套用 event reaction
- 新連線先拿到目前狀態的 snapshot
- Go Live 時優先回到 backend 提供的權威 snapshot

這裡有一個很重要的分工：

- 新 event 負責讓畫面繼續往前變化
- `state_snapshot` 負責讓畫面立刻落在目前正確狀態

若沒有 `state_snapshot`，新分頁只從當下開始收 event，通常只能看到「從現在開始又發生了什麼」，卻無法立刻知道：

- 哪些 node class 已經成立
- 哪些持久狀態早就在更早之前形成
- 目前畫面應該從哪個狀態起跑

因此 `state_snapshot` 對 Topology 的意義，不是把歷史再播一遍，而是直接提供目前畫面的起始狀態。

因此 live Topology 的核心特性是：

- backend runtime 仍在繼續前進
- 前端可以暫時離開 live 視角，但畫面之後仍可回到當前狀態

### Replay

Replay 模式下，Topology 主要依賴既有事件重建畫面。

主要行為包括：

- 啟動時載入 session 事件
- timeline 依既有錄製範圍建立
- pause / scrub / play 都建立在既有事件之上

Replay Topology 的核心特性是：

- 沒有新的 live event 在後面持續產生
- 每個畫面都來自既有 session 的重建與播放

## Topology 適合看什麼，不適合看什麼

### 適合

- 互動方向
- 哪個節點剛剛被觸發
- 哪些節點目前處於持久狀態
- 某條 feature flow 是否實際發生

### 不適合

- 精確數值比較
- 指標時間序列變化
- 判定某個 metrics 是否真的已進 Prometheus / Grafana

這些內容更適合交給 Event Log 與 Grafana。

## 常見誤解

### Topology 有反應，不代表 Grafana 一定有反應

原因是：

- 不是所有 event 都會進 metrics 層
- 有些 event 只會影響 UI 視覺效果或持久 class

### 新分頁沒有補演歷史動畫，不代表資料遺失

新分頁在 live 下先拿到的是當前 `state_snapshot`，不是完整歷史動畫重播。

這種行為的含義是：

- 畫面先對齊目前狀態
- 之後再由新 event 繼續推進

若需要觀察過去是怎麼走到這個狀態，應改用 DVR 與 event history，而不是期待新分頁自動補演整段 live 歷史。

## 對照閱讀

- Event Log：[`./event-log.md`](./event-log.md)
- DVR Controls：[`./dvr-controls.md`](./dvr-controls.md)
- Event History：[`./event-history.md`](./event-history.md)
- deeper reference：[`../design/frontend/topology.md`](../design/frontend/topology.md)
