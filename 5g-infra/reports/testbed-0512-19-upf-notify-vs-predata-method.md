# testbed 0512-19: UPF notify vs `pre_data` comparison method note

## Purpose

這份文件專門記錄 `docs/5g-infra/reports/testbed-0512-19-upf-notify-vs-predata-all-ip.md` 背後實際使用的比對腳本邏輯，目的是讓之後跨 session 時，可以不依賴當下對話記憶，直接重建相同方法。

這裡記錄的是：

- 當時腳本**實際做了什麼**
- 有哪些對齊假設
- 哪些欄位是經由實際對照後確立，不是從文件直接抄來
- 哪些地方是容易踩坑的細節

這份文件不是最終分析報告，而是分析方法的重建說明。

> 注意：
> 這份方法最初是為 `0512-19` 這一輪比對整理的。  
> 在後續 `0513-13` 的 pseudo-only trace run 中，已經證明**不能把這份方法中的所有對齊常數直接視為通用真理**。  
> 特別是：
> - phase2 anchor
> - `pre_abs_slot = slot_raw + 1`
> - 「live `UPF VOLUME` 可以直接拿來代表 pseudo-driver 原始輸出」
>
> 這些都必須在每個新 session 重新驗證。

## Scope

這份方法文件對應的是：

- `docs/5g-infra/reports/testbed-0512-19-upf-notify-vs-predata-all-ip.md`

它只描述：

- `nwdaf.log` 中 `UPF VOLUME` per-IP rows
- 與 `go-upf pre_data/*.parquet` 聚成 30s slot 後的逐 slot 對照

它**不處理**：

- replay `exp46` 的 prediction / trigger analysis
- `upf-ees.log` 的 code-path trace
- `gnb/ue` control-plane log 分析

那些屬於後續調查的下一層。

## Why this note needs to be more explicit

在 `0513-13` 這一輪，我們遇到了一個很重要的情況：

- `pseudo-only notify` 已確認生效
- `kernel_push_report` 的 `ulBytes/dlBytes` 全是 `0`
- `UPF EES trace` 和 `NWDAF log` 是對得上的
- 但它們一起和 raw `pre_data parquet` 對不上

後續再沿著 `pseudodriver.go` 實際 trace，並用
[verify_upf_pseudo_alignment.py](/home/chingje/testbed/5G_Infrastructure/.agent/verify_upf_pseudo_alignment.py)
做全量驗證後，`0513-13` 的正確結論其實是：

- `UPF EES trace`
- `NWDAF UPF VOLUME`
- raw `pre_data`

三者在**正確的 pseudo-driver 對齊模型**下，是可以完全對上的。

真正的問題不是 pseudo-driver 數值錯，而是：

- 我們一開始把 `0512-19` 的 group-level slot 對齊規則，錯誤套到 `0513-13`
- 因而把 per-subscription rebased window 誤判成 mismatch / tail

這代表：

- 之前在 `0512-19` 觀察到的對齊關係，**不一定能直接搬到下一輪**
- 每次做 session 級比對時，都必須先回答：
  - `UPF VOLUME` 代表的是哪一層的資料？
  - 這一輪的 pseudo-driver slot index 是怎麼定義的？
  - pseudo-driver 用的是 group-level time base，還是 subscription / UE-specific time base？

因此這份方法文件除了記錄 `0512-19` 當時怎麼做，也要明確記錄：

- 哪些步驟是可重用的
- 哪些步驟只是當次 session 驗出來的結果
- 哪些步驟在新 session 必須重新校正

## Data inputs

當時使用的輸入資料如下：

### 1. Live side

- `5G_Infrastructure/.agent/tmp/0512-19/nwdaf.log`

只從裡面抽這種格式的 row：

```text
UPF VOLUME: ip=..., startTime=..., total=..., ul=..., dl=...
```

### 2. Replay source side

- `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/file.json`
- `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/file.json`
- `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet`
- `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet`

### 3. Prior reference

- `docs/5g-infra/reports/testbed-0505-22-vs-exp27-sharedscaler.md`
- [testbed-0513-13-pseudo-only-trace-alignment-validation.md](/home/chingje/testbed/docs/5g-infra/reports/testbed-0513-13-pseudo-only-trace-alignment-validation.md)

這份舊報告主要提供：

- `phase1 = 900s`
- pseudo-driver 在 testbed 上的時間對齊概念

但本次比對的很多細節，仍是重新用現場數據驗出來的，不是直接沿用舊報告。

### 4. Verification helper

- [verify_upf_pseudo_alignment.py](/home/chingje/testbed/5G_Infrastructure/.agent/verify_upf_pseudo_alignment.py)

這支工具把 `0513-13` 最後確認過的對齊模型固定成可重跑流程：

- per-subscription `minTS` rebasing
- 由 pseudo trace 的最早 `startTime` 反推 live `GridAnchor`
- 逐筆驗證：
  - raw parquet -> pseudo trace
  - raw parquet -> `NWDAF UPF VOLUME`

若後續 session 結構仍和 `0513-13` 類似，可以先跑這支工具做一次全量 sanity check，再決定是否需要 deeper manual investigation。

## High-level comparison flow

當時的比對腳本邏輯可以分成 7 步。

但在真正重做分析時，建議改成 **兩階段流程**：

1. 先做 `calibration`
2. 再做 `comparison`

如果跳過 `calibration`，很容易把：

- slot 漂移
- pseudo-driver 的 per-subscription rebasing
- 看似異常的 dataset tail

誤判成純粹的 `pre_data mismatch`。

下面先寫原始的 7 步，再補「後來證明必須加上的 calibration 步驟」。

## 0513-13 correction: the missing calibration step

`0513-13` 之所以一開始看起來整組都壞掉，真正缺的是下面這個 calibration：

### Pseudo-driver actual slot model

`pseudodriver.go` 的實際流程不是：

- 直接用 group-level `slot_raw = floor(ts/30)`

而是：

1. `scanTimestampRange()` 會先套 subscription filter
2. 對每個 subscription 單獨求：
   - `minTS`
   - `maxTS`
3. `streamAndAccumulate()` 再做：
   - `globalTS = row.Timestamp - minTS + globalTimeOffset`
4. 單檔單 subscription 時：
   - `globalTimeOffset = 0`
   - 所以等價於：
     - `globalTS = row.Timestamp - minTS`
5. 再切：
   - `wIdx = floor(globalTS / period)`

因此在 `0513-13` 這種單一 parquet、單一 UE subscription 的 pseudo-only trace run 中，真正的對齊規則是：

- `10.10.0.1` / `10.100.0.1`：以 `0s` 為 `wIdx=0`
- `10.10.0.2` / `10.100.0.2`：以 `15s` 為 `wIdx=0`
- `10.10.0.3` / `10.100.0.3`：以 `30s` 為 `wIdx=0`

這也是為什麼：

- 一開始用 group-level slot 去比，會把 `.2` / `.3` 誤判成大規模 mismatch
- 但一旦改成 per-subscription rebasing，6 個 IP 都能 `180/180` exact match

### Implication

因此之後做新 session 比對時，應先問：

- 這次 pseudo-driver 是不是一個 subscription 只對一個 UE？
- `minTS` 是 group-level 還是 subscription-level？
- `globalTimeOffset` 是否為 0？還是有多檔串接？

如果這三點沒先校正，就不應直接拿 raw parquet 的 group-level `slot_raw` 去對 `UPF VOLUME`。

### Step 1. Parse `UPF VOLUME` rows from `nwdaf.log`

先從 `nwdaf.log` 中抽所有：

```text
UPF VOLUME: ip=..., startTime=..., total=..., ul=..., dl=...
```

對每筆 row 取出：

- `ip`
- `startTime`
- `total`
- `ul`
- `dl`

並把 `startTime` 轉成 timestamp，方便做 slot 對齊。

#### `0513-13` lessons learned

在 `0513-13` 之後，要補一個重要提醒：

- `nwdaf.log` 的 `UPF VOLUME` row 只能保證是 **NWDAF collector 收到的 notify payload**
- 它不保證一定等同於 raw `pre_data` 聚合值
- 在 pseudo-driver path 有問題時，`UPF VOLUME` 只是錯誤輸出的最終呈現，不是原始真值

所以：

- `UPF VOLUME` 是比對目標
- 不是對齊基準本身

### Step 2. Read Parquet and aggregate to 30s slots

`training_packets_run001.parquet` 是 packet-level row，不是直接可比的 slot-level usage。

腳本做法是：

1. 讀出每筆 row 的：
   - `ts`
   - `ue_ip`
   - `direction`
   - `len`
2. 算：
   - `slot_raw = floor(ts / 30)`
3. 依下面 key 聚合：
   - `(ue_ip, slot_raw, direction)`
4. 再把相同 `(ue_ip, slot_raw)` 的 `ul/dl` 合起來

也就是先得到每個：

- `ue_ip`
- `slot_raw`
- `pre_ul`
- `pre_dl`
- `pre_total = pre_ul + pre_dl`

但從 `0513-13` 起，要再往前加一步：

- 決定這次 pseudo-driver 會不會先對每個 subscription 做 `minTS` rebasing

若會，就不能直接把這裡的 `slot_raw` 當成最終對齊依據，而要改算：

- `slot_rebased = floor((ts - minTS_for_this_subscription) / 30)`

#### `0513-13` lessons learned

這一步在跨 session 時仍然是必要的，但有兩個細節不能偷懶：

1. 必須先確認這次 pseudo-driver 讀的是不是同一份 parquet
2. 必須先確認 pseudo-driver 是否真的直接以 raw 30s packet aggregate 產生 `UsageMeasures`

在 `0513-13` 中，證據顯示：

- `pseudo_history_push`
- `merge_seed`
- `notify_dispatch`
- `NWDAF UPF VOLUME`

彼此可以完全對上，而 raw parquet 若先經過**正確的 per-subscription rebasing**，也能和它們完全對上。

所以 raw parquet 聚合雖然仍是必要的「理論基準」，但不能先假設：

- group-level `slot_raw` 就是 pseudo-driver 真正使用的 slot

### Step 3. Resolve direction mapping from data, not assumption

雖然原始欄位叫 `direction`，但當時**不是直接假設**：

- `0 = ul`
- `1 = dl`

而是先拿已知正常的 IP 對照 `UPF VOLUME` 實際數字，才確立這次資料集的方向對應是：

- `direction=0 -> ul`
- `direction=1 -> dl`

這一步很重要，因為如果方向猜反，會造成：

- `ul/dl` 欄位全面錯位
- `total` 看起來也可能局部「接近但不 exact」

所以在之後重建腳本時，這個 mapping 不應該只靠想當然耳，最好重新做 spot-check。

#### `0513-13` lessons learned

這一步在新 session 仍然要保留，而且最好要有**明確的 spot-check 對象**。

建議順序：

1. 先拿 `upf.log` 中某個 `pseudo_history_push`
2. 再找同一 IP / 同一 `startTime` 的 `UPF VOLUME`
3. 再檢查 raw parquet 的 `direction=0/1` 聚合

若：

- `UPF VOLUME` 和 `pseudo_history_push` 一致
- 但和 raw parquet 不一致

那代表問題不在 direction mapping，而在 pseudo-driver 生成路徑本身。

## Time alignment details

這次比對最容易出錯的地方，不是 Parquet 聚合，而是 live slot 和 raw `pre_data` slot 的時間對齊。

### Step 4. Read `breaking time` from `file.json`

`group1/file.json` 與 `group2/file.json` 都寫：

- `breaking time = 900`

因此 pseudo-driver 的語意是：

- `phase1` 先消耗 900 秒
- 之後才進入真正有對照意義的 `phase2`

換算成 30s slot：

- `900 / 30 = 30`

也就是：

- `phase2` 會從 **absolute slot 30** 開始

#### `0513-13` lessons learned

`breaking time = 900` 仍然只告訴你：

- phase1 理論上持續 900s
- phase2 理論上從第 30 個 30s window 開始

但它**不能單獨決定 raw parquet 和 live notify 的 slot mapping**。

你還必須搭配：

- 該 subscription 的 `minTS`
- `anchorTime`
- `referenceTime = anchorTime - alignedBreakingTime`

一起看，才能把 raw `wIdx` 映到正確的 live `startTime`。

但它**沒有保證**：

- live anchor 要怎麼選
- pseudo-driver 在 trace 中對應到哪個 `startTime`
- phase1 history 的最後一筆一定緊鄰 phase2 第一筆

也就是：

- `breaking time` 提供的是 high-level schedule
- 不是最終可直接代入比對腳本的 slot mapping

### Step 5. Choose the phase2 live anchor from actual `nwdaf.log`

這一步不是直接用某個固定時鐘，而是看 `nwdaf.log` 中最早的 live aggregated slot。

當時對齊結果是：

- `group1` phase2 anchor = `2026-05-12T09:56:39Z`
- `group2` phase2 anchor = `2026-05-12T09:56:40Z`

所以腳本中 live absolute slot 是這樣算：

- `group1_abs_slot = round((startTime - 09:56:39Z) / 30s) + 30`
- `group2_abs_slot = round((startTime - 09:56:40Z) / 30s) + 30`

這裡用 `round` 而不是 `floor`，是因為 live `startTime` 已經是經過系統各層處理後的對齊時間，當時想避免微小秒差導致 slot 漂移。

#### `0513-13` lessons learned

這一步是最容易被誤用的地方。

在 `0512-19` 中，直接用 `nwdaf.log` 最早的 aggregated slot 當 phase2 anchor 是可用的。  
但在 `0513-13` 中，後來證明：

- 這個 anchor 只能代表 **collector 收到 notify 的最早時間**
- 不能保證它和 raw parquet 的 phase2 起點存在固定的 `+1` 或固定 slot 對映

更安全的做法應該是：

1. 先從 `upf.log` 抽：
   - 第一筆 `pseudo_history_push`
   - 第一筆 `pseudo_phase2_push`
2. 再看這兩筆和 `UPF VOLUME` 是否逐筆一致
3. 之後才決定要不要把 `nwdaf.log` 的最早 slot 當成 live anchor

也就是說：

- `nwdaf.log anchor` 在 `0512-19` 是可行經驗值
- 在後續 session 中，應該降級成**待驗證假設**

### Step 6. Resolve the off-by-one between live slots and raw Parquet slots

這次比對裡最關鍵的一步，是發現：

- live `abs_slot = 30`
- 對應到的不是 `pre_data slot_raw = 30`
- 而是 `pre_data slot_raw = 29`

也就是說，腳本最後採用：

- `pre_abs_slot = slot_raw + 1`

如果忽略這個 `+1`，會造成：

- 連完全正常的 IP 都會看起來全部不一致

這個 `+1` 不是理論推導，而是透過已知正常 IP 反推得出的實務對齊值。

因此這一步是整份方法裡最重要的「session 重建要保留的經驗值」之一。

#### `0513-13` lessons learned

這裡要特別加粗提醒：

- **`pre_abs_slot = slot_raw + 1` 不是通用規則**

它只是在 `0512-19` 當時，對那輪 session 成立的經驗值。

在 `0513-13` 中，若沿用這個 `+1`，會導致：

- `10.10.0.1` 在第一筆 `pseudo_history_push` 就和 raw parquet 嚴重不一致

具體例子：

- `pseudo_history_push`
  - `startTime=2026-05-13 03:24:10Z`
  - `ul=160146`
  - `dl=105867`
- `UPF VOLUME`
  - 同 slot 完全相同
- 但 raw parquet 若用 `slot_raw=29`（也就是沿用 `+1` 規則）
  - `ul=202450`
  - `dl=211973`

反而 raw parquet 的：

- `slot_raw=0`
  - `ul=160146`
  - `dl=105867`

才和 `pseudo_history_push` 對上。

這表示：

- 在 `0513-13` 中，`+1` 不成立
- 或更精確地說：
  - pseudo-driver 自己的 slot 編排邏輯，已經和 `0512-19` 不同

因此之後的通用規則應改成：

- **永遠不要先假設固定 off-by-one**
- 必須先用 trace log 對單一 IP 做小範圍校正，再決定這輪 session 的 slot mapping

## Exact-match logic

### Step 7. Compare at `ip + abs_slot` granularity

當時腳本最後對每個：

- `ip`
- `abs_slot`

做逐欄位比較：

- `live_ul == pre_ul`
- `live_dl == pre_dl`
- `live_total == pre_total`

不是只比較 `total`。

只是最終 summary table 有兩種層級：

1. **exact-total match**
   - 只要求 `live_total == pre_total`
   - 用來快速看 phase-by-phase 的整體對齊比例

2. **exact field-level match**
   - `ul/dl/total` 全部一致
   - 用來判定「真正完全正常」

也就是說：

- 某些 IP 可以在 `total` 上看起來有一部分 exact
- 但 `ul/dl` 仍可能存在相鄰 slot 小偏移

這就是為什麼 `10.100.0.2` 被分類成：

- 「接近，但不是全程 exact」

而不是完全正常。

#### `0513-13` lessons learned

在 pseudo-only trace run 中，應該先做三層對照，而不是直接做 `live vs parquet`：

1. `pseudo_history_push / pseudo_phase2_push`
2. `UPF VOLUME`
3. raw parquet aggregate

先問：

- `1` 和 `2` 是否一致？

若一致，再問：

- `1/2` 和 `3` 是否一致？

這樣才能區分：

- collector / notify 問題
- 還是 pseudo-driver materialization 問題

## Phase bucket definitions

當時腳本為了方便統計，把 `abs_slot` 分成五段：

- `phase1`: `abs_slot < 30`
- `cat1_phase2`: `30 <= abs_slot < 90`
- `cat2_phase2`: `90 <= abs_slot < 150`
- `cat3_phase2`: `150 <= abs_slot < 182`
- `tail`: `abs_slot >= 182`

這裡的關鍵語意是：

- `cat3_phase2` 仍然屬於 pseudo-driver 還在播放的有效資料段
- `tail` 才是 pseudo-driver 播完後的 0 流量尾段

這個切法後來有特別修正過，因為一開始把 `cat3` 和尾段混在一起，會讓 `tail` 的解讀失真。

#### `0513-13` lessons learned

`tail` 在後續 session 裡不應該只被當作「最後一段統計方便切分」，而應該被當成：

- **獨立的 debug 診斷區段**

因為在 `0513-13` 中：

- raw parquet 預期已播完
- 但 `tail` 仍出現 `80M+` 等級的非零流量

這種情況不是一般 slot 對齊誤差可以解釋的，通常意味著：

- replay loop
- stale buffer reuse
- dataset end condition 失效
- phase1/history data 被延續到尾段

所以後續重建方法時，`tail` 應單獨檢查：

- `pre_data` 是否理論上已為 0
- pseudo trace 是否仍持續 emit 非零 measure
- `UPF VOLUME` 是否持續收到大流量

這三層應分開確認。

## What the script really proved

這份腳本比對真正證明的是：

1. 有些 IP 可以做到逐 slot、逐欄位 exact match  
   例如：
   - `10.10.0.1`
   - `10.100.0.1`

2. 有些 IP 明顯不是 exact replay  
   例如：
   - `10.10.0.3`
   - `10.100.0.3`

## Recommended calibration procedure for future sessions

綜合 `0512-19` 和 `0513-13` 的經驗，之後若要在新 session 重建這類比對，建議固定先做下面這組 calibration，再開始正式統計。

### Calibration Step A. 先看 trace，不要直接看 `UPF VOLUME`

若該輪有 trace log，先抽：

- `pseudo_history_push`
- `pseudo_phase2_push`
- `merge_seed`
- `notify_dispatch`

確認：

- pseudo path 的輸出本身長什麼樣

### Calibration Step B. 用單一 IP 對第一個小窗口

先選一條 IP，例如：

- `10.10.0.1`

只對：

- 第一筆 history
- 第一筆 phase2
- `slot_raw 0..5`

做小範圍對帳，確認：

- `UPF VOLUME` 是否和 pseudo trace 一致
- pseudo trace 對應到 raw parquet 的哪個 `slot_raw`

### Calibration Step C. 再決定這輪的 anchor 與 off-by-one

只有在完成 A/B 後，才決定：

- 這輪 session 的 live anchor
- 是否存在 `+1`、`0`、或其他 offset

不要把上一輪的 offset 直接套用。

### Calibration Step D. 最後才跑全 IP / 全 phase summary

一旦小範圍對帳確認無誤，再跑：

- 全 IP
- 全 phase bucket
- `tail`

這樣可以避免先用錯 mapping，導致整份 summary 全部失真。

## Main pitfalls checklist

之後只要重建這類腳本，建議先逐項檢查：

- 是否重新驗證 `direction=0/1` 對應？
- 是否重新驗證 live anchor？
- 是否重新驗證 off-by-one？
- 是否先確認 `UPF VOLUME` 和 pseudo trace 本身一致？
- 是否單獨檢查 `tail` 是否理論上應為 0？
- 是否把 `0512-19` 的結論錯當成通用規則？

若以上任一步沒做，後面的 mismatch 解讀就很容易出錯。

3. 還有一些 IP 處於中間態  
   例如：
   - `10.100.0.2`

所以這份方法不是只在做：

- `這次 live aggregate 看起來怪不怪`

而是在做：

- `同一個 IP、同一個 slot、同一個欄位，到底有沒有 exact 等於 pre_data`

## What the script did not prove

這份比對腳本本身**沒有證明**：

- 那些多出來的 bytes 一定來自 kernel
- pseudo-driver 一定沒有任何 contribution error
- NWDAF collector 一定完全沒有改寫資料

它只證明：

- `final UPF VOLUME` 和 `pre_data` 的關係是什麼
- 哪些 IP / slot exact
- 哪些 IP / slot 不 exact

至於為什麼不 exact，則是靠後續：

- `upf-ees.log`
- `go-upf` code path
- `gnb/ue` 輔助 log

才往下推論。

## Practical reconstruction notes

之後若要重建這份分析，最容易漏掉的細節有：

1. `direction` mapping 不能只憑印象，要再用正常 IP spot-check 一次。
2. live phase2 anchor 是從 `nwdaf.log` 反推出來，不是硬編碼的理論值。
3. `pre_abs_slot = slot_raw + 1` 這個 off-by-one 非常重要。
4. `tail` 要和 `cat3_phase2` 分開，不然會混到 pseudo-driver 播完後的 0 流量段。
5. `total exact` 和 `ul/dl exact` 要分開看，不能混為一談。

## Recommended follow-up

如果之後要把這份方法從「session 內臨時分析」變成可重複使用工具，建議下一步是：

1. 把當時的 ad hoc 比對邏輯整理成一支正式腳本
2. 腳本輸出至少包含：
   - per-IP exact ratio
   - per-phase exact ratio
   - `ul/dl/total` 的逐欄位 diff sample
3. 把：
   - anchor time
   - direction mapping
   - off-by-one alignment
   都做成明確參數或中間輸出

這樣下次就不需要只靠報告文字重建。
