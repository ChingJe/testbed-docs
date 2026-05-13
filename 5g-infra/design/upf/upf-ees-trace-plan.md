# UPF EES Trace Plan

## Purpose

這份文件整理 `go-upf` EES 路徑的 trace 與診斷開關規劃，目的是把目前對 `UPF notify` 與 `pre_data` mismatch 的判斷，從「高可信度推論」提升到更接近「直接證據」。

目前已知現象是：

- `pseudo-driver` phase1 / phase2 對齊看起來正常
- `dispatcher` 路徑的 kernel report 也持續進來
- `test-EES-with-pseudodriver` branch 會把不同來源的同 slot measure 直接 `+=`
- 最終 `UPF notify` 並不是對所有 IP 都 exact 等於 `pre_data`

但目前仍缺少一種逐筆對帳能力：

- 對某個 `ueIp + startTime` window
- 能明確看到 `pseudo contribution`
- 能明確看到 `kernel contribution`
- 能明確看到 `final notify`

這份規劃目前分成兩個主要目的：

1. 補上詳細 trace log，從 log 觀察問題點
2. 補上一個暫時性的 `pseudo-only notify` 診斷模式，單獨測試 pseudo-driver 流量

目前狀態是：

- **trace/log 能力：已實作**
- **pseudo-only notify 診斷模式：已實作**

## Scope

目前這份計畫包含兩類修改：

1. trace / log 補強
2. `pseudo-only notify` 診斷開關

目前兩類都已實作。

也就是說：

- 目前還**沒有**把系統改成 pseudo-only
- 也還**沒有**改成 kernel overwrite pseudo
- 也還**沒有**改 NWDAF collector

目前已完成的部分包括：

- `go-upf` 可用 trace log 說清楚資料來源與 merge 結果
- notify 可透過診斷開關暫時只採用 pseudo-driver 資料

## Pseudo-only diagnostic mode

除了純 trace 之外，這份規劃也需要一個**暫時性的診斷模式**，目的是把目前的問題拆成兩個主要工作：

1. 先從 trace log 觀察問題點，確認 `kernel / pseudo / final notify` 的資料路徑
2. 再讓 notify 端**只送 pseudo-driver 流量**，單獨測試 pseudo-driver 是否能讓 NWDAF 行為回到更接近 replay / `pre_data` 預期的狀態

也就是說，這個模式不是為了直接修產品行為，而是為了做更乾淨的 `pseudo-only` 對照實驗。

### Intended behavior

建議新增一個可切換的診斷開關，例如：

- `EES_PSEUDO_ONLY_NOTIFY=true`

目前實作就是採用這個環境變數開關，而且**預設全域開啟**：

- 不設環境變數：視為 `true`
- 若真的要回到 fused mode，才顯式設：
  - `EES_PSEUDO_ONLY_NOTIFY=false`

開啟後的預期行為是：

- `dispatcher` / `kernel report` 仍然照常進入 EES
- `PushReport()` 仍然照常 parse 出 `UsageMeasures`
- `kernel` 路徑的 trace log 仍然照常保留
- 但在真正進入 notify 用的 aggregation buffer 前，`kernel` contribution 會被標記為 ignored，不再參與最終 notify
- `pseudo history` 與 `pseudo phase2` 則仍照常進入 buffer 與 notify

換句話說：

- **觀察層面**：保留 `kernel` 與 `pseudo` 兩條路徑的 trace
- **notify 行為層面**：只讓 `pseudo-driver` 的資料進入最終輸出

這個行為是對所有符合 subscription 的 IP 一體生效，不需要另外逐 IP 設定。

### Why this mode is useful

這個模式的主要用途是：

#### 驗證 NWDAF 行為是否因 kernel fusion 而偏移

如果開啟 `pseudo-only notify` 後：

- `UPF notify` 與 `pre_data` 重新回到 exact / near-exact
- NWDAF 的 trigger / monitor 行為也回到接近 replay 預期

那就能非常強地支持：

- 先前的偏差主要來自 `kernel live URR` 混入 notify

這裡的重點始終是：

- **主要目標**：把 notify 切成 pseudo-only，驗證 NWDAF 是否恢復預期
- **附帶要求**：不要把 trace 能力弄丟

也就是說，保留 `kernel` 路徑 log 是為了不影響第 1 項 trace 目標，不是另一個獨立主問題。

### Expected trace behavior in pseudo-only mode

開啟 `EES_PSEUDO_ONLY_NOTIFY=true` 後，trace 預期應該出現：

- `kernel_push_report`
  - 代表 kernel 路徑真的有進來
- `kernel_report_ignored_pseudo_only`
  - 代表這筆 kernel usage 被刻意排除在 notify 之外
- `pseudo_history_push`
- `pseudo_phase2_push`
- `merge_seed` / `merge_accumulate`
  - 但此時 merge 理論上只會看到 pseudo source
- `notify_candidate`
  - final candidate 應與 pseudo path 對齊
- `notify_dispatch`
  - 送出的 bytes 應只反映 pseudo source

### Suggested implementation point

最小改法建議放在：

- `internal/ees/aggregator.go`
- `PushReport()`

流程建議是：

1. 正常把 kernel `SessReport` 轉成 `UsageMeasures`
2. 正常印出 `kernel_push_report`
3. 若 `pseudo-only` flag 開啟：
   - 再印 `kernel_report_ignored_pseudo_only`
   - 然後不要 append 到 notify buffer
4. 若 flag 關閉：
   - 維持現行 fused 行為

這樣修改面最小，而且最容易確保：

- trace 還在
- notify 被切乾淨

### Interpretation of results

這個診斷模式跑完後，可用下列方式解讀：

- 若 `pseudo-only notify` 後 mismatch 消失：
  - 幾乎可直接把 root cause 收斂到 `kernel contribution mixed into notify`
- 若 `pseudo-only notify` 後 mismatch 仍存在：
  - 問題就不只在 kernel fusion
  - 應往 pseudo path 自身、slot alignment、aggregation key、或 notify payload 組裝再查

因此，這個模式是把目前的判斷從：

- 「高度懷疑 kernel 路徑造成偏差」

往前推進到更接近：

- 「經 pseudo-only 對照後，可直接驗證 kernel fusion 是否是主因」

## Trace run workflow

除了 `go-upf` 程式內的 trace log，本次規劃也需要一套實際操作層的 trace run 腳本，讓下次重跑時能同時保留：

- `UPF EES debug log`
- `gNB / UE log`
- `N3` 介面的 `pcap`

建議做法是：

- 保留原本 `run.sh` 當一般啟動腳本
- 額外新增 `run-trace.sh`，只在 trace / root-cause 實驗時使用

這樣可以避免：

- 一般日常使用被 `pcap` / trace log 汙染
- trace 腳本意外改變原本工作流

### Proposed trace scripts

建議新增四個腳本：

- `5G_Infrastructure/UPF-EES/run-trace.sh`
- `5G_Infrastructure/UPF-EES2/run-trace.sh`
- `5G_Infrastructure/gNB/run-trace.sh`
- `5G_Infrastructure/gNB2/run-trace.sh`

另外也應各自新增配對的收尾腳本：

- `5G_Infrastructure/UPF-EES/clean-trace.sh`
- `5G_Infrastructure/UPF-EES2/clean-trace.sh`
- `5G_Infrastructure/gNB/clean-trace.sh`
- `5G_Infrastructure/gNB2/clean-trace.sh`

### Trace script goals

每個 `run-trace.sh` 都應做到：

1. 建立帶 timestamp 的 trace 目錄
2. 啟動主要程序
3. 同時啟動 `tcpdump`
4. 把 log 固定落盤
5. 另外以非阻塞方式背景 `tail -F` 該 trace log，讓原本 terminal 仍可持續看到輸出
6. 保留簡單的 pid / 路徑資訊，方便事後抓取

每個 `clean-trace.sh` 都應做到：

1. 優先讀取最新 `trace-*` 目錄內的 `*.pid`
2. 停止主要程序
3. 停止 `tcpdump`
4. 停止背景 `tail -F`
5. 若 `pid` 檔缺失，退回用 process pattern 搜尋補清
6. 列出殘留的 trace 相關 process，方便確認實驗是否真的收尾完成

### Expected output layout

#### UPF-EES / UPF-EES2

建議輸出到：

- `~/free5gc/log/trace-<timestamp>/`

至少包含：

- `upf.log`
- `tcpdump.log`
- `n3.pcap`
- `upf.pid`
- `tcpdump.pid`

#### gNB / gNB2

建議輸出到：

- `~/UERANSIM/logs/trace-<timestamp>/`

至少包含：

- `gnb.log` / `gnb2.log`
- `ue1.log` ... `ue6.log`
- `tcpdump.log`
- `n3.pcap`
- `gnb.pid`
- `ue*.pid`
- `tcpdump.pid`

### Interface and capture filter

目前 trace 目標是 `N3` 上的 `GTP-U`，因此建議：

- 介面：`enp0s8`
- filter：`udp port 2152`

也就是：

```bash
sudo tcpdump -i enp0s8 udp port 2152 -n -w ...
```

### Relationship with `EES_TRACE_IPS`

`run-trace.sh` 不需要自己硬寫某組 IP，但應該：

- 尊重外部傳入的 `EES_TRACE_IPS`
- 並在啟動時把目前 `EES_TRACE_IPS` 印出來

這樣可以支援兩種模式：

- 不設 `EES_TRACE_IPS`：追全部 IP
- 設 `EES_TRACE_IPS=10.10.0.3,10.100.0.3`：只追指定 IP

### Relationship with debug log level

`run-trace.sh` 不應該偷偷改 config 檔內容，但必須假設：

- `UPF-EES.yaml`
- `UPF-EES2.yaml`

已經是 `debug` level。

也就是說，trace 實驗的前提仍然是：

- **先把 UPF config 調成 `debug`**
- **再用 `run-trace.sh` 啟動**

### Why trace scripts are needed even after code trace is added

程式內 trace log 只能回答：

- 哪些 bytes 經過哪些 code path

但若要確認：

- phase2 期間 `N3` 上到底有沒有真實 GTP-U

仍然需要 `pcap`。

因此這次的完整證據鏈預期是：

1. `gNB / UE log`
2. `UPF EES debug trace`
3. `N3 pcap`
4. `NWDAF UPF VOLUME log`

四者對時後，才能比較完整地說明：

- 是 pseudo-only
- kernel contribution
- 還是真的有 packet on wire

## Runtime requirement

這份 trace 計畫有一個明確前提：

- **重跑 trace 實驗時，UPF EES 必須以 `debug` log level 啟動**

原因是：

- 多數新增 trace 點會建議放在 `debug` level
- 若仍以 `info` 啟動，很多細粒度 trace 會被直接吃掉

因此實作完成後，實驗時應確認：

- `UPF-EES.yaml` / `UPF-EES2.yaml` 的 logger level 為 `debug`
- 若 `run.sh` 會覆蓋或複製 config，也要確認最後實際生效的 config 仍是 `debug`

如果之後需要把 trace 做成可切換模式，也建議維持這個約定：

- 一般穩定運行：`info`
- trace / root-cause run：`debug`

## Files In Scope

- `5G_Infrastructure/go-upf-ess/go-upf/internal/ees/handler.go`
- `5G_Infrastructure/go-upf-ess/go-upf/internal/ees/aggregator.go`
- `5G_Infrastructure/go-upf-ess/go-upf/internal/ees/notifier.go`
- 視需要補：
  - `5G_Infrastructure/go-upf-ess/go-upf/internal/ees/types.go`
  - `5G_Infrastructure/go-upf-ess/go-upf/internal/ees/api.go`

## Trace goals

補完後應該能直接回答：

1. `dispatcher` 路徑到底對哪個 `ueIp`、哪個 `slot` 產生了多少 usage？
2. `pseudo-driver` phase2 對同一個 `ueIp`、同一個 `slot` 又放了多少 usage？
3. `consolidateWithPriority()` 是否真的把兩者加總？
4. `final notify` 送出去的數字是否等於前兩者的和？

## Shared field set

所有 trace log 儘量統一用同一組欄位，方便後續 grep / 對帳：

- `path`
- `subscriptionId`
- `seid`
- `ueIp`
- `source`
- `startTime`
- `endTime`
- `ulBytes`
- `dlBytes`
- `ulPkts`
- `dlPkts`

視情況再加：

- `reportCount`
- `windowIndex`
- `mergedUlAfter`
- `mergedDlAfter`
- `lastSentStartTime`
- `reason`

## Proposed trace points

### 1. Dispatcher entry

檔案：

- `internal/ees/handler.go`

函式：

- `NotifySessReport(sessRpt report.SessReport)`

目的：

- 保留最前面的 kernel report 入口摘要
- 讓後面 `PushReport()` 的細節 log 可以回頭對應到哪個 `SessReport`

建議欄位：

- `path=dispatcher_entry`
- `seid`
- `reportCount`
- `reportTypes`
- `urrIds`
- `firstReportStartTime`
- `lastReportEndTime`

建議等級：

- `debug`

回答的問題：

- kernel/dispatcher 路徑當下是否真的有 report 進來
- 一次進來的大概是哪個時間窗

### 2. Kernel usage after extraction

檔案：

- `internal/ees/aggregator.go`

函式：

- `PushReport(sessRpt report.SessReport)`

目的：

- 這裡是 `SessReport -> UsageMeasures` 的關鍵轉換點
- 需要把最終從 kernel 路徑抽出的 usage 明確印出來

建議欄位：

- `path=kernel_push_report`
- `subscriptionId`
- `seid`
- `ueIp`
- `source=KERNEL`
- `reportCount`
- `startTime`
- `endTime`
- `ulBytes`
- `dlBytes`
- `ulPkts`
- `dlPkts`
- `gridAnchor`

另外在被 filter 掉時建議補：

- `path=kernel_report_dropped`
- `subscriptionId`
- `seid`
- `ueIp`
- `startTime`
- `endTime`
- `reason=before_grid_anchor`

建議等級：

- `debug`

回答的問題：

- kernel 路徑到底對哪個 IP、哪個 window 有多少 contribution
- 是否有些 report 根本在進 buffer 前就被丟掉了

### 3. Pseudo phase1 history push

檔案：

- `internal/ees/aggregator.go`

函式：

- `PushHistoricalMeasures(sub *Subscription, measures []UsageMeasures, logicalTime time.Time)`

目的：

- 記錄 warm-start history 灌入了哪些資料
- 雖然主問題在 phase2，但 phase1 也需要能對帳

建議欄位：

- `path=pseudo_history_push`
- `subscriptionId`
- `count`
- `ueIp`
- `source=PSEUDO`
- `startTime`
- `endTime`
- `ulBytes`
- `dlBytes`
- `logicalTime`

建議等級：

- `debug`

回答的問題：

- phase1 history 到底送了哪些 window 進 buffer

### 4. Pseudo phase2 push

檔案：

- `internal/ees/aggregator.go`

函式：

- `PushLiveMeasures(sub *Subscription, measures []UsageMeasures)`

目的：

- 這是 phase2 pseudo window 真正進 `reportBuffer` 的點
- 必須和 `PushReport()` 並列對照

建議欄位：

- `path=pseudo_phase2_push`
- `subscriptionId`
- `ueIp`
- `source=PSEUDO`
- `startTime`
- `endTime`
- `ulBytes`
- `dlBytes`
- `ulPkts`
- `dlPkts`
- `warmupPending`

建議等級：

- `debug`

回答的問題：

- phase2 pseudo window 對每個 IP、每個 slot 到底放了多少值

### 5. Merge / accumulation

檔案：

- `internal/ees/aggregator.go`

函式：

- `consolidateWithPriority(reports []UsageMeasures)`

目的：

- 這是整個 trace 計畫最重要的點
- 要直接證明同一個 key 是否發生了 `PSEUDO + KERNEL`

建議拆成兩種 log：

#### 5.1 Seed log

第一次建立 key 時：

- `path=merge_seed`
- `key=<ueIp-startTime>`
- `source`
- `ueIp`
- `startTime`
- `endTime`
- `ulBytes`
- `dlBytes`

#### 5.2 Accumulate log

同一 key 再有第二筆以上進來時：

- `path=merge_accumulate`
- `key`
- `ueIp`
- `startTime`
- `existingSource`
- `incomingSource`
- `existingUlBefore`
- `existingDlBefore`
- `incomingUl`
- `incomingDl`
- `mergedUlAfter`
- `mergedDlAfter`

若一個 key 可能累加超過兩次，也可補：

- `mergeCount`

建議等級：

- `debug`

回答的問題：

- 哪些 slot 真的發生了 multi-source accumulation
- 最終 merged 值是不是剛好等於兩筆輸入的和

### 6. Final notify candidate

檔案：

- `internal/ees/aggregator.go`

函式：

- `TickOnce(ctx context.Context)`

目的：

- 看最終送出去前，每個 candidate item 的實際數值
- 與 `LastSentStartTime` / duplicate filter 一起看

建議欄位：

- `path=notify_candidate`
- `subscriptionId`
- `ueIp`
- `startTime`
- `endTime`
- `ulBytes`
- `dlBytes`
- `ulPkts`
- `dlPkts`
- `lastSentStartTime`
- `decision=send`

若被 duplicate filter 掉，建議補：

- `path=notify_drop_duplicate`
- `subscriptionId`
- `ueIp`
- `startTime`
- `lastSentStartTime`
- `reason=already_sent`

建議等級：

- `debug`

回答的問題：

- 哪些值真的進入最終 notify payload
- 哪些值只是進了 buffer，最後沒送

### 7. Final notifier summary

檔案：

- `internal/ees/notifier.go`

函式：

- `Notify(subscription *Subscription, measures []UsageMeasures)`

目的：

- 現在只有 `items=N`
- 對 trace 來說不夠

建議欄位：

- `path=notify_dispatch`
- `subscriptionId`
- `notifUri`
- `itemCount`
- `ueIps`
- `startTimes`
- `totalUlBytes`
- `totalDlBytes`

建議不要把完整 JSON payload 全印出來，避免 log 太重。

建議等級：

- `debug`

回答的問題：

- 每一輪 notify 大概送了哪些 UE / 哪些時間窗

## Trace control

不建議所有情況都把細粒度 trace 全開，否則 log 量會很大。

建議加入 trace 開關，例如：

- `TraceEnabled bool`
- `TraceUEIPs []string`
- `TraceOnlyPhase2 bool`

若暫時不想動 config schema，也可以先用 env var：

- `EES_TRACE_IPS=10.10.0.3,10.100.0.3`

優先只追：

- `10.10.0.2`
- `10.10.0.3`
- `10.100.0.2`
- `10.100.0.3`

這樣 log 量會可控很多。

## Suggested helper

建議在 `aggregator.go` 補一個小 helper，避免每個函式都重複堆 `WithFields(...)`：

- `shouldTraceUE(ip string) bool`
- `logUsageMeasure(path string, subID string, m UsageMeasures, extra logrus.Fields)`

好處：

- 欄位風格一致
- 比較不容易漏欄位
- 之後若要改 trace 開關，集中處理比較容易

## Recommended implementation order

若分批做，建議優先順序如下：

1. `PushReport()`
2. `PushLiveMeasures()`
3. `consolidateWithPriority()`
4. `TickOnce()`
5. `Notifier`
6. `Handler`
7. `PushHistoricalMeasures()`

其中 1~3 做完，就已經足夠回答核心問題：

- kernel 路徑到底有沒有值
- pseudo phase2 放了什麼值
- merge 時是否真的相加

## Expected outcome

若照這份規劃補完，下次重跑後應能直接對某個異常 slot 做出這種對帳：

- `ueIp=10.10.0.3`
- `startTime=2026-05-12T09:56:39Z`
- `pseudo window = UL x / DL y`
- `kernel window = UL a / DL b`
- `merged result = UL x+a / DL y+b`
- `final notify = merged result`

做到這一步，對 `UPF notify` mismatch 的根因判斷，就可以從：

- 高可信度推論

提升到：

- 逐筆可驗證的直接證據
