# testbed 0512-19: UPF notify vs `pre_data` per-IP full check

## Purpose

這份報告重新檢查 `0512-19` testbed run 中，NWDAF 從 UPF EES 收到的 per-IP `UPF VOLUME` 是否真的和 `go-upf` 的 `pre_data` 一致。

這次檢查的重點有兩個：

1. 不只看先前報告提到的幾個 IP，而是檢查所有 IP。
2. 對於看起來「正常」的部分，要確認是否是 **逐 slot、逐欄位完全相等**。

這裡的「完全相等」定義很嚴格：

- 若 `pre_data` 某個 30s slot 的 `ul=1222`，則 `UPF VOLUME` 在對應 slot 也必須是 `ul=1222`
- `dl` 也要完全相等
- `total=ul+dl` 也要完全相等

不是只看量級接近，也不是只看 group-level aggregate。

## Data sources

- testbed run log:
  - `5G_Infrastructure/.agent/tmp/0512-19/nwdaf.log`
- pseudo-driver metadata:
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/file.json`
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/file.json`
- per-IP replay source:
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group1/training_packets_run001.parquet`
  - `5G_Infrastructure/go-upf-ess/go-upf/pre_data/group2/training_packets_run001.parquet`
- previous reference:
  - `docs/5g-infra/reports/testbed-0505-22-vs-exp27-sharedscaler.md`

## Comparison method

### 1. Parse live UPF notify rows

從 `nwdaf.log` 解析所有這種格式的 log：

```text
UPF VOLUME: ip=..., startTime=..., total=..., ul=..., dl=...
```

每一筆保留：

- `ip`
- `startTime`
- `ul`
- `dl`
- `total`

### 2. Aggregate `pre_data` to the same 30s slot granularity

`training_packets_run001.parquet` 內每筆是 packet-level row，需要先聚成 30s slot。

做法：

- `slot_raw = floor(ts / 30s)`
- 依 `ue_ip + slot_raw + direction` 聚合 `len`

方向欄位經實際對照後，這次使用：

- `direction=0 -> ul`
- `direction=1 -> dl`

這一點不是猜測，是直接拿已知正常的 IP 對 live `UPF VOLUME` 驗證出來的。

### 3. Align pseudo-driver phase1 with live time

`group1/file.json` 與 `group2/file.json` 都寫明：

- `breaking time = 900`

也就是 pseudo-driver phase1 先消耗 900s，phase2 才開始進入我們關心的 replay 流量。

依 `nwdaf.log` 中最早的 aggregated slot：

- `group1` phase2 anchor: `2026-05-12T09:56:39Z`
- `group2` phase2 anchor: `2026-05-12T09:56:40Z`

因此 live `UPF VOLUME` 的 slot index 用下面方式定義：

- `group1_abs_slot = round((startTime - 09:56:39Z)/30s) + 30`
- `group2_abs_slot = round((startTime - 09:56:40Z)/30s) + 30`

### 4. Resolve the off-by-one between live slot numbering and raw `pre_data` slots

直接對照已知正常的 IP 後，可以確認：

- live `abs_slot=30`
- 對應的是 `pre_data` 的 `slot_raw=29`

也就是在這次 `0512-19` run 裡，正確的對齊應為：

- `pre_abs_slot = slot_raw + 1`

這一步不能省略，否則會把明明完全一致的正常 IP 錯判成全部不相等。

### 5. Exact-match criteria

對每個 `ip + abs_slot`，比較：

- `live_ul == pre_ul`
- `live_dl == pre_dl`
- `live_total == pre_total`

另外依 slot 區間拆成：

- `phase1`: `abs_slot < 30`
- `cat1_phase2`: `30 <= abs_slot < 90`
- `cat2_phase2`: `90 <= abs_slot < 150`
- `cat3_phase2`: `150 <= abs_slot < 182`
- `tail`: `abs_slot >= 182`

這裡的意思是：

- `cat3_phase2` 仍然是 pseudo-driver 還在播放的有效資料
- `tail` 才是 pseudo-driver 播完之後的零流量尾段

因此：

- `cat3_phase2` 仍然有真實比較意義
- `tail` 的 exact 數字多半只代表雙方都進入播放結束後的 0 流量狀態

若目的是分析 retrain trigger，最重要的仍是 `phase1`、`cat1_phase2`、`cat2_phase2`。

## Result summary

### Overall exact-total match ratio

| IP | exact total / all slots | ratio | interpretation |
|---|---:|---:|---|
| `10.10.0.1` | 218 / 218 | 100.0% | 完全正常 |
| `10.100.0.1` | 218 / 218 | 100.0% | 完全正常 |
| `10.100.0.2` | 94 / 218 | 43.1% | 早段部分接近，但不是全程 exact |
| `10.10.0.3` | 40 / 218 | 18.3% | 嚴重異常 |
| `10.10.0.2` | 38 / 218 | 17.4% | 明顯異常 |
| `10.100.0.3` | 37 / 218 | 17.0% | 嚴重異常 |

### Phase-by-phase exact-total match

| IP | phase1 | cat1_phase2 | cat2_phase2 | cat3_phase2 | tail |
|---|---:|---:|---:|---:|---:|
| `10.10.0.1` | 29/29 | 60/60 | 60/60 | 32/32 | 37/37 |
| `10.100.0.1` | 29/29 | 60/60 | 60/60 | 32/32 | 37/37 |
| `10.100.0.2` | 2/29 | 30/60 | 25/60 | 0/32 | 37/37 |
| `10.10.0.3` | 1/29 | 2/60 | 0/60 | 0/32 | 37/37 |
| `10.10.0.2` | 1/29 | 0/60 | 0/60 | 0/32 | 37/37 |
| `10.100.0.3` | 0/29 | 0/60 | 0/60 | 0/32 | 37/37 |

## What is truly normal

這次最重要的結論是：

- `10.10.0.1` 是 **逐 slot、逐欄位完全等於** `pre_data`
- `10.100.0.1` 也是 **逐 slot、逐欄位完全等於** `pre_data`

也就是說，這次 testbed 並不是所有 IP 都出問題；至少有兩條流量路徑是完全對上的。

### Exact examples

`10.10.0.1`

| abs slot | startTime | live ul | pre ul | live dl | pre dl | live total | pre total |
|---|---|---:|---:|---:|---:|---:|---:|
| 30 | `09:56:39Z` | 202450 | 202450 | 211973 | 211973 | 414423 | 414423 |
| 31 | `09:57:09Z` | 196727 | 196727 | 211766 | 211766 | 408493 | 408493 |
| 32 | `09:57:39Z` | 185285 | 185285 | 212135 | 212135 | 397420 | 397420 |

`10.100.0.1`

| abs slot | startTime | live ul | pre ul | live dl | pre dl | live total | pre total |
|---|---|---:|---:|---:|---:|---:|---:|
| 30 | `09:56:40Z` | 198057 | 198057 | 138 | 138 | 198195 | 198195 |
| 31 | `09:57:10Z` | 205518 | 205518 | 92 | 92 | 205610 | 205610 |
| 32 | `09:57:40Z` | 196293 | 196293 | 138 | 138 | 196431 | 196431 |

所以如果 `pre_data` 在某個時間點寫的是 `1222`，live UPF EES 並不是做不到給 `1222`。至少在這兩個 IP 上，它就是逐筆 exact match。

## IPs that are close but not exact

### `10.100.0.2`

`10.100.0.2` 在 `CAT1 phase2` 前半段看起來接近，但不是 exact。

特徵：

- `cat1_phase2` 有 `30/60` 個 slot 在 `total` 上完全相等
- 但 `ul/dl` 常出現小幅錯位
- `total` 常有 `+44/-45/+1/-89/+89` 這種小偏差

代表例子：

| abs slot | live total | pre total | delta |
|---|---:|---:|---:|
| 30 | 2625 | 2581 | +44 |
| 31 | 2581 | 2626 | -45 |
| 32 | 2670 | 2625 | +45 |
| 34 | 2626 | 2625 | +1 |

對應到欄位可看到：

- `ul` 有時 exact
- `dl` 會在相鄰 slot 之間出現 44/45 byte 級別的偏移

這種情況不能算正常，因為它不符合「同 slot、同欄位 exact equal」的要求；但它和 `10.10.0.3` / `10.100.0.3` 那種爆量級錯誤是不同等級的問題。

另外，`10.100.0.2` 到了 `cat3_phase2` 已經是 `0/32` exact-total slots，不再只是小幅偏移。

## IPs with clear per-IP problems

### `10.10.0.2`

`10.10.0.2` 不是完全爆掉，但在 `cat1_phase2`、`cat2_phase2`、`cat3_phase2` 都沒有任何 exact-total slot。

`cat1_phase2` 的代表異常：

| abs slot | live total | pre total | fold |
|---|---:|---:|---:|
| 63 | 194338 | 2448737 | 12.6x |
| 81 | 198060 | 2575100 | 13.0x |
| 33 | 15637520 | 7378409 | 2.1x |

它不是單純小誤差，而是已經明顯偏離 `pre_data`。

### `10.10.0.3`

`10.10.0.3` 仍然是最嚴重的問題點之一。

`cat1_phase2` 只有 `2/60` exact-total slots，`cat2_phase2` 與 `cat3_phase2` 都是 `0`。

代表異常：

| abs slot | live total | pre total | fold |
|---|---:|---:|---:|
| 77 | 46294615 | 132472 | 349.5x |
| 108 | 81918 | 37204681 | 454.2x |
| 127 | 732835 | 85201384 | 116.3x |

### `10.100.0.3`

`10.100.0.3` 也仍然是核心問題點。

`cat1_phase2`、`cat2_phase2`、`cat3_phase2` 都沒有任何 exact-total slots。

代表異常：

| abs slot | live total | pre total | fold |
|---|---:|---:|---:|
| 65 | 44136744 | 211037 | 209.1x |
| 140 | 626041 | 85185259 | 136.1x |
| 124 | 679013 | 85397361 | 125.8x |

## Comparison with the previous report

和 `testbed-0505-22-vs-exp27-sharedscaler.md` 相比，這次可以更明確地下結論：

- `10.10.0.3` 的問題仍然存在，而且依然很嚴重
- `10.100.0.3` 的問題仍然存在，而且依然很嚴重
- `10.100.0.2` 這次在 `CAT1 phase2` 前半段不是完全失真，但仍然沒有達到 exact-match 要求
- `10.10.0.2` 這次也不能視為正常，已經有明顯 per-IP 偏差

## Further investigation with `upf-ees` logs and `go-upf` code

在這份 per-IP exact-match 報告之後，又進一步對齊了：

- `5G_Infrastructure/.agent/tmp/0512-19/upf-ees.log`
- `5G_Infrastructure/.agent/tmp/0512-19/upf-ees2.log`
- `5G_Infrastructure/go-upf-ess/go-upf/internal/ees/*.go`

目的是釐清問題到底在：

1. pseudo-driver 時間對齊錯誤
2. notify 傳送錯誤
3. 還是 EES 端本身把 pseudo replay 和 live kernel URR 混在一起

### Pseudo-driver 本身看起來是對齊的

先看兩台 UPF 的 pseudo-driver 啟動流程。

`UPF-EES / group1`

- `upf-ees.log` 顯示三個 subscription 建立後，都有啟動 warm-start replay stream
- 都正確讀到 `breakingTimeSec=900`
- 第一個 live URR 到達後，有正確 broadcast `first URR signal`
- 後續也有 `gridAnchor` 對齊與 phase2 push

`UPF-EES2 / group2`

- `upf-ees2.log` 也出現相同流程
- 一樣正確讀到 `breakingTimeSec=900`
- 一樣等到第一個 live URR 才對齊 phase2

這表示：

- pseudo-driver 沒有忽略 `file.json` 的 `breaking time`
- phase1/phase2 切換沒有明顯跑錯時間
- 不是單純「播錯資料」或「anchor 沒對齊」

### Notify 本身也沒有失敗

兩份 `upf-ees` log 都持續出現：

- `ees notify success`

目標 URI 也一致是：

- `http://192.168.127.5:8080/collector/upf-notify`

因此：

- 不是 pseudo-driver 有資料但沒送出去
- 也不是 NWDAF 完全沒收到 notify

問題比較像是：

- **送出去的內容本身，就已經不是純 `pre_data` replay**

### 關鍵原因：這個 branch 會把 pseudo phase2 和 live kernel URR 累加

目前 `go-upf` 使用的 branch 是：

- `test-EES-with-pseudodriver`

其 HEAD commit `95dc04f` 的說明已經直接寫出：

- `Replaced Priority Overwrite with Data Accumulation (+=) to fuse Kernel and Pseudo-driver reports.`

也就是說，這個 branch 的設計目標本來就不是：

- 在 pseudo replay 和 kernel live report 之間擇一

而是：

- **把兩者融合**

對應到 `internal/ees/aggregator.go`，最關鍵的是 `consolidateWithPriority()`：

```go
if true {
    existing.ULBytesDelta += r.ULBytesDelta
    existing.DLBytesDelta += r.DLBytesDelta
    existing.ULPacketsDelta += r.ULPacketsDelta
    existing.DLPacketsDelta += r.DLPacketsDelta
}
```

這段代表：

- 只要同一個 `(ueIp, startTime)` window 同時出現多筆 measure
- 不管來源是 `SourceKernel` 或 `SourcePseudo`
- 目前都會**直接累加**

而不是：

- kernel 覆蓋 pseudo
- 或 pseudo 覆蓋 kernel
- 或只保留單一來源

### Log 也支持「pseudo + kernel 同時存在」

在兩份 `upf-ees` log 裡，都可以看到這個模式：

1. pseudo-driver 已開始 phase2 push
2. 幾乎同時仍然持續收到 `EES Handler received report from dispatcher`
3. 之後 aggregator tick 會送出 notify

也就是：

- pseudo-driver phase2 並不是在「完全隔離」的狀態下單獨送資料
- live kernel URR 仍然持續進入同一個 `reportBuffer`

而 `PushReport()` 會把 kernel URR2 report 轉成 `UsageMeasures` 放進 `reportBuffer`；
`PushLiveMeasures()` 也會把 pseudo phase2 window 放進同一個 `reportBuffer`。

最後 `TickOnce()` 在同一輪 tick 內經過 `consolidateWithPriority()`，就把它們加總成單一 slot 再送出去。

### 為什麼只有部分 IP 完全正常

如果問題是 pseudo-driver anchor 完全錯掉，理論上應該所有 IP 都一起壞。

但這次實際看到的是：

- `10.10.0.1`：`218/218` 全程 exact match
- `10.100.0.1`：`218/218` 全程 exact match
- 其餘 4 個 IP 則有不同程度偏差

這種現象更符合下面這種情況：

- pseudo replay 本身是正確的
- 但某些 IP 在實際 kernel live traffic 上有非零貢獻
- 這些 live bytes 又被加到 pseudo replay slot 上
- 因而造成只有部分 IP 偏掉

換句話說：

- `10.10.0.1` / `10.100.0.1` 很可能在這次 run 中 live kernel contribution 幾乎是 0，所以仍然 exact
- `10.10.0.2` / `10.10.0.3` / `10.100.0.2` / `10.100.0.3` 則有不同程度的 live contribution，被和 pseudo replay 累加

這也能解釋：

- `10.100.0.2` 為什麼早段只是小幅偏移
- `10.10.0.3` / `10.100.0.3` 為什麼會出現數十倍到數百倍的差距

### Current root-cause assessment

基於目前的 code 與 log，比較合理的根因判斷是：

- **pseudo-driver phase1=900s 對齊本身沒有明顯錯**
- **notify 路徑也沒有壞**
- **真正的問題是在這個 branch 的 EES aggregator 設計：同一 slot 會把 pseudo replay 和 live kernel URR 累加**

因此這次 `0512-19` 的 per-IP mismatch，不應優先解讀成：

- NWDAF collector 解析錯誤
- pseudo-driver 讀錯 `pre_data`
- 或 `file.json` phase1 沒對齊

而應優先解讀成：

- **`test-EES-with-pseudodriver` branch 的「data fusion」行為，破壞了「UPF notify 必須逐 slot exact 等於 `pre_data`」這個驗證前提**

### Implication for future validation

若後續目標仍然是驗證：

- `UPF notify` 是否能逐 slot exact 重現 `pre_data`

那需要先明確二選一：

1. pseudo-only validation mode  
   phase2 期間不要把 live kernel report 加進來

2. fused mode acceptance  
   接受 notify 不是 exact replay，而是 pseudo + live 混合結果

在目前這個 branch 下，實際行為屬於第 2 種，而不是第 1 種。
- `10.10.0.1` 和 `10.100.0.1` 則提供了很強的反例：testbed 是有能力把 `pre_data` 逐欄位原樣送出的

## Implication for the missing retrain trigger

對這次 `0512-19` 沒有 retrain 的解讀，這份 per-IP 檢查提供了一個更精確的背景：

- 並不是整個 testbed 流量都和 `pre_data` 脫鉤
- 而是 **部分 IP 完全正常、部分 IP 明顯異常**
- 這會讓 group-level aggregate 形成一種混合態
- NWDAF 的 prediction / monitor baseline 因此不一定會呈現 replay `exp46` 那種乾淨的 CAT 切換型態

以 `CAT1 -> CAT2` 的觀察來說，最有意義的不是後段 `tail`，而是：

- `10.10.0.1` / `10.100.0.1` 完全對齊
- `10.10.0.2` / `10.10.0.3` / `10.100.0.3` 出現不同程度的 per-IP 錯位
- `10.100.0.2` 處於「有時接近、有時不 exact」的中間狀態

這足以解釋為什麼這次 live aggregate 的 baseline 與 replay `exp46` 不會完全同型。

## Conclusion

這次 full-IP exact check 的結論是：

1. `0512-19` 並不是所有 IP 都有問題。
2. `10.10.0.1` 與 `10.100.0.1` 在全部 218 個 slots 上，`ul/dl/total` 都和 `pre_data` **完全相等**。
3. 因此「正常部分是否真的逐數字對齊」的答案是：**有，確實存在完全逐數字對齊的 IP。**
4. 但 `10.10.0.3`、`10.100.0.3` 仍然有非常明顯的 per-IP 錯位／爆量問題。
5. `10.10.0.2` 這次也不能視為正常。
6. `10.100.0.2` 雖然在 `CAT1 phase2` 有不少接近值，但仍未達到 exact-match 要求，`cat3_phase2` 也已經完全失真。

如果後續要繼續追 root cause，最值得優先盯的是：

- `10.10.0.3`
- `10.100.0.3`
- `10.10.0.2`

而不是已經證明完全對齊的 `10.10.0.1` / `10.100.0.1`。
