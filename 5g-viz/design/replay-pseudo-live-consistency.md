# Replay Pseudo-Live Consistency Notes

本文件整理 replay mode 下，`PAUSED / backfill` 與 `PLAYING / pseudo-live` 圖表不一致的問題。這些議題是在 DVR 實作過程中暴露出來的資料一致性問題，應與一般 chart window / 邊界裁切問題分開討論。

## 範圍

主要討論：

- 為什麼 `Pause` 與 `Play` 看到的同段歷史資料可能不同
- 為什麼同一段歷史在不同次 `Play` 之間也可能明顯不同
- pseudo-live 的 timestamp remap 與 Prometheus / Grafana step 對齊問題
- 哪一條路徑目前比較可信

不在本文件範圍內：

- 小時間窗右邊界缺線
- `centered` / `trailing`
- over-fetch / epsilon  
  這些另見 [grafana-chart-rendering.md](grafana-chart-rendering.md)

## 背景

replay mode 目前有兩條圖表資料路徑：

1. `PAUSED / SCRUBBING`
   - 查原始 replay session 的 backfill 資料
   - 使用原始 event timestamp

2. `PLAYING`
   - 查每次 `Play` 產生的 `pseudo_session`
   - 由 pseudo-live pipeline 將歷史資料重新映射到 `now` 相對時間，再透過 remote write 寫入 Prometheus

因此，即使使用者表面上看的是「同一段歷史」，底層實際上查的是兩組不同時間軸上的 series。

## 觀察到的現象

- `Pause` 時的 backfill 圖表，與 `Play` 時的 pseudo-live 圖表，對同一段歷史區間可能長得不一樣。
- 某些情況下，即使是 `Play` 當下「過去」那一段已經 pre-seed 的資料，也會和先前 `Pause` 看到的值差很多。
- 實測上甚至可能出現同一時間點差到數倍的情況。

## 目前判讀

### 1. Pause 與 Play 不是在看同一條時間軸

- `Pause`：看原始 replay backfill
- `Play`：看當次按下播放後生成的 `pseudo_session`

`pseudo-live` 每次按下 `Play` 都會用當下的 wall clock 重新映射 sample timestamp，因此同一段歷史資料在不同次播放間，不會落在完全相同的 timestamp 上。

### 2. Grafana / Prometheus step 會放大這種差異

Grafana range query 會依 panel 的 step 取樣；同一段資料若落在不同 timestamp 相位上，最後取到的點位組合就可能不同，尤其在資料本身較稀疏時更明顯。

### 3. `sum by(target)` 不是目前的主因

目前資料裡，每個 `group` 只對應 1 個 `sub_id`。因此 panel query 中的 `sum by(target)` 在現況下幾乎是 no-op，不是眼前數倍差異的主因。

但這不代表 `sum by(target)` 永遠安全；若未來同一個 group 對應到多個 `sub_id`，它仍可能成為新的風險來源。

## 現階段結論

- 若目標是「忠實看歷史值」，`PAUSED / backfill` 比 `PLAYING / pseudo-live` 更可信。
- `PLAYING / pseudo-live` 應被視為「近似 live 的播放視圖」，而不是與 backfill 完全等價的精確重建。
- 此問題影響的是 replay chart consistency，不直接阻塞 DVR 主流程，但會影響數值可信度，應以獨立議題持續追蹤。

## 待評估方向

### 1. 接受 pseudo-live 為近似視圖

- 明確把 `PLAYING` 定位為播放體感優先，而不是數值完全一致
- 在文件中強調 `Pause` 才是較可信的歷史檢視模式

### 2. 讓 pseudo-live 時間映射更穩定

- 例如將 sample timestamp 量化到固定 bucket
- 目標是降低不同次 `Play` 間的相位漂移
- 可能只能減輕問題，不一定根治

### 3. 調整資料建模

- 避免直接用目前這種 raw sparse gauge 做 pseudo-live replay
- 若後續要追求更高一致性，可能需要重新思考 replay 圖表的資料表示方式

## 結論

這是 DVR / replay 實作中暴露出來的資料一致性問題，但不應與一般 chart window 渲染問題混在同一份文件裡。後續若要處理，建議以獨立 replay consistency 任務評估。
