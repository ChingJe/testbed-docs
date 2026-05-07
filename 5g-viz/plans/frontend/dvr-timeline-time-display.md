# DVR Timeline Time Display

日期：2026-05-07

定位：此文件描述的是 5g-viz DVR 控制列中，時間軸右側時間文字的可讀性改善規劃。範圍屬於前端 UI / interaction，而不是 replay 資料模型或後端 API 設計。

## 0. 實作收斂

此題已於 2026-05-07 完成第一輪落地，現況如下：

- 主顯示已從絕對時鐘改為相對時間。
- replay 現在顯示 `elapsed / total`。
- live 現在顯示 `elapsed live`。
- 絕對時間已降級為 `title` tooltip，而非主畫面常駐資訊。
- 毫秒已從主顯示移除。
- 額外補上了 playhead heartbeat，避免時間文字只在 event 到達時才刷新。
- heartbeat 覆蓋：
  - `live + LIVE`
  - `live + PLAYING`
  - `replay + PLAYING`

因此本文件後續除了原始設計脈絡，也同時記錄這輪實作中補出的行為邊界。

## 1. 問題定義

目前時間軸右側顯示為：

```text
21:08:58.920 / 22:51:58.638
```

這種格式對工程除錯有幫助，但對一般使用者不直覺，原因是：

- 它呈現的是絕對 wall-clock time，不是播放進度
- 使用者必須自行心算「現在播到哪裡」與「總長多久」
- 毫秒常駐顯示會增加視覺噪音
- 在 replay 模式下，這個區塊看起來更像 log timestamp，而不像 media player 的 timeline

除此之外，還有一個互動層問題：

- 若時間文字只在 event 進來時才更新，當兩筆 event 間隔較長時，畫面會看起來像暫停或卡住
- 這在 replay `PLAYING` 與 live mode 的 buffered `PLAYING` 狀態都會造成錯誤感知

## 2. 設計目標

- 讓使用者一眼看懂「目前播到哪裡 / 總共多長」
- 保留必要的絕對時間資訊，但不讓它成為主視覺
- 讓時間文字在播放期間持續更新，而不是依賴下一筆 event
- 不改變現有 replay、scrub、pseudo-live、Grafana sync 的時間計算邏輯
- 優先採用前端顯示層改動，不新增後端依賴

## 3. 推薦顯示模型

### 3.1 Replay

主顯示改為相對時間：

```text
00:05 / 01:43:05
```

語意：

- 左側：目前 playhead 相對於 session start 的 elapsed time
- 右側：整段 replay 的 total duration

次要資訊改為絕對時間，建議放在：

- `title` tooltip
- 或第二行小字

範例：

```text
00:05 / 01:43:05
2026-05-05 21:08:53 -> 2026-05-05 22:51:58
```

### 3.2 Live

live 沒有固定總長度，不適合沿用 `current / total`。

建議主顯示為：

```text
00:12:34 live
```

或：

```text
elapsed 00:12:34
```

絕對開始時間則放在 tooltip 或第二行：

```text
since 2026-05-07 14:21:03
```

### 3.3 毫秒顯示策略

毫秒不應作為常駐主顯示。

建議規則：

- 主顯示只到秒
- `title` tooltip 可保留毫秒
- 若未來有需求，scrubbing / paused 狀態可再考慮顯示毫秒，但不作為第一版必做項目

## 4. 實作範圍

### 4.1 主要修改點

- `frontend/events.js`
  - 新增相對時間 formatter
  - 新增簡化版絕對時間 formatter
  - 重寫 `_syncTimelineUi()` 的顯示字串組裝邏輯
  - 新增 playhead heartbeat，讓播放期間的時間顯示能獨立刷新
- `frontend/index.html`
  - 視需要將 `#dvr-time` 從單一 `<span>` 擴充成兩行顯示
  - 若第一版採 tooltip-only，可維持單一節點
- `frontend/index.html` 內嵌 CSS
  - 微調 `#dvr-time` 的寬度、對齊與 mobile breakpoint

### 4.2 明確不改的地方

- 不改 `timeline` slider 本身的 min/max/value 計算
- 不改 replay event scheduling
- 不改 `/api/session-info`
- 不改 Grafana query window 計算
- 不改 replay / live 的 session metadata contract

## 5. 最小可行方案

第一版建議採用最低風險版本：

1. 保留單一 `#dvr-time` 節點
2. replay 顯示 `elapsed / total`
3. live 顯示 `elapsed live`
4. 絕對時間資訊放到 `title`
5. 主畫面移除毫秒
6. 播放期間由 heartbeat 持續刷新時間文字，不等待下一筆 event

這個方案的優點：

- 不需要新增新 DOM 結構
- 幾乎只改 `events.js`
- 對 mobile layout 衝擊最小
- 已能大幅改善可讀性

## 6. 進階方案

若之後要再提升可讀性，可升級為雙層顯示：

```text
00:05 / 01:43:05
21:08:53 -> 22:51:58
```

此方案的好處是不用 hover 也能讀到絕對時間，但代價是：

- 需要改 HTML 結構
- 需要補 CSS
- 在窄螢幕上較容易擠壓控制列

因此建議作為 Phase 2，而不是第一步。

## 7. 風險

### 7.1 將顯示邏輯和時間計算邏輯混在一起

這次改動應只影響「如何呈現字串」，不應動到：

- `_timelineMinMs`
- `_timelineMaxMs`
- `_timelinePosMs`
- `_setTimelinePos()`
- `_recomputeTimelineBounds()`

若把顯示格式改動和 timeline 計算一起重構，容易誤傷 replay、scrub、pause、play。

### 7.1A 時間顯示只靠 event 刷新的錯覺風險

若時間顯示只在 `dispatch(event)` 或 `_handleLiveEvent()` 時更新，使用者在以下情況會誤判系統狀態：

- replay 播放中，但下一筆 event 還沒到
- live mode 在播放 buffered timeline，但事件分布較稀疏
- live mode 處於 `LIVE` 狀態但上游暫時安靜

因此時間顯示刷新必須與 event arrival 解耦。

### 7.2 Imported session 的 lead-in 影響認知

目前 imported session 預設會加入 5 秒 lead-in。

這代表：

- replay 開頭的 `00:00 -> 00:05` 可能沒有事件
- 這是刻意設計，不是時間錯誤

因此 UI 文案應接受「相對時間從 session start 算起」，而不是從第一筆 event 算起。

### 7.3 Live 與 replay 的語意不同

replay 有明確總長度，live 沒有。

若硬把 live 也做成 `current / total`，會讓使用者誤以為 live session 有固定結尾。這是不正確的心智模型。

### 7.4 Mobile 版控制列擠壓

若改成雙行或加入更多說明文字，`#dvr-controls` 在窄螢幕可能變得擁擠。

第一版若採 tooltip-only，可大幅降低這個風險。

### 7.5 絕對時間時區認知

前端若使用瀏覽器本地時區格式化絕對時間，可能和 session 原始時區不同。

短期內可接受，因為目前 `Date` 物件整體就是這樣運作；但文件應知道這是一個顯示層選擇，不是 replay 核心時間錯誤。

## 8. 驗證計畫

### 8.1 功能驗證

- replay 模式載入後，右側主時間顯示為相對時間
- live 模式載入後，右側主時間不再顯示 `... / now` 的絕對時鐘形式
- play / pause / scrub 時，主時間會隨 playhead 正常更新
- imported session 的 5 秒 lead-in 能正常顯示為開頭空白段

### 8.2 互動驗證

- timeline 拖曳時顯示數字不卡頓
- replay 播放時數字持續更新且不抖動
- live mode 的 `PLAYING` 狀態下，時間數字持續更新，不會只在 event 到達時跳動
- live mode 的 `LIVE` 狀態下，若暫時沒有新 event，時間仍持續往前
- pause 後數字穩定停留在當前位置

### 8.3 版面驗證

- 桌面版控制列未溢出
- mobile breakpoint 下時間文字不會把 slider 擠到不可用

## 9. 實作建議

### Phase 1

- 只改 `events.js`
- 將 replay 主顯示改為 `elapsed / total`
- 將 live 主顯示改為 `elapsed live`
- 將絕對時間放在 `title`
- 補一層前端 playhead heartbeat，讓 `LIVE/PLAYING` 期間的數字持續刷新

### Phase 2

- 視需求改成雙行顯示
- 若雙行顯示成立，再補精簡絕對時間格式

## 10. 決策摘要

- 本題歸類在 `plans/frontend/`
- 這是 DVR UI 可讀性問題，不是 replay 資料流問題
- 第一版優先做最小風險顯示層改動
- 主顯示採相對時間
- 絕對時間降級為次要資訊
- 毫秒不常駐顯示
- 時間刷新不能依賴 event arrival，播放期間必須有獨立 heartbeat
