# testbed 0513-13: pseudo-only trace run vs `pre_data`

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

## Comparison method

這次仍沿用 `0512-19` 方法筆記中的核心做法：

- 從 `nwdaf.log` 抽 `UPF VOLUME` per-IP row
- 將 `training_packets_run001.parquet` 聚成 30s slot
- 使用：
  - `direction=0 -> ul`
  - `direction=1 -> dl`
- 以每個 group 在 live log 中最早的 aggregated slot 當 phase2 anchor

本次 live anchor 為：

- `group1`: `2026-05-13 03:24:10+00:00`
- `group2`: `2026-05-13 03:24:08+00:00`

phase bucket 定義維持：

- `cat1_phase2`: `30 <= abs_slot < 90`
- `cat2_phase2`: `90 <= abs_slot < 150`
- `cat3_phase2`: `150 <= abs_slot < 182`
- `tail`: `abs_slot >= 182`

`tail` 的語意是：

- dataset 預期已播完後的尾段
- 在這個區段中，若 pseudo-driver 行為正確，應該趨近於 0 流量

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

## Result summary: pseudo-only did **not** restore match to `pre_data`

即使 final notify 只採用 pseudo-driver，live `UPF VOLUME` 仍然沒有回到和 raw `pre_data` 一致。

整體 exact-match 統計如下：

| bucket | rows | exact `ul/dl/total` | mismatch |
|---|---:|---:|---:|
| `cat1_phase2` | 360 | 0 | 360 |
| `cat2_phase2` | 360 | 131 | 229 |
| `cat3_phase2` | 192 | 0 | 192 |
| `tail` | 168 | 0 | 168 |

這代表：

- `cat1_phase2` 完全沒有任何一筆逐欄位 exact
- `cat2_phase2` 有部分 exact，但仍有大多數 mismatch
- `cat3_phase2` 與 `tail` 都完全不吻合

若只看 per-IP / per-bucket exact counts：

| IP | `cat1_phase2` | `cat2_phase2` | `cat3_phase2` | `tail` |
|---|---:|---:|---:|---:|
| `10.10.0.1` | 0/60 | 51/60 | 0/32 | 0/28 |
| `10.10.0.2` | 0/60 | 0/60 | 0/32 | 0/28 |
| `10.10.0.3` | 0/60 | 0/60 | 0/32 | 0/28 |
| `10.100.0.1` | 0/60 | 31/60 | 0/32 | 0/28 |
| `10.100.0.2` | 0/60 | 49/60 | 0/32 | 0/28 |
| `10.100.0.3` | 0/60 | 0/60 | 0/32 | 0/28 |

## Direct evidence: mismatch already exists before NWDAF

這次最關鍵的證據，不是統計表，而是 `UPF EES trace -> NWDAF log -> raw pre_data` 的三段對照。

### Example: `10.10.0.1` first historical slot

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

但 raw `pre_data` 在對應 slot 的聚合值其實是：

| source | `ul` | `dl` | `total` |
|---|---:|---:|---:|
| `UPF notify / NWDAF` | 160146 | 105867 | 266013 |
| `pre_data slot_raw=29` | 202450 | 211973 | 414423 |

所以這裡可以直接收斂：

- mismatch 不是 NWDAF 端加工後才出現
- mismatch 已經在 pseudo-driver / EES 產生 notify 的那一層存在

## Tail behavior: dataset should be exhausted, but live traffic stays large

從 raw `pre_data` 的 slot range 來看：

- `10.10.0.1`: `slot_raw 0..179`
- `10.10.0.2`: `slot_raw 0..180`
- `10.10.0.3`: `slot_raw 1..180`
- `10.100.0.1`: `slot_raw 0..179`
- `10.100.0.2`: `slot_raw 0..180`
- `10.100.0.3`: `slot_raw 1..180`

也就是說：

- 大部分 IP 在 `abs_slot 180/181` 左右就應該已經播完
- `tail (abs_slot >= 182)` 應接近全 0

但實際上 `tail` 完全不是這樣。

代表例子：

| IP | abs slot | startTime | live total | `pre_data` total |
|---|---:|---|---:|---:|
| `10.10.0.3` | 200 | `2026-05-13T04:49:10Z` | 86,183,090 | 0 |
| `10.100.0.1` | 193 | `2026-05-13T04:45:38Z` | 85,938,656 | 0 |
| `10.100.0.1` | 182 | `2026-05-13T04:40:08Z` | 85,198,075 | 0 |
| `10.100.0.3` | 192 | `2026-05-13T04:45:08Z` | 85,102,634 | 0 |
| `10.100.0.2` | 191 | `2026-05-13T04:44:38Z` | 85,059,291 | 0 |

這表示：

- dataset 播完之後，pseudo path 並沒有自然掉到 0
- `tail` 仍在持續輸出大量非零 traffic

這個現象本身就足以說明：

- 問題不只是 phase2 slot 對齊誤差
- 還包含 dataset end / replay tail 行為異常

## Interpretation

這次 `0513-13` trace run 回答了兩件事。

### 1. 這次的主要偏差不是 kernel live URR fusion

理由：

- pseudo-only notify 確實生效
- kernel report 全數被 ignore
- 這次看到的 kernel report 本身也全是 0 bytes
- tcpdump 沒看到明顯 GTP-U

因此：

- 至少在這次 run 中，不能再把主要 mismatch 歸因於 kernel fusion

### 2. 問題已經收斂到 pseudo path 本身

因為：

- `UPF EES trace -> NWDAF log` 是對得上的
- 但它們一起和 raw `pre_data` 對不上
- `tail` 在 dataset 播完後仍持續輸出大流量

所以這次更合理的結論是：

- pseudo-driver 的資料生成 / windowing / slot alignment / dataset end 行為本身有問題

## What this report does **not** prove

這份報告可以排除：

- 這次 run 的主要偏差不是 kernel live URR fusion

但它還不能直接證明：

- pseudo-driver 內部究竟是哪一段程式邏輯造成偏差

也就是說，現在還缺的是：

- 對 pseudo-driver 讀 parquet、切 slot、產生 `UsageMeasures` 的程式路徑再往下 trace

## Next step

下一步最值得查的是 `go-upf` pseudo path 本身：

1. `pseudo_history_push` 的數值是怎麼從 parquet 算出來的
2. `pseudo_phase2_push` 的 window 與 raw slot 是否存在額外位移
3. dataset 播完後，為什麼 `tail` 仍持續輸出非零 traffic
4. 是否有 replay reuse / stale buffer / tail replay 的行為

就診斷順序來說，這次 `0513-13` 已經足夠支持：

- 先不要再把主要精力放在 kernel fusion
- 應優先往 pseudo-driver 路徑本身追查

## Recommended debug order

為了避免下一輪又同時追太多層，建議 debug 順序固定如下。

### 1. 先確認 pseudo-driver 內部 slot 是怎麼 materialize 出來的

第一優先不是看 NWDAF，也不是再看 kernel，而是直接查：

- raw parquet row
- pseudo-driver 內部 slot
- `pseudo_history_push` / `pseudo_phase2_push`

目標是回答：

- `UsageMeasures` 是不是在進入 aggregator 前就已經和 raw `pre_data` 不同？

這一層若已經偏掉，後面 `merge` / `notify` 都只是把錯誤數值原樣送出。

### 2. 先鎖單一 IP 做端到端對帳

建議先從：

- `10.10.0.1`

開始，而不是一開始就看全部 IP。

原因：

- 這條線在先前案例中曾經有正常對齊過
- 目前 `0513-13` 中它也有完整的 `pseudo_history_push -> merge_seed -> notify -> NWDAF` 路徑

建議先只對帳：

- `slot_raw 29..35`
- 對應的 live `abs_slot 30..36`

若連這條線都對不起來，就代表問題已經確定在 pseudo path 本身。

### 3. 在 pseudo-driver 讀 parquet 後、進 buffer 前補更細的 trace

目前已有：

- `pseudo_history_push`
- `pseudo_phase2_push`

但還缺中間層：

- raw `slot_raw`
- 該 slot 的 `direction=0` sum
- 該 slot 的 `direction=1` sum
- `packet_count`
- 最後 materialize 成的 `UsageMeasures`

也就是說，下一輪最值得新增的 trace 不是再補 kernel，而是補：

- `pseudo_slot_materialized`
- `pseudo_slot_emitted`

讓 raw parquet aggregate 和 pseudo-driver 內部中介表示能一一對上。

### 4. 特別檢查 history / phase2 handoff

這次 `cat1_phase2` 完全沒有任何 exact match，這讓交界處特別可疑。

要特別查：

- 最後一筆 `pseudo_history_push` 是哪個 slot
- 第一筆 `pseudo_phase2_push` 是哪個 slot
- 兩者之間是否有：
  - off-by-one
  - 重複
  - 漏 slot
  - 時間窗重疊

這一步的目的，是確認 phase1 warm-start 和 phase2 live replay 是否在 handoff 處就已經失真。

### 5. 把 `tail` 當成獨立 debug 線處理

`tail` 目前比一般 slot mismatch 更值得優先查，因為它的語意更直接：

- raw `pre_data` 已經沒有資料
- pseudo path 卻仍持續輸出 `80M+` 等級的非零流量

這通常代表：

- replay loop
- stale buffer reuse
- dataset end condition 失效
- slot index 在尾端處理錯誤

因此建議把 `tail` 當成獨立問題來查，而不是只把它視為 phase2 對齊誤差的延伸。

### 6. 暫時不要把主要精力放在 kernel path

依這次 run 的結果：

- `kernel_push_report` 全是 `0/0`
- `pseudo-only notify` 確實生效
- tcpdump 幾乎沒有 GTP-U

因此下一輪 debug 不應再以 kernel fusion 為主。

kernel 目前只需要保留兩個觀察點：

- 它是否還會影響時間軸控制
- trace 是否持續保持可觀測

但主問題已不在 kernel bytes 本身。
