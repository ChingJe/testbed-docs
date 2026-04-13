# UPF Double Notification Investigation Report

**Date:** 2026-03-18
**Component:** UPF-EES (go-upf-ess) ↔ NWDAF
**Symptom:** UPF-EES 的每個 5 秒週期，NWDAF 都會收到包含 2 個 items 的 UPF notification，兩個 item 的 startTime 相差 5 秒：item1 來自 Pseudo Driver 的 Parquet replay，item2 來自 Kernel URR 的即時量測。item2 在 78% 的情況下為 zero traffic，導致 AnLF 大量出現 dedup collision warning。

---

## 實驗設置

| 元件 | 說明 |
|------|------|
| UPF-EES | 負責 UE1 的資料面，UE IP：`10.10.0.1` |
| UPF-EES2 | 負責 UE2 的資料面，UE IP：`10.100.0.1` |
| UE1 | 透過 gNB1 連至 UPF-EES，持續執行 `ping -I uesimtun0 8.8.8.8` |
| UE2 | 透過 gNB2 連至 UPF-EES2，持續執行 `ping -I uesimtun0 8.8.8.8` |
| NWDAF | 部署於 5GC VM，同時訂閱 UPF-EES（corr-1）與 UPF-EES2（corr-2）的 URR 通知 |
| Pseudo Driver | UPF-EES 啟用 Parquet warm-start 功能。訂閱建立時先 burst 推送所有歷史 windows（Phase 1），之後進入 Phase 2：每個 EES tick 推一筆 Parquet replay window，與 Kernel URR 並行進入同一 buffer |

每個 UE 僅以 ICMP ping 產生即時流量（預設每秒一個 packet，即每 5 秒 window 約 ~1984 bytes），但 EES 的通知同時含有 Parquet replay 的較大流量資料。

---

## Observation

NWDAF log 顯示同一個 notification（`items: 2`）內有兩筆相同 IP 但不同 startTime 的 volume report：

```
time="2026-03-18T01:09:25.062004062Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:09:25.062049953Z" level="info" msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-03-18T01:09:15Z, total=9367, ul=3745, dl=5622, totalPkts=49, ulPkts=22, dlPkts=27" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:09:25.063741868Z" level="info" msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-03-18T01:09:20Z, total=0, ul=0, dl=0, totalPkts=0, ulPkts=0, dlPkts=0" CAT="Proc" NF="NWDAF"
```

兩個 items 的 IP 相同（`10.10.0.1` 是 UE 的 IPv4 地址），StartTime 差 5 秒：item1 對應 Parquet replay window，item2 對應 Kernel URR 即時量測 window。

---

## 白話說明

### NWDAF 觀察到的現象

從 NWDAF 的角度，UPF-EES（corr-1）**每個週期**都收到 items:2 的 notification，裡面帶著同一個 UE（IP）的兩筆資料，兩筆的 startTime 相差剛好 5 秒。item1 為 Parquet replay 的較大流量（數萬到數十萬 bytes），item2 在 55 筆中有 43 筆（78%）為 zero，其餘 12 筆為小流量（~1984 bytes / 31 packets 的 ICMP ping）。

問題不在兩筆各自的內容，而在 **item2 的 startTime 和下一個 notification 的 item1 startTime 完全相同**：前一個 notification 把 T+5 window 作為 Kernel URR（item2）報出，緊接著下一個 notification 又把同一個 T+5 window 作為 Parquet replay（item1）報出，造成 RawUpfData 中該 startTime 出現兩次。這個重複是**結構性的**，由 Phase 2 Parquet 與 Kernel URR 的時間網格 off-by-one 決定，在整個 Phase 2 期間每個 cycle 都發生。

RawUpfData 的陣列順序固定（notifications 依時序追加）：item2(N) 先寫入、item1(N+1) 後寫入。AnLF last-wins 因此每次都覆蓋 item2(N)（Kernel zero 或小流量）、保留 item1(N+1)（Parquet 流量）。就資料保存而言結果是確定且正確的，但每次 AnLF 執行都會對歷史資料中**每一個** slot 觸發一次 collision warning。

### 為什麼 UPF 會一次送兩筆（補充）

UPF-EES 啟用了 **Pseudo Driver**（Parquet warm-start），會在訂閱後持續以 Parquet 預錄流量資料補充 EES buffer（Phase 2）。每個 EES tick 時，buffer 內同時有兩個來源的資料：

- **Phase 2 Pseudo Driver**：推送 Parquet window N（startTime=T），流量來自預錄資料
- **Kernel URR**：推送當前量測 window N+1（EndTime snap 後 startTime=T+5），流量來自即時 ping

兩者的 IP 相同（`10.10.0.1`），但 startTime 相差 5 秒（off-by-one）。`consolidateReports()` 以 `(IP + StartTime)` 為 key 合併——key 不同，保留為兩筆獨立 item → items:2。

```
每個 EES tick 時的 reportBuffer 內容：
  Phase2 Parquet[N]  ip=10.10.0.1, startTime=T,   bytes=237K  ← Parquet replay
  Kernel URR[N+1]    ip=10.10.0.1, startTime=T+5, bytes=0~1984 ← 即時 ping

consolidateReports key:
  "10.10.0.1-T"    → item1（Parquet 流量）
  "10.10.0.1-(T+5)" → item2（ping 或 zero）

→ Notification: items=2
```

此相位差是**結構性且固定的**：Pseudo Driver 的 anchorTime（整秒 grid 基準）比 Kernel URR 的 snap grid 早一個 window，因此每個 tick 的兩筆 startTime 永遠差 5 秒，不會被合併。

---

## Part 1：UPF 端（go-upf-ess EES Aggregator）

### 根本原因

UPF-EES 啟用了 Pseudo Driver（Parquet warm-start），在訂閱成立後進入 **Phase 2**：每個 EES tick 推一筆 Parquet replay window，與 Kernel URR **並行**進入 `reportBuffer`，造成每個 tick 有兩個不同 startTime 的資料同時存在。

流程：

1. **Phase 2 Pseudo Driver**：每個 EES tick 前，`PushLiveMeasures()` 推送 Parquet window N（startTime=T）
2. **Kernel**：每 5 秒觸發一次 `TYPE_PERIO_TIMEOUT`，`PushReport()` 推送量測 window N+1（EndTime snap 後 startTime=T+5）
3. **TickOnce()**：drain buffer，對所有 measures 先 snap StartTime，再呼叫 `consolidateReports()`
4. `consolidateReports()` 以 `(IP + SnappedStartTime)` 為 key 合併——兩筆 startTime 分別是 T 和 T+5，key 不同，保留為 2 items → `items: 2`

**startTime 差 5 秒的根本原因**（off-by-one）：Pseudo Driver 的時間網格（`anchorTime`）以第一個 TickOnce 為基準，replay window N 的 startTime = anchorTime + (N-startWIdx)*5s。Kernel URR 的 EndTime snap 後落在下一個 5 秒邊界，startTime = T+5。兩者永遠差一個 window，consolidateReports 無法合並。

### UPF Log 證據

Kernel 每 5 秒只觸發一次 periodic timeout（不是兩次），items:2 的兩筆**不是**兩筆 Kernel URR，而是 Phase 2 Parquet + Kernel URR：

```
time="2026-03-18T01:10:08.719483899Z" level="info" msg="recv event[TYPE_PERIO_TIMEOUT][{eType:3 lSeid:0 urrid:0 period:5000000000}]" CAT="Perio" NF="UPF"
time="2026-03-18T01:10:13.720218375Z" level="info" msg="recv event[TYPE_PERIO_TIMEOUT][{eType:3 lSeid:0 urrid:0 period:5000000000}]" CAT="Perio" NF="UPF"
```

### 為什麼只有 UPF-EES 比較明顯？

UPF-EES 的 Parquet 資料 UE IP（`10.10.0.1`）與真實 UE 相同，Phase 2 replay 資料會匹配到訂閱並進入 buffer，導致**每個 cycle 都是 items:2**。UPF-EES2 的情況不同（Phase 2 replay 與真實 UE 的 IP/流量對齊程度有差異），items:2 和 items:1 交替出現，collision 頻率較低。

---

## Part 2：NWDAF 端（AnLF dedup collision）

### Dedup 邏輯

NWDAF `upf_notify.go` 將每個 item 的 `startTime` 存為 `Timestamp` 寫入 `RawUpfData`。

AnLF 在 `alignAndZipInMemory` 中，以每個 IP 的第一筆資料時間為 anchor，把 `RawUpfData` 裡所有資料點 snap 到 `anchor + n*si`（si=5s）的 slot center。若兩筆資料 snap 到同一個 center，後出現在陣列中的覆蓋前面的（last-wins）：

```go
for _, dp := range td.RawUpfData {
    n := round((dp.Timestamp - anchor) / si)
    centerUnix := anchor + n*si
    perIP[centerUnix] = dp   // 後來的覆蓋前面的
}
```

### Dedup Collision Log 證據

```
time="2026-03-18T01:12:23.726372824Z" level="warning" msg="dedup collision ip=10.10.0.1: t=1773796270 and previous both snap to center=1773796270 (anchor=1773796210 si=5), keeping later" CAT="AnLF" NF="NWDAF"
time="2026-03-18T01:12:23.726409187Z" level="warning" msg="dedup collision ip=10.10.0.1: t=1773796275 and previous both snap to center=1773796275 (anchor=1773796210 si=5), keeping later" CAT="AnLF" NF="NWDAF"
time="2026-03-18T01:12:23.725673992Z" level="warning" msg="dedup collision ip=10.100.0.1: t=1773796131 and previous both snap to center=1773796131 (anchor=1773796126 si=5), keeping later" CAT="AnLF" NF="NWDAF"
```

### 影響分析

Collision 發生在相鄰兩個 notification 之間：Notification N 的 item2 與 Notification N+1 的 item1 擁有**完全相同的 startTime**（不是 snapping 誤差，而是精確重複），兩者先後存入 RawUpfData，AnLF 處理時發生碰撞。

```
Notification N   通知：  [ T+0: Parquet ~23K ]  [ T+5: Kernel 0~1984 ]   ← item2.startTime = T+5
Notification N+1 通知：  [ T+5: Parquet ~237K]  [ T+10: Kernel 0~1984]   ← item1.startTime = T+5（相同！）
                                                       ↕
                                          RawUpfData 中 startTime=T+5 出現兩次
                                          → AnLF snap 到同一 center=T+5 → collision

Slot grid：  |──── T+0 ────|──── T+5 ────|──── T+10 ────|
                                 ↑
                            碰撞發生在此
```

Notification N 的 item2（Kernel URR，startTime=T+5）與 Notification N+1 的 item1（Phase 2 Parquet，startTime=T+5）的 startTime **完全相同**，先後存入 RawUpfData，AnLF 處理時發生碰撞。

RawUpfData 的陣列追加順序固定（依 notification 到達時間）：item2(N) 先寫入，item1(N+1) 後寫入。AnLF last-wins 因此**每次都保留 item1(N+1)（Parquet 流量），丟棄 item2(N)（Kernel zero 或小流量）**。就資料保留而言結果是正確的——有流量的那筆被保留。但每次 AnLF 執行都在歷史資料的**每一個 slot** 觸發 collision warning，造成大量無意義的 warning。

---

## 流程總結

```
Phase 2 Pseudo Driver (每個 EES tick 前)
  └─ PushLiveMeasures() ──push──▶ EES reportBuffer
      └─ Parquet window N: ip=10.10.0.1, startTime=T, bytes=~237K

Kernel (every 5s)
  └─ TYPE_PERIO_TIMEOUT → PushReport() ──push──▶ EES reportBuffer
      └─ URR window N+1: ip=10.10.0.1, startTime=T+5, bytes=0~1984

EES Aggregator TickOnce()
  ├─ snap StartTime（以 GridAnchor 為基準，整 5 秒邊界）
  ├─ consolidateReports: (10.10.0.1, T) ≠ (10.10.0.1, T+5) → 保留為 2 items
  └─ Notify NWDAF: items=2
      ├─ item1: startTime=T,   bytes=~237K  ← Parquet replay
      └─ item2: startTime=T+5, bytes=0~1984 ← Kernel URR（78% zero）

      ※ 下一個 notification 的 item1.startTime = T+5（與本次 item2 相同）

NWDAF AnLF alignAndZipInMemory
  └─ startTime=T+5 在 RawUpfData 中出現兩次
      ├─ item2(N)：Kernel zero 或小流量（先寫入）
      └─ item1(N+1)：Parquet 流量（後寫入）
      → last-wins 保留 Parquet 流量，丟棄 Kernel zero 或小流量
      ⚠ 資料保留結果正確，但每個 slot 都觸發 collision warning
```

---

## 修法方向

| 方向 | 位置 | 做法 |
|------|------|------|
| **修 NWDAF dedup** | `anlf/analytics.go` | dedup 改 keep-max（保留 volume 較大的那筆）取代 last-wins，確保有流量的資料不會被 zero 覆蓋；同時消除 warning 噪音 |
| **修 EES Phase 2 對齊** | `ees/pseudodriver.go` | 修正 Phase 2 replay window 的 startTime 計算，使其比現在提前一個 period，對齊 Kernel URR snap 後的 startTime（off-by-one 修正），讓兩筆能被 consolidateReports 合併為 items:1 |
| **接受現狀** | — | Phase 2 播完後問題自然消失（items 回歸 1），影響時間有限；若 warning 不影響功能，可僅記錄文件 |

---

## 附錄：實際 Log 證據

### A-1：UPF-EES Kernel Periodic Timeout

每 5 秒觸發一次 `TYPE_PERIO_TIMEOUT`，確認 Kernel 每個 tick 只產生一筆 URR。items:2 的另一筆來源是 Phase 2 Pseudo Driver（Parquet replay），而非 Kernel 重複發送。

```
time="2026-03-18T01:12:08.719774636Z" level="info" msg="recv event[TYPE_PERIO_TIMEOUT][{eType:3 lSeid:0 urrid:0 period:5000000000}]" CAT="Perio" NF="UPF"
time="2026-03-18T01:12:13.719637196Z" level="info" msg="recv event[TYPE_PERIO_TIMEOUT][{eType:3 lSeid:0 urrid:0 period:5000000000}]" CAT="Perio" NF="UPF"
time="2026-03-18T01:12:18.719426793Z" level="info" msg="recv event[TYPE_PERIO_TIMEOUT][{eType:3 lSeid:0 urrid:0 period:5000000000}]" CAT="Perio" NF="UPF"
time="2026-03-18T01:12:23.720314875Z" level="info" msg="recv event[TYPE_PERIO_TIMEOUT][{eType:3 lSeid:0 urrid:0 period:5000000000}]" CAT="Perio" NF="UPF"
```

每兩筆間隔約 5 秒，與 PFCP URR 設定的 `period: 5000000000 ns` 一致。

---

### A-2：NWDAF 收到 items:2 的頻率

以下為 01:11:35 ～ 01:12:20 間的 notification 紀錄，corr-1（UPF-EES, 10.10.0.1）**每個 cycle 皆為 items:2**，從未出現 items:1：

```
time="2026-03-18T01:11:35.062361751Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:11:40.059875806Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:11:45.060988159Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:11:50.062328915Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:11:55.059429996Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:00.059676589Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:05.061407700Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:10.066289568Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:15.059517330Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:20.060222726Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
```

相比之下，corr-2（UPF-EES2, 10.100.0.1）在同一期間交替出現 items:1 和 items:2，反映 UPF-EES2 的 Phase 2 Parquet replay 與真實 UE IP/流量的對齊情況不同，並非每個 cycle 都能產生兩筆同時入 buffer 的情況。

---

### A-3：相鄰兩個 Notification 的 startTime 重疊（直接證據）

以下兩個相鄰 notification（間距 5 秒）清楚呈現 **item2 的 startTime 與下一個 notification 的 item1 startTime 相同**：

**Notification @ 01:12:15（items: 2）：**
```
time="2026-03-18T01:12:15.059517330Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:15.059654471Z" level="info" msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-03-18T01:12:05Z, total=23005, ul=8085, dl=14920, totalPkts=87, ulPkts=42, dlPkts=45" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:15.062557035Z" level="info" msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-03-18T01:12:10Z, total=0, ul=0, dl=0, totalPkts=0, ulPkts=0, dlPkts=0" CAT="Proc" NF="NWDAF"
```

**Notification @ 01:12:20（items: 2）：**
```
time="2026-03-18T01:12:20.060222726Z" level="info" msg="Processing UPF notification, items: 2, correlationId: corr-1" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:20.060269857Z" level="info" msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-03-18T01:12:10Z, total=237290, ul=23869, dl=213421, totalPkts=431, ulPkts=179, dlPkts=252" CAT="Proc" NF="NWDAF"
time="2026-03-18T01:12:20.061828810Z" level="info" msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-03-18T01:12:15Z, total=1984, ul=1024, dl=960, totalPkts=31, ulPkts=16, dlPkts=15" CAT="Proc" NF="NWDAF"
```

`startTime=2026-03-18T01:12:10Z` 出現兩次：
- 第一次（item2 of 01:12:15）：zero bytes，來自 **Kernel URR** 當前量測 window
- 第二次（item1 of 01:12:20）：237,290 bytes，來自 **Phase 2 Parquet replay** 下一個 window

注意 item1 的 237,290 bytes 遠大於 ping 流量（~1984 bytes），確認為 Parquet 預錄資料而非即時 ping。

兩者依序追加進 RawUpfData，AnLF last-wins 保留 Parquet 流量（237K）、丟棄 Kernel zero——資料正確性沒有問題，但每個 slot 都觸發 collision warning。

由於 UPF-EES 的 Phase 2 Parquet replay 持續進行，此 overlap 模式在**整個 Phase 2 期間每個 cycle 皆發生**（本次實測 55 個 notifications 全為 items:2），Phase 2 播完後自然消失。

---

### A-4：AnLF Dedup Collision（系統性連續警告）

AnLF 每次執行 `alignAndZipInMemory` 時，因 RawUpfData 中存在大量重複 startTime，導致**連續多個 slot 皆出現 collision warning**。以下為 01:12:23 這次分析的部分輸出（10.10.0.1 共觸發 24 個連續 slot 的 collision）：

```
time="2026-03-18T01:12:23.724620797Z" level="warning" msg="dedup collision ip=10.10.0.1: t=1773796215 and previous both snap to center=1773796215 (anchor=1773796210 si=5), keeping later" CAT="AnLF" NF="NWDAF"
time="2026-03-18T01:12:23.724977915Z" level="warning" msg="dedup collision ip=10.10.0.1: t=1773796220 and previous both snap to center=1773796220 (anchor=1773796210 si=5), keeping later" CAT="AnLF" NF="NWDAF"
time="2026-03-18T01:12:23.725177230Z" level="warning" msg="dedup collision ip=10.10.0.1: t=1773796225 and previous both snap to center=1773796225 (anchor=1773796210 si=5), keeping later" CAT="AnLF" NF="NWDAF"
...（共 24 筆，center 從 1773796215 連續到 1773796330，間距 5 秒）
time="2026-03-18T01:12:23.729219529Z" level="warning" msg="dedup collision ip=10.10.0.1: t=1773796330 and previous both snap to center=1773796330 (anchor=1773796210 si=5), keeping later" CAT="AnLF" NF="NWDAF"
```

10.100.0.1（UPF-EES2）在同一次分析中也出現 collision，但數量較少（5 筆，間距 15 秒），反映 UPF-EES2 的 Phase 2 Parquet replay 並非每個 cycle 都與 Kernel URR 同時存在於 buffer，items:1 和 items:2 交替出現，collision 頻率遠低於 UPF-EES。
