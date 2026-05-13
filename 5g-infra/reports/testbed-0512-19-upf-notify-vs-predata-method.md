# testbed 0512-19: UPF notify vs `pre_data` comparison method note

## Purpose

這份文件專門記錄 `docs/5g-infra/reports/testbed-0512-19-upf-notify-vs-predata-all-ip.md` 背後實際使用的比對腳本邏輯，目的是讓之後跨 session 時，可以不依賴當下對話記憶，直接重建相同方法。

這裡記錄的是：

- 當時腳本**實際做了什麼**
- 有哪些對齊假設
- 哪些欄位是經由實際對照後確立，不是從文件直接抄來
- 哪些地方是容易踩坑的細節

這份文件不是最終分析報告，而是分析方法的重建說明。

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

這份舊報告主要提供：

- `phase1 = 900s`
- pseudo-driver 在 testbed 上的時間對齊概念

但本次比對的很多細節，仍是重新用現場數據驗出來的，不是直接沿用舊報告。

## High-level comparison flow

當時的比對腳本邏輯可以分成 7 步。

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

也就是最後得到每個：

- `ue_ip`
- `slot_raw`
- `pre_ul`
- `pre_dl`
- `pre_total = pre_ul + pre_dl`

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

### Step 5. Choose the phase2 live anchor from actual `nwdaf.log`

這一步不是直接用某個固定時鐘，而是看 `nwdaf.log` 中最早的 live aggregated slot。

當時對齊結果是：

- `group1` phase2 anchor = `2026-05-12T09:56:39Z`
- `group2` phase2 anchor = `2026-05-12T09:56:40Z`

所以腳本中 live absolute slot 是這樣算：

- `group1_abs_slot = round((startTime - 09:56:39Z) / 30s) + 30`
- `group2_abs_slot = round((startTime - 09:56:40Z) / 30s) + 30`

這裡用 `round` 而不是 `floor`，是因為 live `startTime` 已經是經過系統各層處理後的對齊時間，當時想避免微小秒差導致 slot 漂移。

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
