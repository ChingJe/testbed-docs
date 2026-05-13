# testbed 0512-19: UPF notify vs `pre_data` re-validation

## Purpose

這份報告重寫 `0512-19` 的結論，只保留在 `0513-13` 完成 pseudo-driver 對齊驗證之後，重新檢查仍然成立的內容。

這次的目標不是沿用舊報告的 slot 假設，而是回答更直接的問題：

1. `0512-19` 的 `NWDAF UPF VOLUME`，在正確對齊後，是否真的和 raw `pre_data` 一致？
2. 先前報告裡那些 per-IP mismatch，有多少其實只是對齊方法錯誤造成的誤判？
3. `0512-19` 真正剩下的現象是什麼？

## Data sources

- live output:
  - `5G_Infrastructure/.agent/tmp/0512-19/nwdaf.log`
- pseudo-driver timing metadata:
  - `5G_Infrastructure/.agent/tmp/0512-19/upf-ees.log`
  - `5G_Infrastructure/.agent/tmp/0512-19/upf-ees2.log`
- replay source:
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet`
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet`
- alignment reference:
  - [testbed-0513-13-pseudo-only-trace-alignment-validation.md](/home/chingje/testbed/docs/5g-infra/reports/testbed-0513-13-pseudo-only-trace-alignment-validation.md)

## Correct alignment model

`0513-13` 已經證明 pseudo-driver 的正確對齊不是 group-level `slot_raw = floor(ts/30)`，而是：

1. 對每個 UE / subscription 各自計算 `minTS`
2. 用
   - `wIdx = floor((ts - minTS) / 30)`
   做 per-subscription rebasing
3. live `startTime` 再透過 pseudo-driver 建立的 `GridAnchor` 映回：
   - `startTime = GridAnchor + 30s * wIdx`

`0512-19` 沒有 `pseudo_history_push` / `pseudo_phase2_push` 這種新 trace，所以這次使用 `grid alignment established` 裡的 `gridAnchor` 作為 live 對齊基準。

從當次 log 可讀到：

- `group1 gridAnchor = 2026-05-12 09:42:09Z`
- `group2 gridAnchor = 2026-05-12 09:42:10Z`

因此 `0512-19` 的 live `UPF VOLUME` 這次是用：

- `group1 wIdx = round((startTime - 09:42:09Z) / 30s)`
- `group2 wIdx = round((startTime - 09:42:10Z) / 30s)`

去和 raw parquet 的 per-UE rebased windows 對帳。

## Result summary

重算後，六個 IP 的結果完全一致：

- `10.10.0.1`: `180/218 exact`
- `10.10.0.2`: `180/218 exact`
- `10.10.0.3`: `180/218 exact`
- `10.100.0.1`: `180/218 exact`
- `10.100.0.2`: `180/218 exact`
- `10.100.0.3`: `180/218 exact`

這裡的 `218` 不是「218 個有效 dataset windows」。

實際結構是：

- `wIdx 0..179`: 有效 dataset windows，共 `180` 筆
- `wIdx 180..217`: dataset 播完後仍持續送出的 tail，共 `38` 筆

因此 `180/218 exact` 的真正意思是：

- **有效 dataset 區間 `180/180` 全部 exact match**
- 額外多出的 `38` 筆不屬於 raw dataset 本體

## What the tail actually is

重新檢查後，這 `38` 個 tail windows 並不是污染流量。

它們的共同特徵是：

- `total = 0`
- `ul = 0`
- `dl = 0`

也就是說：

- pseudo-driver / notify 在 dataset 播完後，仍持續送出一段全零 window
- 這個 tail 是真實存在的
- 但它不是先前報告推測的「非零異常流量尾段」

## What changed relative to the old report

這次重寫後，先前 `0512-19` 報告裡的幾個核心說法需要撤回：

1. `10.10.0.3` 和 `10.100.0.3` 並不是在有效 dataset 區間特別異常
2. `10.100.0.2` 也不是「前段正常、後段再變壞」
3. 舊報告中大量的 per-IP mismatch，主要來自：
   - 把 group-level slot 對齊直接套到實際使用 per-subscription rebasing 的 pseudo-driver 流程

也就是說，先前的 mismatch 結論大多不是資料真的錯，而是 comparison model 錯。

## What still remains true

`0512-19` 在重審後，真正還成立的現象只有兩個：

1. 有效 dataset windows 的 final `UPF VOLUME` 可以和 raw `pre_data` 完全對上
2. dataset 播完後，系統仍持續送出一段全零 tail windows

因此目前更準確的描述是：

- `0512-19` 並沒有證據顯示 final notify bytes 在有效區間被 kernel URR 或 pseudo-driver 算錯
- 先前的問題主要出在對齊方法

## Cross-session implication

把 `0512-19` 和 `0513-13` 放在一起看，目前最穩的結論是：

- 只要使用正確的 pseudo-driver 對齊模型
- raw `pre_data`
- `UPF EES` 發出的 notify payload
- `NWDAF UPF VOLUME`

都可以逐 window 完全對上。

這代表接下來跨 session 做這類比對時，應先做校正，不應再直接沿用舊版 group-level `slot_raw + 1` 的做法。

## Final conclusion

`0512-19` 這次重新驗證後的最終結論是：

- 有效 dataset 區間 `wIdx 0..179`：六個 IP 都是 `180/180 exact match`
- 額外 `wIdx 180..217`：是全零 tail，不是非零污染流量
- 先前報告中的主要 mismatch 敘述，應視為舊對齊方法造成的誤判

如果之後還要重新做同類比對，應以：

- per-subscription `minTS` rebasing
- pseudo-driver `GridAnchor`
- `startTime -> wIdx`

這組流程作為唯一基準。
