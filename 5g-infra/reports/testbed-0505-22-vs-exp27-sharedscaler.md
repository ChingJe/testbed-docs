# Testbed `0505-22` vs Replay `exp27_sharedscaler`

**Date:** 2026-05-05  
**Scope:** 重新比對 testbed 實驗 `0505-22` 與 replay 基準 `exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler`，並修正先前分析中**漏掉 phase1=900s** 所造成的對齊錯誤。

**Related:**

- [nwdaf-retrain-replay-current-setup.md](nwdaf-retrain-replay-current-setup.md)
- [upf-duplicate-starttime-hybrid-replay-2026-05-05.md](upf-duplicate-starttime-hybrid-replay-2026-05-05.md)

---

## 摘要

重新把 `breaking time = 900s` 納入後，最重要的修正是：

- `phase1` 已先一次性吃掉 `900s`
- 在 `30s` slot 下，等於 **先吃掉 30 個 absolute slots**
- 因此 testbed live phase2 的第 `i` 個 slot，應對齊 replay / `pre_data` 的 **absolute slot `i + 30`**

一旦補上這個 `+30 slots` offset，先前某些「很誇張的流量差距」會大幅收斂。修正後的結論如下：

1. `0505-22` 的前兩次 retrain 仍可合理視為 `CAT1→CAT2`、`CAT2→CAT3` 驅動。
2. 第三次 retrain 幾乎貼著 dataset 理論尾端發生，應視為尾端 artifact，不應納入正常 transition 評估。
3. 在 dataset 尾端異常前，**live `UPF raw`、live `NWDAF aggregated`、`exp27`、`pre_data` 的整體平均量級其實是接近的**。
4. 目前真正需要關注的，不再是「整體量級全面失真」，而是：
   - 少數 **per-slot / per-IP outlier**
   - 這些 outlier 在 `UPF raw` 就已經存在
   - `NWDAF latest aggregated slot` 只是把它們原樣反映出來

因此，這次修正後較準確的結論是：

- `0505-22` 比先前理解的更接近 replay `exp27_sharedscaler`
- `0505-22` 的主要問題不在整體流量量級，而在：
  - `group-test-001` / `group-test-002` 的少數異常 slot
  - 尤其是 `10.10.0.3` 與 `10.100.0.2/10.100.0.3`
  - 以及 dataset 尾端的 duplicate slot / 0-volume 行為

---

## 分析輸入

本文件主要依據：

- `.agent/tmp/0505-22/nwdaf.log`
- `.agent/tmp/0505-22/nwdafcfg.yaml`
- replay 基準：
  - `NWDAF/NWDAF/tools/retrain_replay/out/exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler/config/nwdafcfg.yaml`
  - `NWDAF/NWDAF/tools/retrain_replay/out/exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler/slots.parquet`
- dataset：
  - `go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet`
  - `go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet`
  - `file.json` 中 `breaking time = 900`
  - `total_duration_seconds ≈ 5429.18`

---

## 基準 replay 行為

`exp27_local_initial_buf5_nochronic_rwin1200_sharedscaler` 是目前 replay 主線基準：

- 第一次 retrain：
  - `00:34:30`
  - `group2`
  - `degradation`
- 第二次 retrain：
  - `01:06:00`
  - `group1`
  - `degradation`

相對 transition 的延遲：

- `CAT1→CAT2` 在 `00:30:00`，第一次 retrain 延遲 `+4m30s`
- `CAT2→CAT3` 在 `01:00:00`，第二次 retrain 延遲 `+6m00s`

---

## `0505-22` 的有效設置

### NWDAF 側

`0505-22/nwdafcfg.yaml` 與 replay 主線大致一致，關鍵值包括：

- `samplingInterval: 30`
- `checkInterval: 90`
- `minSamples: 2`
- `primaryMetric: WAPE`
- `recentBufferSize: 12`
- `minBufferSamples: 5`
- `minStd: 0.14`
- `fixedFloor: 0.05`
- `zScoreThreshold: 1.5`
- `decisionWindowSize: 5`
- `requiredHitsInWindow: 3`
- `chronicPolicy.enabled: false`
- `adrf.retrainWindow: 1200`
- `NUM_ROUNDS: 30`
- `ES_PATIENCE: 5`
- `LR_PATIENCE: 3`
- `local_epochs: 3`

刻意保留的 testbed 差異：

- `warmupDuration: 20`

### Group membership

這次已恢復為每組 3 個 SUPI：

- `group-test-001`: `001 / 002 / 003`
- `group-test-002`: `004 / 005 / 006`

`NWDAF` log 也證明：

- `Resolved groupId group-test-001 to 3 SUPIs`
- `Resolved groupId group-test-002 to 3 SUPIs`

以及 6 個 SMF subscription 都成功建立。

---

## 正確的時間線對齊

### Phase 2 起點

`0505-22` 中，第一個可見的 `latest aggregated slot` 為：

- `2026-05-05 13:09:00 UTC`

這裡可視為：

- live phase2 relative `slot_idx = 0`

### Phase 1 offset

因為 pseudo phase1 已先一次性輸入前 `900s` dataset：

- `900s / 30s = 30 slots`

所以在做 live vs replay / `pre_data` 對齊時，必須使用：

- `replay_abs_idx = live_phase2_idx + 30`

這是本次重新分析最重要的修正。  
先前若直接把 live phase2 `slot_idx = i` 對到 replay absolute `slot_idx = i`，會把前 30 個 phase1 slots 漏掉，導致流量形狀整段錯位。

### 推回 dataset 時間軸

由於 phase2 的 `slot_idx = 0` 對應 dataset `t = 900s`：

- dataset `t = 0` 約在 `12:54:00`
- `CAT1→CAT2`（dataset `t = 1800s`）約在 `13:24:00`
- `CAT2→CAT3`（dataset `t = 3600s`）約在 `13:54:00`

### Dataset 尾端

若 `total_duration_seconds ≈ 5429.18`，phase2 還剩：

- `5429.18 - 900 = 4529.18s`

因此 dataset 理論尾端約在：

- `13:09:00 + 1:15:29 ≈ 14:24:29`

這和本次尾端異常幾乎完全重合。

---

## `0505-22` 的 retrain 時間線

從 `nwdaf.log` 可確認本次共有 3 次 retrain：

1. `2026-05-05 13:30:19`
   - scope: `group-test-001`
   - reason: `degradation`
2. `2026-05-05 13:58:57`
   - scope: `group-test-001`
   - reason: `degradation`
3. `2026-05-05 14:25:16`
   - scope: `group-test-002`
   - reason: `degradation`

相對 CAT transition：

- 第一次 retrain：
  - 相對 `CAT1→CAT2 (13:24:00)`：`+6m19s`
- 第二次 retrain：
  - 相對 `CAT2→CAT3 (13:54:00)`：`+4m57s`
- 第三次 retrain：
  - 幾乎貼著 dataset 尾端 `14:24:29`

### 判讀

- **第一次 retrain**
  - 可合理視為 `CAT1→CAT2` 驅動
- **第二次 retrain**
  - 可合理視為 `CAT2→CAT3` 驅動
- **第三次 retrain**
  - 應視為 dataset 尾端 artifact
  - 不應納入正常 transition-driven retrain 計數

因此本次若只看「是否有兩個 transition-driven retrain」：

- **有**

---

## 比對方法

### 觀測線

本次同時比三條線：

1. live `UPF raw`
   - 從 `nwdaf.log` 解析每筆 `UPF VOLUME`
   - 對同一個 `ip + startTime` 保留最後一筆
2. live `NWDAF aggregated`
   - `nwdaf.log` 中的 `latest aggregated slot`
3. replay / dataset 基準
   - `exp27_sharedscaler/slots.parquet` 的 group aggregated `actualTotal`
   - `pre_data` 重新聚成 `30s` slot 的 group / per-UE baseline

### 去重與對齊

- live raw：
  - 同一個 `ip + slot_time` 保留最後一筆
- 分析範圍：
  - `phase2_start <= slot_time < 2026-05-05 14:24:00`
  - 也就是 dataset 尾端異常前
- replay / `pre_data`：
  - 使用 `replay_abs_idx = live_phase2_idx + 30`

---

## `UPF raw` vs `NWDAF aggregated`

### 直接結果

在 phase2 且尾端異常前，將 live `UPF raw` 依 `group + slot_time` 聚合後，再與同 slot 的 `NWDAF latest aggregated slot totalVol` 比較：

| Group | matched slots | `agg_total == raw_total_dedup` | mean abs diff |
|---|---:|---:|---:|
| `group-test-001` | `150` | `1.0` | `0` |
| `group-test-002` | `149` | `1.0` | `0` |

### 意義

這表示：

- 在 `latest aggregated slot` 這條觀測線上
- `NWDAF` 沒有額外把 `totalVol` 放大或縮小
- 它只是把 upstream `UPF raw` 的 group total 原樣反映出來

因此若只問：

> 某個 slot 的 `totalVol` 為什麼會很大？

答案應先回到：

- `UPF raw` 本身該 slot 是否已經很大

而不是優先懷疑 `NWDAF latest aggregated slot` 聚合邏輯。

---

## 整體量級：修正後其實接近

### live aggregated vs `exp27`

在 phase2 且尾端異常前，使用 `+30 slots` 對齊後：

| Group | Live mean | Replay mean | Median ratio | P90 ratio | MAPE | Correlation |
|---|---:|---:|---:|---:|---:|---:|
| `group-test-001` | `17.96M` | `17.42M` | `1.008x` | `12.96x` | `3.30` | `-0.02` |
| `group-test-002` | `41.49M` | `41.37M` | `0.998x` | `5.68x` | `3.47` | `0.74` |

### live raw vs `pre_data`

同樣的 phase2 與 `+30 slots` 對齊下：

| Group | Live raw mean | `pre_data` mean | Median ratio | P90 ratio | MAPE | Correlation |
|---|---:|---:|---:|---:|---:|---:|
| `group-test-001` | `17.96M` | `17.42M` | `1.008x` | `12.96x` | `3.30` | `-0.02` |
| `group-test-002` | `41.60M` | `41.43M` | `0.999x` | `5.68x` | `3.45` | `0.75` |

### 判讀

這是本次修正後最重要的新結論：

- **整體平均量級其實是接近的**
- 先前覺得「量級差很大」的主要原因，是把 live phase2 錯對到 replay absolute slot 0，而不是 absolute slot 30

因此，現在不應再說：

- `0505-22` 的整體流量和 `exp27` 差很多

更準確的說法是：

- `0505-22` 的**整體量級已大致對齊**
- 問題集中在**少數極端 slot / per-IP outlier**

---

## 分段結果

以下窗口以 live phase2 relative `slot_idx` 切分：

- `pre_CAT12`: `0 ~ 29`
- `CAT12_to_CAT23`: `30 ~ 89`
- `CAT23_to_tail`: `90 ~ 150`

### live aggregated vs `exp27`

#### `group-test-001`

| Window | Live mean | Replay mean | Median ratio | Correlation | MAPE |
|---|---:|---:|---:|---:|---:|
| `pre_CAT12` | `6.62M` | `6.74M` | `1.000x` | `0.944` | `0.104` |
| `CAT12_to_CAT23` | `9.24M` | `9.21M` | `1.004x` | `0.131` | `2.271` |
| `CAT23_to_tail` | `32.35M` | `30.97M` | `1.236x` | `-0.327` | `5.918` |

#### `group-test-002`

| Window | Live mean | Replay mean | Median ratio | Correlation | MAPE |
|---|---:|---:|---:|---:|---:|
| `pre_CAT12` | `5.17M` | `5.55M` | `1.001x` | `0.937` | `0.090` |
| `CAT12_to_CAT23` | `7.79M` | `7.75M` | `0.972x` | `0.021` | `6.186` |
| `CAT23_to_tail` | `94.22M` | `93.76M` | `0.960x` | `0.522` | `2.423` |

### 判讀

- `pre_CAT12`：
  - 兩個 group 都很接近 replay
- `CAT12_to_CAT23`：
  - 平均量級仍接近，但波形相關性下降
- `CAT23_to_tail`：
  - 平均量級仍接近
  - 但局部 outlier 變多，導致：
    - `group-test-001` correlation 變差
    - `group-test-002` 仍有部分 slot 異常偏大

因此這裡真正的問題，不是「整段平均量級失真」，而是：

- **某些 slot 內有少數 UE/IP 值暴衝**
- 它們會破壞局部波形，但不會把整段 mean 完全帶走

---

## per-IP 問題點

### `group-test-001`

若以 same-slot、same-IP 比對 live raw 與 `pre_data`：

- `10.10.0.1` 在多個異常 slot 上可完全對上
- `10.10.0.2` 多數 slot 也只差小量
- 真正最異常的是 **`10.10.0.3`**

代表例子：

| phase2 idx | abs idx | live `10.10.0.3` | `pre_data` `10.10.0.3` |
|---:|---:|---:|---:|
| `52` | `82` | `34.86M` | `0.94M` |
| `100` | `130` | `84.78M` | `1.29M` |
| `131` | `161` | `84.86M` | `1.85M` |
| `133` | `163` | `84.89M` | `1.12M` |

也就是：

- `group-test-001` 的 group-level 異常，主要是由 `10.10.0.3` 撐出來的
- 不是整個 group 三個 UE 都一起失真

### `group-test-002`

`group-test-002` 的型態更分明：

- `10.100.0.1` 在部分異常 slot 上其實能 exact 對上
- 真正異常的是 **`10.100.0.2`** 和 **`10.100.0.3`**

代表例子：

| phase2 idx | abs idx | live `10.100.0.2` | `pre_data` `10.100.0.2` | live `10.100.0.3` | `pre_data` `10.100.0.3` |
|---:|---:|---:|---:|---:|---:|
| `97` | `127` | `85.05M` | `2.75M` | `84.99M` | `1.36M` |
| `103` | `133` | `85.11M` | `22.59M` | `1.20M` | `1.25M` |
| `127` | `157` | `1.32M` | `1.76M` | `0.71M` | `84.96M` |

判讀：

- `group-test-002` 的問題不是 group total 全面失真
- 而是某些 slot 內：
  - 某個 IP 仍然 exact
  - 另一些 IP 卻暴衝到 `80M+`

### 不是簡單的 IP permutation

我另外測了：

- 同一個 group
- 同一個 slot
- 允許 3 個 IP 在 slot 內做 `3!` permutation

結果：

- 對上述最異常的 slot，**最佳 permutation 仍然是 identity**
- 也就是說，這些 outlier 不是「三個 UE 值配錯 IP」就能解釋

因此目前較合理的解讀是：

- 問題在 upstream source slot value 本身
- 不是單純的 group 內 IP label 互換

---

## 從 `go-upf` 原始碼可直接支持的潛在問題點

本節只列出目前從 `go-upf` 程式碼**可以直接支持**、而且和本報告中的「某些 UE/IP 會出現異常大 raw value」直接相關的潛在問題點。

### 問題點 1：Kernel report 在 pseudo mode 下會被重寫到「目前 phase2 window」

位置：

- `internal/ees/aggregator.go`
- `PushReport()`

關鍵邏輯：

- 先呼叫 `pseudoDriver.GetPhase2Window()`
- 若 `simActive == true`
  - `m.StartTime = p2Start`
  - `m.EndTime = p2End`

這表示：

- kernel report 原本自己的時間窗
- 不會保留下來直接進 buffer
- 而是被**強制對齊到目前 pseudo phase2 window**

對本報告中的異常流量而言，這是最核心的高風險點之一。  
只要某個 kernel report 到達時間跨越 tick 邊界，它就可能被改寫到不該屬於它的 logical slot。

### 問題點 2：同一個 `UE IP + StartTime` 的資料會被直接累加

位置：

- `internal/ees/aggregator.go`
- `consolidateReports()`

關鍵邏輯：

- key 使用：
  - `UE IP + StartTime.Unix()`
- 若 key 相同，直接：
  - `ULBytesDelta += ...`
  - `DLBytesDelta += ...`
  - packet counters 也一併相加

這代表：

- 對同一個 `UE IP + StartTime`
- code 不會去判斷這是不是重複 source-side report
- 而是預設它們應該合併

若 timing 邊界導致：

- 同一段 kernel traffic
- 兩次落到同一個 logical slot

那最終結果不是覆蓋，而是直接形成**異常大值**。

### 問題點 3：Phase 2 與 Kernel 被設計成在同一輪 Tick 收集，對 timing 邊界非常敏感

位置：

- `internal/ees/pseudodriver.go`
- `simulateFutureRealTime()`

關鍵邏輯：

- `WaitForTick()`
- 發布目前 phase2 window
- `PushLiveMeasures(sub, winMeasures)`

註解明確說設計目標是：

- 在某輪 `TickOnce[N]` 前
- 讓 pseudo phase2 與 kernel 都把 window `N` 的資料放進 buffer
- 再由 `TickOnce[N]` 一起收走

這個設計在理想情況下沒有問題，但一旦：

- kernel report 稍晚
- 或 phase2 window 已切到下一格

就可能出現：

- 某個 slot 被多次算進來
- 或原本屬於 `N` 的資料被掛到 `N+1`

### 問題點 4：`kernelReady + sleep 50ms` 會放大 tick 邊界競態

位置：

- `internal/ees/aggregator.go`
- `Run()`

關鍵邏輯：

- ticker 觸發後，先等 `kernelReady`
- 收到 signal 後，再固定 `sleep 50ms`

這表示：

- 某輪 `TickOnce()` 到底吃到哪些 kernel report
- 不只取決於 pseudo window
- 還取決於：
  - signal 何時到
  - sleep 期間又有沒有更多 report 進來
  - runtime scheduling

這會讓問題呈現出：

- 不是每次都發生
- 但一旦發生，就可能連續好幾個 slot 出現同一種異常大值

### 問題點 5：first-URR ticker re-sync 可能改變 phase2 與 kernel 的相位關係

位置：

- `internal/ees/aggregator.go`
- `PushLiveMeasures()`

關鍵邏輯：

- 第一次 live URR 進來時，會送 `tickerReset`
- 重新對齊 Aggregator ticker phase

這代表：

- phase2 已經在跑時
- ticker 的相位仍可能被重設一次

它不一定是唯一主因，但在 hybrid mode 中，這是一個足以影響：

- 哪些 report 落進哪一輪 Tick
- 哪些 slot 會被錯誤累加

的高風險點。

### 問題點 6：目前沒有 source-side idempotency guard

位置：

- `internal/ees/aggregator.go`
- `PushReport()`
- `consolidateReports()`

目前對於：

- 同一個 `UE IP`
- 同一個 logical slot
- 同一份或等價的 kernel usage

沒有額外的 idempotency key 或防重保護。

因此只要 source-side timing 讓同一段 traffic 兩次落到同一個 `UE IP + StartTime`，它就會被加總成異常大值。

---

## 異常點數量與時間線分佈

為了回答「到底有多少異常點」以及「它們分佈在哪些 CAT 區段」，本節使用較直觀且可重複的定義：

- 比對對象：
  - same-slot
  - same-IP
  - live `UPF raw`
  - 對 `pre_data` per-UE `30s` slot baseline
- 對齊方式：
  - 使用 `replay_abs_idx = live_phase2_idx + 30`
- 指標：
  - `fold_err = max(live/pre, pre/live)`

以下主要看三個門檻：

- `> 5x`
- `> 10x`
- `> 20x`

其中 `> 10x` 可視為「明顯異常」；`> 20x` 可視為「非常明顯異常」。

### 總數

phase2 且 dataset 尾端異常前，總共有：

- per-IP slot 點：`770`
- per-group slot 點：`300`

若用 `fold_err > 10x`：

- per-IP 異常點：`124 / 770`
- per-group slot 異常點：`60 / 300`

若用 `fold_err > 20x`：

- per-IP 異常點：`86 / 770`
- per-group slot 異常點：`36 / 300`

若用更嚴格的 `fold_err > 50x`：

- per-IP 異常點：`44 / 770`
- per-group slot 異常點：`2 / 300`

### 主要異常來源 IP

以 `fold_err > 10x` 計：

| IP | 異常點數 |
|---|---:|
| `10.10.0.3` | `54` |
| `10.100.0.3` | `41` |
| `10.100.0.2` | `15` |
| `10.10.0.2` | `14` |
| `10.10.0.1` | `0` |
| `10.100.0.1` | `0` |

這再次說明：

- `10.10.0.1`、`10.100.0.1` 幾乎沒有問題
- 異常主要集中在：
  - `10.10.0.3`
  - `10.100.0.3`
  - `10.100.0.2`
  - 次要是 `10.10.0.2`

### 時間線切分

以下將 live phase2 relative slot 切為五段：

- `CAT1_tail`: `0 ~ 29`
- `CAT2_early`: `30 ~ 59`
- `CAT2_late`: `60 ~ 89`
- `CAT3_early`: `90 ~ 119`
- `CAT3_late`: `120 ~ 150`

### per-IP 分佈（`fold_err > 10x`）

| IP | CAT1 tail | CAT2 early | CAT2 late | CAT3 early | CAT3 late | Total |
|---|---:|---:|---:|---:|---:|---:|
| `10.10.0.1` | `0` | `0` | `0` | `0` | `0` | `0` |
| `10.10.0.2` | `0` | `5` | `4` | `2` | `3` | `14` |
| `10.10.0.3` | `2` | `11` | `9` | `13` | `19` | `54` |
| `10.100.0.1` | `0` | `0` | `0` | `0` | `0` | `0` |
| `10.100.0.2` | `0` | `0` | `0` | `11` | `4` | `15` |
| `10.100.0.3` | `1` | `7` | `6` | `14` | `13` | `41` |

### 如何解讀

- `10.10.0.3`
  - 在 `CAT2_early` 就已經開始明顯異常（`11` 次）
  - 到 `CAT3_late` 最嚴重（`19` 次）
- `10.100.0.3`
  - 從 `CAT2_early` 起就持續異常
  - `CAT3_early` / `CAT3_late` 都很明顯
- `10.100.0.2`
  - 幾乎不是早段問題
  - 主要在 `CAT3_early`（`11` 次）後才出現
- `10.10.0.2`
  - 是較次要的異常來源
  - 分佈在 `CAT2` 到 `CAT3` 間，但數量明顯少於 `10.10.0.3`

所以如果要用一句話概括時間分佈：

- `group-test-001` 的核心問題從 `CAT2` 就開始，由 `10.10.0.3` 主導，並在 `CAT3_late` 進一步惡化。
- `group-test-002` 的異常由 `10.100.0.3` 持續主導，而 `10.100.0.2` 主要在 `CAT3` 才變得明顯。

### per-group 分佈（`fold_err > 10x`）

把每個 group 的總量拿來比 `pre_data` group total：

| Group | CAT1 tail | CAT2 early | CAT2 late | CAT3 early | CAT3 late |
|---|---:|---:|---:|---:|---:|
| `group-test-001` | `0` | `6` | `5` | `9` | `19` |
| `group-test-002` | `1` | `7` | `6` | `3` | `4` |

這裡可以看到：

- `group-test-001`
  - group total 異常在後段最明顯
  - 尤其 `CAT3_late` 達到 `19` 個異常 slot
- `group-test-002`
  - group total 異常比較平均分布在 `CAT2`
  - 到 `CAT3_late` 反而不如 `group-test-001` 那麼集中

這和 per-IP 分析一致：

- `group-test-001` 是少數大 outlier 把 group total 在後段明顯拉壞
- `group-test-002` 則比較像 `CAT2` 起就持續存在 source-side 問題

---

## 尾端異常與第三次 retrain

本次在：

- `2026-05-05 14:24:00` 左右

開始出現長段 `totalVol=0` 的 `latest aggregated slot`，並且：

- `2026-05-05 14:24:58`
  - 開始出現大規模 dedup warning
- `2026-05-05 14:25:16`
  - 第三次 retrain 觸發

這與 dataset 理論尾端 `14:24:29` 高度重合。

因此第三次 retrain 應視為：

- dataset 尾端 artifact
- 不應納入正常 transition-driven retrain 評估

---

## 結論

### 結論 1

把 `phase1 = 900s` 納入後，live phase2 與 replay / `pre_data` 的正確對齊方式應是：

- `replay_abs_idx = live_phase2_idx + 30`

先前若忽略這點，會高估流量差距。

### 結論 2

修正後，`0505-22` 與 `exp27_sharedscaler` 的**整體平均量級其實接近**，不應再用「整段量級失真」來描述。

### 結論 3

`NWDAF latest aggregated slot` 與 live `UPF raw_dedup` 在 phase2、尾端異常前的逐 slot total 完全一致：

- `eq_frac = 1.0`
- `mean abs diff = 0`

所以 `NWDAF` 不是在 `latest aggregated slot` 這一步把量放大。

### 結論 4

目前真正的問題是：

- 少數 **per-slot / per-IP outlier**
- 尤其：
  - `group-test-001` 的 `10.10.0.3`
  - `group-test-002` 的 `10.100.0.2`、`10.100.0.3`

### 結論 5

第三次 retrain 幾乎貼著 dataset 尾端發生，應視為尾端 artifact，而不是正常 transition retrain。

---

## 建議下一步

1. 之後所有 live vs replay 比對，都必須明確納入 `+30 slots` phase1 offset。
2. 若要追根究底，應優先分析：
   - `10.10.0.3`
   - `10.100.0.2`
   - `10.100.0.3`
   在異常 slot 上為何會出現 `80M+` 等級的 raw value。
3. 既然 `NWDAF aggregated` 與 `UPF raw_dedup` 已逐 slot一致，追查重點應放回：
   - `UPF-EES / go-upf` source-side 行為
   - hybrid replay/live aggregation
   - dataset 尾端與 duplicate slot 問題
