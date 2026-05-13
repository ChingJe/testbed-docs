# testbed 0512-19: UPF notify vs `pre_data` comparison method

## Purpose

這份文件只做一件事：

- 把 `0512-19` 這次重新驗證使用的比對方法固定下來

它不再保留舊版那些混合了錯誤假設與中途推論的內容。  
若要看這次重寫後的結果，請看：

- [testbed-0512-19-upf-notify-vs-predata-all-ip.md](/home/chingje/testbed/docs/5g-infra/reports/testbed-0512-19-upf-notify-vs-predata-all-ip.md)

若要看這個方法是怎麼從 `0513-13` 校正出來的，請看：

- [testbed-0513-13-pseudo-only-trace-alignment-validation.md](/home/chingje/testbed/docs/5g-infra/reports/testbed-0513-13-pseudo-only-trace-alignment-validation.md)

## Scope

這份方法只涵蓋：

1. raw `pre_data parquet`
2. `0512-19` 的 `upf-ees.log` / `upf-ees2.log` 中 `gridAnchor`
3. `0512-19` 的 `nwdaf.log` 中 `UPF VOLUME`

它不涵蓋：

- replay prediction / trigger 分析
- `gnb/ue` log
- kernel URR trace 細節

## Data inputs

- live output:
  - `5G_Infrastructure/.agent/tmp/0512-19/nwdaf.log`
- pseudo-driver anchor:
  - `5G_Infrastructure/.agent/tmp/0512-19/upf-ees.log`
  - `5G_Infrastructure/.agent/tmp/0512-19/upf-ees2.log`
- replay source:
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet`
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet`

## Why the old method was wrong

舊版 `0512-19` 方法混亂的根本原因，是把這種 group-level 規則當成通用真理：

- `slot_raw = floor(ts/30)`
- `pre_abs_slot = slot_raw + 1`
- 用 group-level phase2 anchor 直接對全部 IP

但 `0513-13` 已證明 pseudo-driver 的實際邏輯不是這樣，而是：

1. 先對每個 subscription / UE 自己找 `minTS`
2. 再做 per-subscription rebasing
3. 再切 30s `wIdx`

因此 `0512-19` 重審時，也必須使用同一套模型。

### Plain-language explanation

白話講，舊方法錯在它偷偷假設：

- 同一個 group 裡所有 UE / IP
- 都是從同一個時間點開始算第 0 個 30s window

但實際資料不是這樣。

pseudo-driver 真正做的是：

- 每個 subscription / UE 先看「自己的第一筆資料是幾秒」
- 再把那個時刻當成自己的 `0s`
- 然後才開始切 30s window

所以如果你拿「整個 group 共用一支碼表」去對「每個 UE 自己各自歸零」的輸出，就會把原本正確的數值看成錯位。

## Direct evidence: different UEs really do start at different timestamps

這件事不是推論，而是 raw `pre_data` 可以直接驗到的事實。

從：

- `pre_data/group1/training_packets_run001.parquet`
- `pre_data/group2/training_packets_run001.parquet`

直接取每個 `ue_ip` 的最早 `ts`，得到：

### group1

- `10.10.0.1: min_ts = 0s`
- `10.10.0.2: min_ts = 15s`
- `10.10.0.3: min_ts = 30s`

### group2

- `10.100.0.1: min_ts = 0s`
- `10.100.0.2: min_ts = 15s`
- `10.100.0.3: min_ts = 30s`

這就是「每個 IP 的起始點不同」的直接證據。

因此 pseudo-driver 若對每個 UE 自己做 rebasing：

- `10.10.0.1` 的 `0s` -> 自己的 `wIdx=0`
- `10.10.0.2` 的 `15s` -> 自己的 `wIdx=0`
- `10.10.0.3` 的 `30s` -> 自己的 `wIdx=0`

那就和 group-level `slot_raw = floor(ts/30)` 完全不是同一個對齊方式。

### Why the old group-level slot model misclassifies correct data

舉最簡單的例子：

- 對 `10.10.0.2` 而言，第一筆資料出現在 `15s`
- 舊方法會把它放進：
  - `slot_raw = floor(15/30) = 0`
- 看起來好像也還是第 0 格

但關鍵問題是往後每 30s window 的分組方式不同：

- 舊方法的第 0 格是 `[0s, 30s)`
- pseudo-driver 對 `10.10.0.2` 的第 0 格其實是 `[15s, 45s)`

同理：

- `10.10.0.3` 的第 0 格不是 `[0s, 30s)`
- 而是 `[30s, 60s)`

所以一旦用錯時間原點，整串 windows 都會錯半格或一格，最後看起來像大規模 mismatch，但其實只是 slot 對齊錯了。

## Correct model

### Step 1. Build expected windows from raw parquet

對每個 UE 分開處理：

1. 讀該 UE 的所有 packet rows
2. 計算：
   - `minTS = 這個 UE 在 parquet 中的最早 timestamp`
3. 對每列計算：
   - `wIdx = floor((ts - minTS) / 30)`
4. 依 `(wIdx, direction)` 聚合 `len`

方向對應是：

- `direction = 0 -> ul`
- `direction = 1 -> dl`

最後每個 UE 得到：

- `expected[ueIp][wIdx] = (ul, dl, total)`

這一步是 `0513-13` 驗證後固定下來的正確基準。

### Step 2. Recover live anchors from UPF EES logs

`0512-19` 沒有 `pseudo_history_push` / `pseudo_phase2_push`，所以不能像 `0513-13` 那樣直接由 trace rows 反推 anchor。

這次改用 `grid alignment established` 的 `gridAnchor`。

從 log 直接可讀到：

- `group1 gridAnchor = 2026-05-12 09:42:09Z`
- `group2 gridAnchor = 2026-05-12 09:42:10Z`

對應來源：

- `upf-ees.log` -> `group1`
- `upf-ees2.log` -> `group2`

### Step 3. Parse `UPF VOLUME` rows from `nwdaf.log`

從 `nwdaf.log` 抽出所有：

```text
UPF VOLUME: ip=..., startTime=..., total=..., ul=..., dl=...
```

保留欄位：

- `ip`
- `startTime`
- `ul`
- `dl`
- `total`

### Step 4. Convert live `startTime` back to `wIdx`

對每個 `UPF VOLUME` row：

1. 依 IP 判斷它屬於哪個 group
   - `10.10.x.x -> group1`
   - `10.100.x.x -> group2`
2. 用該 group 的 `gridAnchor`
3. 計算：
   - `wIdx = round((startTime - gridAnchor) / 30s)`

這樣就把 live notify 的時間軸映回 pseudo-driver 的 window index。

### Step 5. Compare `(ueIp, wIdx)` exact values

對每一個 live row：

- 找 `expected[ueIp][wIdx]`
- 比對：
  - `live_ul == expected_ul`
  - `live_dl == expected_dl`
  - `live_total == expected_total`

若 raw dataset 已經沒有該 `wIdx`，則 expected 視為不存在。

## Interpretation of the result

在 `0512-19` 這次重新驗證後，會看到每個 IP 都是：

- `180/218 exact`

這個數字要這樣解讀：

- `wIdx 0..179`: raw dataset 的有效 windows，共 `180` 筆
- `wIdx 180..217`: live side 額外多出的 `38` 筆

重新檢查後，這 38 筆都是：

- `ul=0`
- `dl=0`
- `total=0`

因此它們不是 mismatch，而是：

- **dataset 播完後仍持續送出的 zero tail**

所以 `180/218 exact` 實際上代表：

- 有效資料 `180/180` 全對
- 其餘 `38` 筆只是 dataset 外的全零尾段

## What this method does not assume anymore

這次重寫後，下面這些都不再被當成預設真理：

- `pre_abs_slot = slot_raw + 1`
- group-level `slot_raw` 可以直接代表 pseudo-driver window
- `nwdaf.log` 最早 `UPF VOLUME` 一定能直接當所有 session 的通用 anchor
- `tail` 一定代表非零異常流量

這些都必須透過：

- pseudo-driver code path
- trace
- 或 `gridAnchor`

重新校正。

## Recommended validation order for future sessions

之後若要在新 session 重建比對，建議固定用下面順序：

1. 先確認 pseudo-driver 實際使用的是哪種 rebasing 模型
2. 若有新 trace，優先用 `pseudo_history_push` / `pseudo_phase2_push`
3. 若沒有 trace，再退回 `gridAnchor`
4. 先做 `raw parquet -> pseudo/live` 的 `wIdx` 校正
5. 最後才統計 mismatch / tail / exact rate

不要再直接以 group-level `slot_raw` 做第一版結論。

## Practical note

這次 `0512-19` 重新驗證的過程，實際上已經被 `0513-13` 的驗證工具邏輯驗證過。

如果之後要做類似 session 的檢查，可以參考：

- `/home/chingje/testbed/5G_Infrastructure/.agent/verify_upf_pseudo_alignment.py`

但要注意：

- 這支工具是按 `0513-13` 的 trace 結構寫的
- `0512-19` 由於沒有 pseudo push trace，所以本次仍需要手動改用 `gridAnchor` 邏輯

## Final takeaway

`0512-19` 這次方法重寫後，唯一應保留的核心規則是：

- **先做 per-subscription / per-UE rebasing**
- **再用 pseudo-driver 的 live anchor 映回 `wIdx`**
- **最後才逐 window 做 exact comparison**

若缺少這一步，得到的大量 mismatch 幾乎沒有解讀價值。
