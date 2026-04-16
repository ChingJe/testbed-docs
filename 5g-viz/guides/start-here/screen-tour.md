# Screen Tour

本文件依畫面區塊說明主畫面的角色與用途。

## 1. Header：模式與連線狀態

畫面最上方的 header 不是裝飾，它負責顯示目前整個畫面的狀態。

Header 包含：

- 標題
- 連線 / 模式狀態
- 如果不是 default profile，還會有 profile badge

最重要的是狀態文字：

- `connected`：live 模式下 WebSocket 已連上
- `disconnected`：live 模式下目前斷線
- `replay`：目前不是 live stream，而是在 replay session

這個區塊是判斷目前是否處於 live stream 的第一個指標。

## 2. DVR Controls：時間視角控制

Header 下方那排 controls 是整個畫面的時間控制中心。

主要按鈕與控制項包括：

- `Pause`
- `Play`
- `Go Live`
- timeline slider
- speed selector
- chart window
- reset chart

這一排不是只控制 topology。  
它實際上會一起影響：

- topology 現在顯示哪個時間點
- event log 現在顯示哪一段
- Grafana 現在應該看即時視窗，還是某個 trailing window

這一排 controls 的基本語意是：

- `Pause` 控制的是當前觀看視角，不是 backend runtime 本身

其中 `Go Live` 只在 live mode 有意義；在 replay 裡沒有「回到最新 live 狀態」這件事。

## 3. Filter Sidebar：可視化過濾

左側的 filter sidebar 是 topology 的可視化過濾器。

它不是硬編碼清單，而是根據目前 topology config 生成的，所以：

- node filter 會跟著 topology 裡定義的 nodes 出現
- edge filter 會跟著 `edge_styles` 裡定義的 label types 出現

主要用途有兩個：

- 只看某些 NF
- 只看某些 SBI / edge 類型

要注意的是：

- filter 只影響目前畫面上的可見性
- 它不會改變 event 是否存在於 session，也不會改變 backend 真正收到了哪些資料

## 4. Topology：事件反應與狀態投影

Topology 不是整個系統的原樣複製，比較準確的理解是：

- 它顯示的是事件對 UI 的反應結果

這個區塊包含幾種不同性質的內容：

- node 的持久狀態  
  例如某個 node 正處於 `retraining`，或某些持久 class 已經成立

- transient edge / pulse  
  例如某次 SBI call、某次 internal flow、某次 pulse 效果

- static rebuild 結果  
  當 pause / scrub 發生時，topology 不是在等新事件，而是在某個時間點上重建畫面

需要區分的是：

- edge 在閃，不代表 Grafana 一定會有線
- node 有 class，不代表 event log 正在即時 append

因為 topology、event log、Grafana 不是同一條路。

## 5. Grafana 區塊：metrics 層

Topology 下方的 Grafana 區塊是嵌入進來的 dashboard。

這裡主要用來看：

- ground truth traffic
- predicted traffic
- model deviation
- retrain annotation

這個區塊最重要的前提是：

- Grafana 看的不是原始 event list
- 而是「部分 event 被轉成 Prometheus metrics 之後」的圖表

所以：

- 有些 event 會讓 topology 動，但 Grafana 不會有反應
- Grafana 沒有線，也不代表 topology 一定沒東西發生

在 live 模式下，它通常跟著 `now` 走。  
在 pause / scrub / replay 時，它會改看別的時間窗；在 replay `playing` 時，它甚至會切到另一條 pseudo-live 路徑。

## 6. Event Log：結構化事件清單

最下方的 event log 是整個畫面裡最接近「系統剛剛到底認定發生了什麼」的一塊。

它列的是 parser 產生的結構化事件，而不是原始 log line。

主要用途有兩個：

- 確認剛剛 topology 的動作對應到哪種 event
- 確認 event payload 裡有哪些欄位

要注意的是：

- 這裡不會把所有 internal 狀態都列出來
- `state_snapshot` 不是主要的事件閱讀對象
- pause / scrub 時，這塊會從「即時 append」切成「某個時間點之前的 event tail」

所以 event log 不只是單純往下長的 console；它也會跟著 DVR 視角切換。

## 7. 主畫面的三層觀測

主畫面可拆成三層觀測：

1. topology 層  
   看事件如何被轉成 node / edge / class / pulse

2. event 層  
   看 parser 最終產出的結構化事件

3. metrics 層  
   看部分 event 如何進一步變成 Grafana 上的圖

很多模式差異與 troubleshooting，都建立在這三層不是同一路徑的前提上。

## 8. live 與 replay 的快速區分點

最容易判斷的幾個觀察點包括：

- Header 是 `connected` 還是 `replay`
- `Go Live` 有沒有意義
- timeline 右邊界是不是還在往前動
- event log 是不是在即時長
- Grafana 是在跟 `now` 跑，還是在某個固定時間窗

這些觀察點比 deep reference 更適合作為第一層判斷方式。

## 9. 後續閱讀

- 模式差異：回看 [Live Vs Replay](./live-vs-replay.md)
- deeper reference：
  - [`../../design/frontend/topology.md`](../../design/frontend/topology.md)
  - [`../../design/frontend/events-and-dvr.md`](../../design/frontend/events-and-dvr.md)
  - [`../../design/frontend/grafana-embed.md`](../../design/frontend/grafana-embed.md)
