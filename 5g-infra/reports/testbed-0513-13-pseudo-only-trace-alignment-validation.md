# testbed 0513-13: pseudo-only trace alignment validation

## Purpose

這份報告整理 `0513-13` trace run 的主要結果，目的是回答兩個問題：

1. 依照 [upf-ees-trace-plan.md](/home/chingje/testbed/docs/5g-infra/design/upf/upf-ees-trace-plan.md) 的設計，這次 `pseudo-only notify` 是否真的生效？
2. 如果 final notify 只採用 pseudo-driver 流量，`UPF VOLUME` 是否會回到更接近 `pre_data` 的狀態？

這次也一併檢查：

- `kernel` 路徑在 trace log 中實際長什麼樣子
- `tail` 區段在 dataset 播完後是否會回到 0 流量

## Data sources

- trace run directory:
  - `5G_Infrastructure/.agent/tmp/0513-13/`
- verification tool:
  - [verify_upf_pseudo_alignment.py](/home/chingje/testbed/5G_Infrastructure/.agent/verify_upf_pseudo_alignment.py)
- core logs:
  - [nwdaf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/nwdaf.log)
  - [UPF-EES/upf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES/upf.log)
  - [UPF-EES2/upf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES2/upf.log)
- tcpdump summaries:
  - [UPF-EES/tcpdump.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES/tcpdump.log)
  - [UPF-EES2/tcpdump.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES2/tcpdump.log)
  - [gNB/tcpdump.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/gNB/tcpdump.log)
  - [gNB2/tcpdump.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/gNB2/tcpdump.log)
- replay source:
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet`
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet`
- comparison method reference:
  - [testbed-0512-19-upf-notify-vs-predata-method.md](/home/chingje/testbed/docs/5g-infra/reports/testbed-0512-19-upf-notify-vs-predata-method.md)

## Verification workflow used in this report

這份報告最後的修正結論，不只來自單次人工 spot-check，而是經過三輪驗證：

1. 直接讀 `pseudodriver.go`，確認 pseudo-driver 的真實 slotting 模型
2. 用 Python 對單一 IP 做人工 spot-check，驗證：
   - raw parquet rebased window
   - `pseudo_history_push` / `pseudo_phase2_push`
   - `NWDAF UPF VOLUME`
   三者能逐筆對上
3. 再用 [verify_upf_pseudo_alignment.py](/home/chingje/testbed/5G_Infrastructure/.agent/verify_upf_pseudo_alignment.py) 對整個 `0513-13` 目錄做全量驗證

工具的核心檢查是：

- 對每個 UE 各自計算 `minTS`
- 做：
  - `wIdx = floor((ts - minTS) / 30)`
- 依 `pseudo_history_push` 最早的 `startTime` 推回每個 UE 的 live `GridAnchor`
- 再逐筆比：
  - raw parquet -> pseudo trace
  - raw parquet -> `NWDAF UPF VOLUME`

這也是本報告現在能把結論改成「6 個 IP 都 `180/180 exact`」的依據。

## Correct comparison method

這次一開始沿用了 `0512-19` 的 group-level slot 對齊方式，因此一度錯誤地判斷：

- pseudo-only 之後仍大量 mismatch
- `tail` 在 dataset 播完後還持續輸出大流量

後續重新 trace `pseudodriver.go` 後，確認 `0513-13` 正確的對齊方式應該是：

1. `scanTimestampRange()` 會**先套 subscription filter**，再計算 `minTS`
2. `streamAndAccumulate()` 會以該 subscription 對應 UE 的 `minTS` 做 rebasing：
   - `globalTS = row.Timestamp - minTS`
3. 再以：
   - `wIdx = floor(globalTS / 30)`
   切 30s window
4. `GridAnchor` 則由：
   - `referenceTime = anchorTime - alignedBreakingTime`
   決定
5. 因此 live notify 的 `startTime` 對應的是：
   - `startTime = GridAnchor + 30s * wIdx`

這代表 `0513-13` 不能再用：

- group-level `slot_raw = floor(ts/30)`
- 或 `pre_abs_slot = slot_raw + 1`

去和 `UPF VOLUME` 直接對齊。

### Per-subscription rebasing model

這次資料集本身就有 per-UE offset：

- `10.10.0.1` / `10.100.0.1`：從 `0s` 開始
- `10.10.0.2` / `10.100.0.2`：從 `15s` 開始
- `10.10.0.3` / `10.100.0.3`：從 `30s` 開始

因此 pseudo-driver 的實際 windowing 是：

- `x.x.0.1` 以 `0s` 為 `wIdx=0`
- `x.x.0.2` 以 `15s` 為 `wIdx=0`
- `x.x.0.3` 以 `30s` 為 `wIdx=0`

這個模型一旦用對，後面的比對結果會完全改觀。

## Trace result: pseudo-only notify is active

這次最先確認的是：`pseudo-only notify` 的程式路徑確實有生效。

從 [UPF-EES/upf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES/upf.log) 與 [UPF-EES2/upf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES2/upf.log) 可直接看到：

- `path="kernel_push_report"`
- `path="kernel_report_ignored_pseudo_only"`
- `path="pseudo_history_push"`
- `path="pseudo_phase2_push"`
- `path="merge_seed"`
- `path="notify_dispatch"`

而且 `kernel_push_report` 與 `kernel_report_ignored_pseudo_only` 的次數是一致的：

| file | `kernel_push_report` | `kernel_report_ignored_pseudo_only` |
|---|---:|---:|
| `UPF-EES/upf.log` | 3147 | 3147 |
| `UPF-EES2/upf.log` | 3147 | 3147 |

也就是：

- kernel report 有進來
- 但每一筆都被 pseudo-only 模式忽略
- final notify 沒有把 kernel report append 進 buffer

更重要的是，這次兩台 UPF 的 `kernel_push_report` 都是：

- `ulBytes="0"`
- `dlBytes="0"`

也就是：

- 不只是被忽略
- 這次 trace run 中，kernel contribution 本身也沒有提供任何非零 usage

這點值得特別寫清楚：

- 這次不是單純「kernel bytes 有進來，但被 pseudo-only 擋掉」
- 而是 **kernel path 自己記錄到的 bytes 本身就是 0**

因此就這次 `0513-13` run 而言，可以先做一個較強的中間結論：

- `kernel` 路徑在「流量數值」這一層，**看起來本來就不會影響 final notify**
- 換句話說，這次的 mismatch 不能合理解讀成「kernel bytes 雖然被忽略，但其實本來很大」

目前仍要保留的只有一個 caveat：

- kernel path 仍可能影響時間軸控制，例如 `SignalFirstURR` 或 `GridAnchor`
- 但如果只看 `ulBytes/dlBytes` 數值本身，這次沒有看到任何非零 contribution

## Tcpdump result: no meaningful GTP-U traffic observed

四份 tcpdump summary 幾乎都沒有看到有效 N3 GTP-U：

| location | captured | received by filter |
|---|---:|---:|
| `UPF-EES` | 0 | 0 |
| `UPF-EES2` | 0 | 0 |
| `gNB` | 0 | 0 |
| `gNB2` | 0 | 1 |

可解讀成：

- 這次 trace run 沒有觀察到明顯的 N3 user-plane traffic
- 至少從封包層看，不支持「有真實 GTP-U 流量混進來」這條線

## Result summary: pseudo-only **does** match `pre_data` when aligned correctly

在改用 pseudo-driver 真正的 per-subscription rebasing 後，`UPF VOLUME` 和 raw `pre_data` 是逐 window exact match 的。

本次 6 個 IP 的結果如下：

| IP | exact / rows |
|---|---:|
| `10.10.0.1` | `180 / 180` |
| `10.10.0.2` | `180 / 180` |
| `10.10.0.3` | `180 / 180` |
| `10.100.0.1` | `180 / 180` |
| `10.100.0.2` | `180 / 180` |
| `10.100.0.3` | `180 / 180` |

也就是：

- 這次 pseudo-only 後的 final notify，對所有 IP 都能和 `pre_data` 逐 window exact 對上
- 先前報告中的大規模 mismatch，是由錯誤對齊方法造成的誤判

實際工具輸出也固定為：

```text
10.10.0.1: trace=180/180 (history=30, phase2=150, contiguous=True) ; nwdaf=180/180
10.10.0.2: trace=180/180 (history=30, phase2=150, contiguous=True) ; nwdaf=180/180
10.10.0.3: trace=180/180 (history=30, phase2=150, contiguous=True) ; nwdaf=180/180
10.100.0.1: trace=180/180 (history=30, phase2=150, contiguous=True) ; nwdaf=180/180
10.100.0.2: trace=180/180 (history=30, phase2=150, contiguous=True) ; nwdaf=180/180
10.100.0.3: trace=180/180 (history=30, phase2=150, contiguous=True) ; nwdaf=180/180
```

## Direct evidence: `UPF EES trace -> NWDAF log -> raw pre_data` can be closed

這次最關鍵的修正，是把三段對照用正確的 pseudo-driver slot 語意重新接起來。

### Example: `10.10.0.1` first historical slot (`wIdx=0`)

在 [UPF-EES/upf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/UPF-EES/upf.log) 可看到：

- `path="pseudo_history_push"`
- `ueIp="10.10.0.1"`
- `startTime="2026-05-13 03:24:10 +0000 UTC"`
- `ulBytes="160146"`
- `dlBytes="105867"`

在 [nwdaf.log](/home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13/nwdaf.log) 同一 slot 可看到：

- `UPF VOLUME: ip=10.10.0.1, startTime=2026-05-13T03:24:10Z, total=266013, ul=160146, dl=105867`

也就是：

- `UPF EES trace` 和 `NWDAF collector` 是對得上的

若依照 pseudo-driver 的真實規則：

- `minTS(10.10.0.1) = 0`
- `wIdx = floor((ts - 0) / 30)`

那 raw `pre_data` 在 `wIdx=0` 的聚合值就是：

| source | `ul` | `dl` | `total` |
|---|---:|---:|---:|
| `UPF notify / NWDAF` | 160146 | 105867 | 266013 |
| `pre_data wIdx=0` | 160146 | 105867 | 266013 |

所以這裡真正能收斂的是：

- `UPF EES trace` 和 `NWDAF collector` 是對得上的
- raw `pre_data` 在**正確 rebasing**後也和它們對得上
- 先前的 mismatch，是把 `wIdx=0` 誤當成了某個 group-level phase2 slot

## Tail behavior: prior `tail` alarm was also a misalignment artifact

先前報告把某些大流量樣本歸到 `tail`，是因為仍在用 group-level absolute slot 去看：

- `10.10.0.3` 的資料從 `30s` 才開始
- `10.100.0.2` 的資料從 `15s` 才開始
- `10.100.0.3` 的資料從 `30s` 才開始

在這種資料集設計下，如果不先做 per-subscription rebasing，就會把：

- 仍屬於 `wIdx 177..179` 的正常資料

錯誤投影到一個看似「dataset 已播完」的 group-level `tail`。

重新按 pseudo-driver 語意比對後，這次每個 IP 都只有：

- `wIdx 0..179`
- 共 `180` 個 live rows

而且這 `180` 個 rows 全都能 exact match raw `pre_data`。

因此 `0513-13` 這次**沒有證據支持 pseudo-driver 在 dataset 結束後還持續輸出虛假的 tail 流量**。

## Interpretation

這次 `0513-13` trace run 回答了兩件事。

### 1. 這次的主要偏差不是 kernel live URR fusion

理由：

- pseudo-only notify 確實生效
- kernel report 全數被 ignore
- 這次看到的 kernel report 本身也全是 0 bytes
- tcpdump 沒看到明顯 GTP-U

因此：

- 至少在這次 run 中，不能再把主要行為歸因於 kernel fusion

### 2. 這次 pseudo-only path 本身在資料數值上是成立的

因為：

- `UPF EES trace -> NWDAF log` 是對得上的
- raw `pre_data` 在 correct rebasing 下也和它們對得上
- 所有 6 個 IP 都達到 `180/180` exact

所以這次更合理的結論是：

- pseudo-driver 的資料數值與 notify 輸出本身沒有問題
- 真正有問題的是我們先前沿用 `0512-19` 的 slot 對齊方式

## What this report does **not** prove

這份報告可以排除：

- 這次 run 的主要行為不是 kernel live URR fusion 造成
- 這次 run 的 pseudo-only final notify 不是錯的

但它還不能直接證明：

- `0512-19` 那次 mismatch 的全部根因是什麼
- 其他 branch / 其他 session 是否也都遵循同樣的 per-subscription rebasing 規則

## Next step

下一步最值得做的是：

1. 更新 `0512-19` 方法文件，把：
   - per-subscription `minTS` rebasing
   - `referenceTime = anchorTime - alignedBreakingTime`
   - `wIdx -> startTime = GridAnchor + 30s * wIdx`
   寫成新的校正流程
2. 回頭重新審視 `0512-19` 的 mismatch，分清楚：
   - 哪些是當時真的有問題
   - 哪些只是沿用了錯的 slot 對齊假設

### Re-run command

若之後需要重跑同型驗證，可直接執行：

```bash
/home/chingje/testbed/5G_Infrastructure/.agent/verify_upf_pseudo_alignment.py \
  /home/chingje/testbed/5G_Infrastructure/.agent/tmp/0513-13
```
4. 是否有 replay reuse / stale buffer / tail replay 的行為

就診斷順序來說，`0513-13` 現在更支持下面這條線：

- 先不要再把主要精力放在 kernel fusion
- 先修正 comparison / calibration 方法
- 再回頭重新審視 `0512-19` 那次真正還剩下哪些異常

## Recommended next steps

### 1. 先把 per-subscription rebasing 納入通用校正流程

後續每次做：

- `UPF notify`
- `UPF EES trace`
- `raw pre_data`

三者對照時，都應先確認：

- 這次 subscription filter 是不是單一 UE
- `scanTimestampRange()` 用的是哪個 `minTS`
- `globalTimeOffset` 是否為 0

只有在這些都明確後，slot comparison 才有意義。

### 2. 回頭重新審視 `0512-19`

現在最值得做的不是繼續深挖 `0513-13` 的 pseudo path，而是把這套對齊模型帶回 `0512-19`：

- 哪些 mismatch 在正確 rebasing 後其實會消失？
- 哪些 IP / slot 在重新校正後仍然異常？

這一步能把：

- 真問題
- 與當時方法造成的假 mismatch

分開。

### 3. 保留 trace，但把 pseudo path 進一步結構化

雖然 `0513-13` 已經證明 final notify 和 raw `pre_data` 可以對上，但 trace 層仍可再補一點中介語意，讓下一輪比較更便宜：

- `minTS`
- `rebased wIdx`
- `referenceTime`
- `startTime = GridAnchor + 30s * wIdx`

這樣之後不用再從 `pseudodriver.go` 反推一次。

### 4. 暫時把 kernel 路徑降為觀察項

在這次 run 中：

- `kernel_push_report` 全是 `0/0`
- `pseudo-only notify` 確實生效
- tcpdump 幾乎沒有 GTP-U

因此 kernel 目前只需要保留兩個觀察點：

- 它是否仍會影響時間軸控制
- trace 是否持續保持可觀測

但這次 session 的主問題已不在 kernel bytes 本身。
