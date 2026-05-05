# UPF Hybrid Replay Duplicate `startTime` Investigation

**Date:** 2026-05-05  
**Component:** `go-upf-ess` EES Hybrid Mode ↔ NWDAF AnLF  
**Audience:** `go-upf` / UPF-EES 維護者  
**Related:** [upf-double-notification-report.md](upf-double-notification-report.md)

---

## 摘要

這次 `0505-18` testbed 實驗顯示，`NWDAF` 端大量出現的 `dedup collision`，不是 `NWDAF` 自己隨機產生的異常，而是因為 `go-upf` 實際上把**同一個 `ip + startTime` 的量測 window 重複送了兩次**。

這個現象在本次實驗中有幾個明確特徵：

- 最早重複的 `startTime` 出現在：
  - `10.10.0.1 @ 2026-05-05T10:00:02Z`
  - `10.100.0.1 @ 2026-05-05T10:00:03Z`
- 同一個 `ip + startTime` 會先在一個 notification 中出現一次，約 `30s` 後又在下一個 notification 中再次出現。
- `NWDAF` 的 `UPF VOLUME` log 已直接證明這些重複上報存在。
- `NWDAF` 之後只是把每個 item 直接 append 到 `RawUpfData`，在 inference 前才做 anchor-relative dedup，因此才會看到大量 `dedup collision ... keeping later`。

這不是單純資料集 checksum 錯誤，也不是 `NWDAF` config 錯誤。  
目前最強的推論是：**`go-upf` 的 hybrid mode 在 pseudo phase2 與 kernel URR 交錯時，讓相鄰兩輪 notify 對同一個 logical window 產生重疊。**

更具體地說，這次問題不是「同一輪 notification 內有兩筆不同 slot 很正常」而已，而是：

1. 某個 slot 已經在 notification `N` 被送出
2. 約 `30s` 後，在 notification `N+1` 又再次出現相同的 `ip + startTime`
3. `NWDAF` 入口目前不做 `ip + startTime` 去重，只是 append
4. 到 inference 前，`analytics.go` 才發現同一個 30s slot center 內有兩筆，於是噴 `dedup collision`

因此這份報告的核心目標不是討論 `NWDAF` 怎麼降低 warning，而是把 source-side 的重複上報條件、程式位置、可能原因整理給 `go-upf` 團隊。

---

## 本次分析範圍

本文件以 `0505-18` testbed run 為準，主要依據：

- `nwdaf.log`
- `nwdafcfg.yaml`
- `go-upf-ess` 目前程式碼：
  - `pkg/app/app.go`
  - `internal/ees/pseudodriver.go`
  - `internal/ees/aggregator.go`
  - `internal/ees/notifier.go`
- `NWDAF` 目前程式碼：
  - `internal/sbi/processor/upf_notify.go`
  - `internal/anlf/analytics.go`
  - `internal/context/traffic_data.go`

本次不討論 `Daisy`、`ML service`、`retrain policy` 本身，只聚焦在：

1. `go-upf` 是否重複推送同一個量測 window  
2. 重複是在哪個時間段發生  
3. 為什麼 `NWDAF` 會因此出現大量 dedup warning
4. 在什麼條件下比較容易發生
5. `go-upf` 哪些程式段落最值得優先檢查

---

## 先說結論

### 結論 1

`go-upf` 在本次實驗中，**確實有把同一個 `ip + startTime` 推送兩次**。

不是推測，是 log 已直接證明。

### 結論 2

這個重複並不是在同一個 notification 內發生，而是：

- 第一次在 notification `N`
- 第二次在 notification `N+1`

也就是相鄰兩輪 notify 之間有重疊。

### 結論 3

`NWDAF` 的 `dedup collision` warning，是因為 `upf_notify.go` 把所有 item 都 append 到 `RawUpfData`，而 `analytics.go` 在 inference 前又會把同一 IP 的資料 round 到 `30s` grid，最後發現同一 slot center 出現兩筆，所以只能 `keeping later`。

### 結論 4

本次現象與 `go-upf` 的 **hybrid mode** 高度相關：

- `PseudoDriver` phase2 會把 replay window push 進 `Aggregator`
- kernel `URR` 也會把 live window push 進同一個 `Aggregator`
- `TickOnce()` 再把 buffer 裡所有 windows 一起送出

這使得相鄰週期之間，存在同一 logical slot 被重複帶出的可能。

### 結論 5

本次現象不是每次都必現，原因很可能不是固定資料 bug，而是**時間邊界競態**：

- `PseudoDriver` Phase 2 的 pacing 是跟 `Aggregator` tick 同步
- kernel `URR` 的進入時機則受真實 session、perio timer、goroutine scheduling 影響
- `TickOnce()` 在收到 `kernelReady` 後還會固定 sleep `50ms`
- tick 本身又有 `time.Now()` / ticker 抖動與 `lastTickTime` 的週期校正

因此只要某次「前一個 slot 的 measure 剛好在 tick 邊界前後跨輪」，就可能出現：

- 第 `N` 輪送出一次
- 第 `N+1` 輪又送一次

這也解釋了為什麼現象「以前常看過，但不一定每次都發生」。

---

## 什麼情況下比較容易發生

依本次程式碼與 log，這個現象最可能出現在以下條件同時成立時：

### 條件 1：EES 以 hybrid mode 啟動

在 `pkg/app/app.go` 中，只要：

- `EES.Enabled = true`
- `ParquetDir` 存在

就會同時建立：

- `PseudoDriver`
- `Aggregator`
- kernel report handler

也就是不是 pure live，也不是 pure replay，而是兩者並行。

### 條件 2：Pseudo replay 已進入 Phase 2

`PseudoDriver.LoadAndReplay()` 會把 replay 切成：

- `phase1`: `globalTS <= alignedBreakingTime`
- `phase2`: `globalTS > alignedBreakingTime`

Phase 1 是歷史 burst；真正會和 kernel 交錯的是 **Phase 2**。  
本次 dedup 大量出現時，系統已經在 Phase 2 很久了。

### 條件 3：同一 SUPI 已有 active session，kernel 也持續送 URR 2

`Aggregator.PushReport()` 只吃：

- `USAReport`
- `URRID == 2`

也就是只有當：

- UE 真的上線
- SMF/UPF session 存在
- kernel 真的在送 URR 2

時，live traffic 才會和 pseudo phase2 一起混進 buffer。

### 條件 4：同一個 UE IP 在 pseudo 和 kernel 路徑上都落到同一個 30s grid

本次是：

- `samplingInterval = 30`
- `UPF-EES periodSec = 30`

而 `NWDAF` 端最後是以 `30s` grid 在 `alignAndZipInMemory()` 裡做 dedup。  
只要 `go-upf` 連續兩輪把同一個 `startTime` 送出，`NWDAF` 就幾乎一定會撞到 dedup。

---

## 測試條件

### Group membership

本次 `NWDAF` group config 只保留每組 1 個 SUPI：

- `group-test-001` → `imsi-208930000000001`
- `group-test-002` → `imsi-208930000000004`

這代表：

- 每個 group 最終只會展開成 1 個 `SMF` subscription target
- 每個 correlation 只對應 1 個 UE IP

因此在本次 log 中，`NWDAF` AnLF 聚合時常看到：

- `Aggregated 30 global slots from 1 corrIds`
- `ipCount/slot:[1,1,1,...]`

也就是說，這次不是多個 IP 合併後互相覆蓋，而是**單一 IP 自己的資料序列就出現重複 slot**。

### Sampling / slot 長度

- `samplingInterval = 30`
- `UPF-EES periodSec = 30`
- pseudo replay `breaking time = 900`

因此本次 dedup 是以 **30s slot** 為單位觀察。

---

## 直接證據：同一個 `ip + startTime` 被上報兩次

### `10.10.0.1`

同一個 `startTime=2026-05-05T10:00:02Z`，在 log 中出現兩次：

```text
time="2026-05-05T10:00:23.117233869Z" ... UPF VOLUME: ip=10.10.0.1, startTime=2026-05-05T10:00:02Z, total=0 ...
time="2026-05-05T10:00:53.112480541Z" ... UPF VOLUME: ip=10.10.0.1, startTime=2026-05-05T10:00:02Z, total=0 ...
```

### `10.100.0.1`

同一個 `startTime=2026-05-05T10:00:03Z`，也在 log 中出現兩次：

```text
time="2026-05-05T10:00:25.464500341Z" ... UPF VOLUME: ip=10.100.0.1, startTime=2026-05-05T10:00:03Z, total=0 ...
time="2026-05-05T10:00:55.489428482Z" ... UPF VOLUME: ip=10.100.0.1, startTime=2026-05-05T10:00:03Z, total=0 ...
```

這已經足夠證明：

- 同一個 measurement window 不是只送一次
- 而是至少被送了兩次

這裡要特別強調：  
這不是「同一個 notification 裡有兩個不同 startTime」而已，而是**相同的 `startTime` 被不同 log time 的兩輪通知各送一次**。

---

## 重複發生的整體時間區間

依 `UPF VOLUME` log 解析，重複 key 有 `45` 個：

- `10.10.0.1`: 多個 `startTime` 各出現 `2` 次
- `10.100.0.1`: 多個 `startTime` 各出現 `2` 次

最早開始於：

- `10.10.0.1 @ 2026-05-05T09:59:32Z`
- `10.100.0.1 @ 2026-05-05T09:59:33Z`

之後持續到：

- 約 `2026-05-05T10:11:03Z`

換句話說，從大約 `09:59:32/33` 開始，後面幾乎每個 `30s` slot 都會被重複送進 `NWDAF`。

從 pattern 看，這不是少量偶發重複，而是進入某個狀態後，後續 slot 幾乎都開始受影響。

---

## `dedup collision` 在 NWDAF 端何時開始出現

`NWDAF` 最早的 dedup warning 出現在：

```text
time="2026-05-05T10:00:30.396714487Z" ... dedup collision ip=10.100.0.1: t=1777975173 ...
time="2026-05-05T10:00:30.396830671Z" ... dedup collision ip=10.10.0.1: t=1777975172 ...
```

其中：

- `1777975172` = `2026-05-05T09:59:32Z`
- `1777975173` = `2026-05-05T09:59:33Z`

也就是：

- `UPF` 先在 `09:59:32/33` 這批 slot 開始重複送
- `NWDAF` 到下一次做 inference 時，才在 `alignAndZipInMemory()` 看到衝突

### Warning 數量隨時間增加

按分鐘統計：

| 分鐘 | dedup count |
|---|---:|
| `10:00` | 2 |
| `10:01` | 10 |
| `10:02` | 18 |
| `10:03` | 26 |
| `10:04` | 34 |
| `10:05` | 42 |
| `10:06` | 50 |
| `10:07` | 58 |
| `10:08` | 65 |
| `10:09` | 72 |
| `10:10` | 80 |
| `10:11` | 88 |

總計：

- `10.10.0.1`: `276`
- `10.100.0.1`: `269`

這個型態很像：

- ring buffer 裡的重複 slot 越積越多
- 每次 inference 都對最近 `inputWindow=30` 做一次 anchor-relative dedup
- 所以 warning 會隨時間累積

---

## 為什麼 NWDAF 會噴 dedup

### `upf_notify.go` 不做去重

`NWDAF` 收到 UPF notification 後，會把每個 item 直接 append 到對應 IP 的 `RawUpfData`：

```go
data.RawUpfData = append(data.RawUpfData, dataPoint)
```

這段在 `upf_notify.go`。

它不會先檢查：

- 同一個 `ip`
- 同一個 `startTime`

是否已經存在。

### `analytics.go` 在 inference 前才發現衝突

`alignAndZipInMemory()` 會：

1. 取該 IP 第一筆資料的 timestamp 當 `anchor`
2. 依 `samplingInterval=30s` 把每一筆資料 round 到最近的 grid center
3. 若兩筆資料落到同一個 center，就記 warning 並保留較晚那筆

也就是這段：

```go
if _, exists := perIP[centerUnix]; exists {
    anlfLog.Warnf(\"dedup collision ... keeping later\")
}
perIP[centerUnix] = dp
```

所以 `dedup collision` 不是 root cause，而是 symptom。

也就是說，`NWDAF` 在這裡只是最後一個看見問題的人。  
真正值得追的是：`go-upf` 為什麼會讓相同 `ip + startTime` 穿過 notify 邊界重複出現。

---

## `go-upf` 內完整的資料路徑

下面把本次相關程式路徑按實際執行順序串起來。

### 1. app 啟動 EES hybrid mode

檔案：

- `pkg/app/app.go`

流程：

1. 建立 `SubscriptionStore`
2. 建立 `Notifier`
3. 若 `ParquetDir` 存在，建立 `PseudoDriver`
4. 建立 `Aggregator`
5. 把 `PseudoDriver` 注入 `Aggregator`
6. 建立 `EES Handler`
7. `Dispatcher.RegisterEESHandler(eesHandler, aggregator)`
8. 啟動 `aggregator.Run(...)`
9. 啟動 `EES API Server`

這一步決定了：後面所有 pseudo data 與 kernel URR 都會匯入同一個 `Aggregator.reportBuffer`。

### 2. SMF 建立 UPF 訂閱後，go-upf 建立 EES subscription

檔案：

- `internal/ees/api.go`

`handleCreateSubscription()` 會：

1. 驗證 request
2. `subscriptionStore.CreateSubscription(...)`
3. 若 `pseudoDriver != nil`
   - `go pseudoDriver.LoadAndReplay(storedSub)`
4. 若是 on-demand 才立即 tick

也就是 **每個 EES subscription 建立後，PseudoDriver 會為該 subscription 啟動一條 replay goroutine**。

### 3. PseudoDriver 讀 parquet，切成 Phase 1 / Phase 2

檔案：

- `internal/ees/pseudodriver.go`

重點步驟：

1. 從 `file.json` 讀 `breaking time`
2. 用 `alignedBreakingTime := ceil(breakingTime / period) * period`
3. 把 packet stream 依 `globalTS <= alignedBreakingTime` 分成：
   - `phase1Accum`
   - `phase2Accum`
4. 等第一個 URR 到來後，用 URR time 建立 `GridAnchor`

這裡決定了 Phase 2 的 pseudo window 時間軸會主動對齊 kernel 第一個 URR。

### 4. Phase 1 / Phase 2 都不是直接送 HTTP，而是先進 shared buffer

檔案：

- `internal/ees/pseudodriver.go`
- `internal/ees/aggregator.go`

Phase 1：

- `LoadAndReplay()` 內會呼叫 `aggregator.PushHistoricalMeasures(sub, measures, logicalTime)`
- `PushHistoricalMeasures()` 只是：
  - `reportBuffer[sub.ID] = append(reportBuffer[sub.ID], measures...)`

Phase 2：

- `simulateFutureRealTime()` 每輪先 `WaitForTick()`
- 再把當前 pseudo window 透過 `PushLiveMeasures()` 丟進 `reportBuffer`

因此 pseudo replay 不論 phase1 或 phase2，都不是直接 notify，而是**先進 buffer，等 TickOnce 送出**。

### 5. kernel report 也進同一個 buffer

檔案：

- `internal/ees/handler.go`
- `internal/ees/aggregator.go`

流程：

1. dispatcher 收到 kernel report
2. `EES Handler.NotifySessReport(...)`
3. `aggregator.PushReport(sessRpt)`

而 `PushReport()`：

- 只取 `URRID == 2`
- 把 session 的 counters 轉成 `UsageMeasures`
- 之後 append 進同一個 `reportBuffer[sub.ID]`

也就是 pseudo 與 kernel 最後會在 **同一個 `reportBuffer`** 匯流。

### 6. kernel 在 pseudo mode 下還會被強制對齊到 current phase2 window

檔案：

- `internal/ees/aggregator.go`
- `internal/ees/pseudodriver.go`

`PushReport()` 會透過 `pseudoDriver.GetPhase2Window()` 取得目前 Phase 2 的 logical window。  
當 pseudo mode active 時，kernel report 的 `StartTime/EndTime` 會被改寫到這個 Phase 2 window 上。

這一步非常關鍵，因為它代表：

- kernel 不只是「自然帶自己的真實時間」
- 而是主動被 snap 到 pseudo 的 window 語意

因此只要 pseudo phase2 的 window publish 與 tick/flush 邊界有重疊，就更容易出現「上一個 logical slot 又被帶一次」。

### 7. TickOnce 把 buffer 整包送出

檔案：

- `internal/ees/aggregator.go`
- `internal/ees/notifier.go`

`TickOnce()` 的流程：

1. swap 出目前整個 `reportBuffer`
2. 對每筆 measure 做 time snapping
3. `consolidateReports()` 以：
   - `key = ueIp + StartTime.Unix()`
   做同輪內合併
4. `notifier.Notify(...)` 發 HTTP

注意：

- `consolidateReports()` 只負責**同一輪 TickOnce 內**的合併
- 它**無法處理「上一輪已送過、下一輪又來一次」**的情況

這正是本次問題的核心。

---

## 為什麼問題源頭更像在 `go-upf`

因為本次 log 已經顯示：

- `Received UPF notification, items: 2`
- 同一個 IP 的某個 `startTime` 在相鄰兩輪 notify 中反覆出現

例如 `corr-1` 在 `10:00:53` 的 notify：

```text
Received UPF notification, items: 2
UPF VOLUME: ip=10.10.0.1, startTime=2026-05-05T10:00:02Z ...
UPF VOLUME: ip=10.10.0.1, startTime=2026-05-05T10:00:32Z ...
```

但 `startTime=2026-05-05T10:00:02Z` 在 `10:00:23` 那輪其實已經送過一次。

同樣情況在 `corr-2` 也成立：

```text
Received UPF notification, items: 2
UPF VOLUME: ip=10.100.0.1, startTime=2026-05-05T10:00:03Z ...
UPF VOLUME: ip=10.100.0.1, startTime=2026-05-05T10:00:33Z ...
```

而 `startTime=2026-05-05T10:00:03Z` 也在前一輪送過。

這說明：

- `NWDAF` 並不是把同一個 item 重複解析兩次
- 而是 `UPF notify` 本身就把同一個 logical window 又送了一次

換句話說，若只看 `NWDAF` 入口，問題已經太晚了。  
在 `NWDAF` 看見重複之前，source notification 就已經重複。

---

## `go-upf` 目前的輸出模式

依程式碼，這次不是 pure replay，而是 **hybrid mode**。

### `app.go`

只要：

- `EES.Enabled = true`
- `ParquetDir` 存在

就會同時啟動：

- `PseudoDriver`
- `Aggregator`
- kernel report handler

### `PseudoDriver`

`PseudoDriver.LoadAndReplay()` 會把資料拆成兩段：

- `phase1`: `globalTS <= alignedBreakingTime`
- `phase2`: `globalTS > alignedBreakingTime`

### Phase 1

phase1 不直接送 HTTP，而是用：

- `PushHistoricalMeasures()`

把 replay window 放進 shared buffer。

### Phase 2

phase2 也不直接送 HTTP，而是：

- `WaitForTick()`
- `PushLiveMeasures()`

把 pseudo window 送進 shared buffer。

### Kernel URR

同時，kernel 量測也會透過：

- `PushReport()`

進入同一個 `reportBuffer`。

### TickOnce

最後 `TickOnce()` 才：

1. snap 時間
2. `consolidateReports()`
3. `Notify()`

所以最終送給 `NWDAF` 的不是純 pseudo，也不是純 live，而是：

- pseudo phase2 window
- kernel URR

在同一個 buffer 裡的混合結果。

這個設計本身不是錯；真正可疑的是：

- **混合是預期的**
- **相鄰兩輪對相同 `startTime` 重複帶出不是預期的**

---

## 最可能的問題位置

就目前證據來看，最可疑的是：

### 1. 相鄰兩輪 notify 對同一個 logical slot 有重疊

也就是：

- 某個 slot 在 notification `N` 已送出
- 下一輪 `N+1` 又被一起帶出

這會直接產生：

- 相同 `ip + startTime`
- 不同 log time

這是本次最直接、也最已被 log 證實的異常型態。

### 2. hybrid phase2 與 kernel URR 的交錯邊界

`go-upf` 現在是：

- pseudo phase2 依 aggregator tick pacing
- kernel URR 依實際 URR timer 推入

只要這兩者在某個 period boundary 上沒有完全對齊，就可能發生：

- 前一輪的 slot 尚未完全從 buffer 消化
- 下一輪又再帶出一次

這裡尤其要注意 `simulateFutureRealTime()` 的順序：

- 先 `WaitForTick()`
- 再 publish 新的 `phase2StartTime/phase2EndTime`
- 再 `PushLiveMeasures()`

而 `Run()` 的 tick loop 則是：

- ticker 到
- 等 `kernelReady` 或 timeout
- `sleep 50ms`
- `TickOnce()`

這兩條 goroutine 的交錯時序，就是最值得檢查的競態區。

### 3. `items: 2` 的模式在後段穩定出現

本次重複最明顯的時間段，恰好就是：

- traffic 幾乎都已經降成 0
- 但 notify 仍規律帶兩個 item

這讓「前一 slot + 當前 slot 一起送」的模式更容易被看見。

換句話說，`items: 2` 本身不等於 bug；bug 在於：

- 其中一個 item 的 `startTime`
- 在前一輪其實已經送過了

---

## 不是什麼原因

以下幾點目前可先排除：

### 1. 不是 `NWDAF` config 設錯

`0505-18` 的 `nwdafcfg.yaml` 和 replay 主線的核心 monitor 參數已基本對齊。

### 2. 不是 dataset 已經跑完

本次異常開始時，整體 dataset 時間軸尚未走到尾端。

### 3. 不是單純因為 `NWDAF` 去重策略太敏感

即使 `NWDAF` 完全不做 dedup warning，source 側重複上報仍然存在。  
`NWDAF` 只是把它顯性化。

---

## 目前最合理的解釋

本次最合理的整體解釋是：

1. `go-upf` 使用 hybrid mode
2. `PseudoDriver` phase2 與 kernel URR 都往同一個 `reportBuffer` 塞資料
3. 某些相鄰週期中，同一個 `ip + startTime` 被分別帶進兩輪 notify
4. `NWDAF` 直接 append 到 `RawUpfData`
5. inference 前才做 dedup，因此出現大量 `dedup collision`

這也解釋了為什麼：

- 現象常見但不穩定
- 有時完全不發生
- 有時某段時間才開始大量出現

因為這更像是：

- tick 邊界
- phase2 pacing
- kernel URR 到達時機

共同造成的時間競態，而不是單一 deterministic bug。

可以把它理解成：

- `go-upf` 目前邏輯允許「多來源 measure 在同一輪 TickOnce 內合併」
- 但在某些時序下，它還會讓「上一輪已送出的 slot」在下一輪又再次出現
- 這不是 `consolidateReports()` 這種單輪內合併可以解的問題

---

## 建議優先檢查的點

### 1. notification 內容是否允許重帶上一個 slot

先確認 `TickOnce()` 送出的 notification，是否可能在相鄰兩輪之間包含同一個 `StartTime`。

如果答案是會，就應先決定：

- 這是否為預期行為
- 如果不是，應在 `Aggregator` 哪一層避免重複送出

建議直接在 `Notify()` 前加暫時性 debug：

- dump 每輪 `subscriptionId`
- dump 每個 item 的 `ueIp + startTime + endTime`
- 比較相鄰兩輪是否有完全相同的 `startTime`

### 2. phase2 pseudo 與 kernel URR 的窗口邊界

建議重點檢查：

- `simulateFutureRealTime()`
- `PushReport()`
- `GetPhase2Window()`
- `TickOnce()`

尤其是：

- phase2 window 何時 publish
- kernel report 何時被對齊到 phase2 window
- 哪些 measure 會留到下一輪

建議優先看：

- `PseudoDriver.simulateFutureRealTime()`
- `Aggregator.PushReport()`
- `Aggregator.PushLiveMeasures()`
- `Aggregator.TickOnce()`

### 3. buffer drain 後是否還會重新引入前一 window

這次看起來很像：

- 前一個 slot 在 notify `N` 已送出
- notify `N+1` 又帶了一次

這表示應檢查：

- `reportBuffer` 在 tick 後是否確實只保留應保留的資料
- pseudo / kernel 是否在下一輪又重新產生了同樣 `startTime` 的 measure

這一點尤其可能出現在：

- 前一輪 tick 已經把 slot `T` 送出
- 下一輪 kernel report 又因 phase2 window 對齊規則，被映射回 `T`
- 或 pseudo phase2 在 publish / push 時，又再次產生 `T`

### 4. 若這是預期設計，至少應在 source 端做去重

若 hybrid mode 的設計本來就允許相鄰兩輪出現同樣 `startTime`，那就應考慮在較早的層次做去重，而不是把重複 item 送到 NWDAF 端再靠 inference 前的 last-wins 解決。

---

## 可行的修法方向

### 方向 A：修 `go-upf`

優先度最高。

目標：

- 不要讓同一個 `ip + startTime` 在相鄰兩輪 notification 中重複出現

這是最乾淨的修法。

### 方向 B：在 `NWDAF upf_notify` 入口做 `ip + startTime` 去重

這能減少症狀，但不能解決 source 端重複上報。

適合作為保險，但不應取代 source 修正。

### 方向 C：保留現狀，只降低 warning 噪音

若短期無法改 `go-upf`，至少可把：

- `dedup collision`

改為更低 log level，避免掩蓋真正重要的 warning。

但這只是降噪，不是修正。

---

## 總結

這次 `0505-18` 的分析已經可以很明確地說：

- `NWDAF` 端看到的 `dedup collision`，源頭不是純推論，而是 `go-upf` 確實送了重複的 `ip + startTime`
- 重複發生在相鄰兩輪 notification 之間
- 現象與 hybrid mode 的 pseudo phase2 + kernel URR 交錯高度相關

若要讓 `go-upf` 團隊快速開始追，最值得先下手的位置是：

1. `internal/ees/pseudodriver.go`
   - `LoadAndReplay()`
   - `simulateFutureRealTime()`
2. `internal/ees/aggregator.go`
   - `PushLiveMeasures()`
   - `PushReport()`
   - `TickOnce()`
   - `consolidateReports()`
3. `internal/ees/notifier.go`
   - 在送 HTTP 前 dump item keys，確認相鄰兩輪重複是否已在 source 端形成

如果後續要把問題真正解掉，最值得優先處理的是：

- **`go-upf` 端避免相鄰兩輪 notification 對同一 logical slot 重複上報**

在此之前，`NWDAF` 端的 dedup 只能算是補救，不是根治。
