# Grafana Window Rendering Notes

本文件整理 DVR 實作過程中發現的「時間窗渲染」問題。這些議題與 DVR 互相關聯，但不屬於 DVR 主流程本身，因此從 `dvr-mode/` 主線抽離。

## 範圍

主要討論：

- 小時間窗下的折線邊界裁切行為
- `centered` 與 `trailing` window 的使用者語意
- 右邊界缺線問題
- 是否需要 over-fetch / dynamic epsilon
- 哪些問題是 Grafana / Prometheus range query 的限制，哪些是 5g-viz 可自行控制的策略

不在本文件範圍內：

- replay `pause/backfill` 與 `play/pseudo-live` 的數值一致性
- pseudo-live 的 timestamp remap / remote write 副作用  
  這些另見 [replay-pseudo-live-consistency.md](replay-pseudo-live-consistency.md)

## 背景

在 DVR 的 `PAUSED / SCRUBBING` 狀態下，使用者會希望觀察較小的時間區間。此時如果 chart window 很窄、而資料點相對稀疏，Grafana 會只把線畫到觀察窗內最後一個 sample 為止；若右邊界與下一個 sample 之間沒有更多窗內資料，圖右側就會出現一段無線區域。

這並不代表資料錯誤，而是因為目前的 `Grafana iframe + Prometheus range query` 路徑不會主動取得右邊界之外的下一個點來做裁切。

## 與 DVR 的關係

下列內容仍屬 DVR 主線，留在 `dvr-mode/`：

- `Chart` 時間窗口控制項本身
- `Pause / Play / Scrub / Go Live`
- replay pseudo-live 與 backfill 切換

下列內容則屬於圖表渲染策略，留在本文件：

- 右邊界缺線是否要修
- 若要修，採用固定 epsilon、動態 epsilon，或改用自製 detail chart
- 使用者選取的 nominal window 與實際查詢 window 是否可以不同

## 目前已知事實

1. 問題在固定大視窗時較不明顯，但在可調小視窗下會被放大。
2. `centered` window 容易掩蓋右邊界缺線，因為右半邊常包含更多未來點。
3. `trailing` window 更符合「觀察某一時間點當下為止」的直覺，但也更容易暴露右邊界沒有下一點可接線的問題。
4. 若實作 over-fetch，必須明確標註這是 rendering / UX 策略，而非名義上精確不變的 timebox。

## 待評估方向

### 1. 保守作法：限制最小 chart window

- 降低問題出現頻率
- 不改變目前 Grafana 使用方式
- 屬於 workaround

### 2. 固定 epsilon over-fetch

- 實際查詢區間略大於使用者選取區間
- 例如 `to_eff = min(to + 15s, hard_limit)`
- 容易落地，但仍是近似解

### 3. 動態 epsilon（延到下一個實際點）

- 以右邊界之後第一個相關點作為實際查詢終點
- 比固定 epsilon 更精準
- 需先定義「哪一類點」算是 chart 的下一個點

### 4. 自製 detail chart

- 在 `PAUSED / SCRUBBING` 時不再完全依賴 Grafana iframe
- 可自行 over-fetch 左右邊界點並做裁切
- 工程量中等，但能更正確控制體驗

## 判斷原則

若需求重點是：

- 快速改善小時間窗 UX：可優先評估 over-fetch
- 嚴格維持觀察窗語意：應傾向自製 detail chart 或更明確的邊界裁切策略

目前結論是：此問題已被確認存在，但不阻塞 DVR 主流程完成；是否要解，應以獨立 chart rendering 任務評估。
