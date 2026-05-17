# NWDAF Prediction Time-Axis Mismatch Issue

**Date:** 2026-05-14  
**Component:** UPF-EES (`go-upf-ess`) ↔ NWDAF (`AnLF`, notifier, accuracy monitor)  
**Type:** issue report / problem statement

---

## 問題摘要

目前 `UPF-EES -> NWDAF -> consumer` 這條線上，同時存在兩套不同的時間體系：

1. **measurement-slot 時間軸**
   - 以 UPF 回報的 `startTime` 為主
   - 代表某筆流量量測所屬的 measurement window start
   - NWDAF 用這條時間軸建立歷史流量序列

2. **wall-clock / callback 時間軸**
   - 以 `now` / `snappedNow` 為主
   - 代表 NWDAF 進行推論或發送 callback 的當下時間
   - NWDAF 目前用這條時間軸產生 `TargetTime`，也把 callback 內的 `ueComm.ts` 設成 `now`

這兩套時間軸在資料幾乎即時送達時，可能剛好沒有明顯衝突；但一旦 `latest aggregated slot` 和 `now` 之間存在可觀 gap，就會出現語意錯位：

- 模型實際上是根據 `startTime` 體系下的歷史 slot 做預測
- callback 卻把結果標到 `now`
- accuracy monitor 又把 prediction 標到 `snappedNow`
- actual ground truth 則仍以 `startTime` 為主

因此目前同一筆 prediction，至少同時存在三個不同的「它所屬時間」：

- `latest aggregated slot ts`
- `TargetTime`
- callback `ueComm.ts`

這不是單純的 logging 差異，而是會直接影響：

- consumer 對 prediction horizon 的解讀
- accuracy monitor 的 prediction/actual 配對
- metric 是否是在比對正確的 future slot

---

## 直接觀察到的現象

在 `5G_Infrastructure/.agent/tmp/0513-13/nwdaf.log` 中，可以看到：

```text
time="2026-05-13T03:42:06.338233410Z" ... msg="latest aggregated slot: ... ts=2026-05-13T03:40:40Z ulVol=10588213 dlVol=6917466 ..."
time="2026-05-13T03:42:06.354342728Z" ... msg="ML inference: ... steps=1 ulVol=4351244 dlVol=2928733 confidence=80 commDur=30s"
```

同一份 log 稍早也能找到對應的 UPF actual：

```text
time="2026-05-13T03:41:43.695851333Z" ... msg="UPF VOLUME: ip=10.10.0.1, startTime=2026-05-13T03:40:40Z ..."
time="2026-05-13T03:41:43.703467226Z" ... msg="UPF VOLUME: ip=10.10.0.2, startTime=2026-05-13T03:40:40Z ..."
time="2026-05-13T03:41:43.710125146Z" ... msg="UPF VOLUME: ip=10.10.0.3, startTime=2026-05-13T03:40:40Z ..."
```

這表示：

- `latest aggregated slot ts=2026-05-13T03:40:40Z` 對應的是 `startTime=03:40:40Z` 的 measurement slot
- 但 NWDAF 寫出 inference log 的當下已經是 `03:42:06Z`

若模型語意是「用歷史最後一格去預測下一格」，那這筆 prediction 更自然的 target 應接近：

- `03:41:10Z`

而不是：

- callback 發送時間 `03:42:06Z`
- 或 `snappedNow=03:42:00Z`

這就是本 issue 的核心錯位。

---

## 目前實作中的兩套時間體系

## 1. measurement-slot 時間軸：`startTime`

UPF-EES notify payload 同時帶有 `timeStamp` 與 `startTime`，但 NWDAF 目前處理 actual 流量資料時，明確以 `startTime` 為 measurement timestamp：

- `internal/sbi/processor/upf_notify.go`
- `measurementTs := item.StartTime`
- 只有 `startTime` 缺失時才 fallback 到 `item.TimeStamp`

接著這個 `measurementTs` 會被寫進：

- in-memory `UpfDataPoint.Timestamp`
- MongoDB `UpfTrafficRecord.Timestamp`

因此：

- NWDAF 用於建歷史流量序列的時間
- accuracy monitor 用於 ground truth lookup 的時間

本質上都是 `startTime` 體系。

---

## 2. wall-clock 時間軸：`now` / `snappedNow`

NWDAF 做推論時，會先取當下時間 `now`，再算：

- `snappedNow = floor(now / samplingInterval) * samplingInterval`

然後：

- `TargetTime = predictionTargetTime(snappedNow, samplingInterval, step)`
- step 0 對應 `snappedNow`

這表示 accuracy monitor 內部存下來的 prediction time，不是由「最後一個歷史 slot」往後推，而是由「目前 wall-clock snapped 到最近邊界」往後推。

此外，回傳給 consumer 的 `models.UeCommunication` 目前也直接使用：

- `Ts: &now`

也就是 callback 內的 `ueComm.ts` 代表的是 callback 當下時間，而不是模型 target slot 的時間。

---

## prediction 與 actual 目前如何被對照

## prediction 在 accmonitor 中如何存

`PredictionRecord` 目前有兩個時間欄位：

- `PredictedAt`: 真正做 prediction 的時間
- `TargetTime`: accmonitor 認定這筆 prediction 所屬的時間

目前寫入方式是：

- `PredictedAt = now`
- `TargetTime = snappedNow + step * samplingInterval`

因此 `TargetTime` 本質上也是 wall-clock 體系的產物。

---

## actual 在 accmonitor 中如何取

accuracy monitor 查 actual 時，會用：

- `pred.TargetTime ± samplingInterval`

作為搜尋窗，然後在每個 corrId 的資料中挑：

- 最接近 `TargetTime` 的 actual record

而 actual record 的 timestamp 本質上是：

- `startTime`

所以配對規則實際上是：

- prediction 用 `TargetTime`
- actual 用 `startTime`
- 再做最近鄰配對

---

## 這樣會造成什麼問題

如果 `latest aggregated slot` 幾乎貼著 `snappedNow`，這套設計可能勉強可用。

但只要中間有 gap，例如：

- 歷史最後一格：`03:40:40`
- 自然下一格：`03:41:10`
- `snappedNow`: `03:42:00`

那麼 accmonitor 目前會拿：

- prediction target = `03:42:00`

去配對：

- 最接近 `03:42:00` 的 `startTime` actual

而不是去配對：

- `03:41:10`

這會導致幾種情況：

1. **配到較晚的一格 actual**
   - prediction 原本想回答的是 `latest historical slot` 的下一格
   - metric 卻拿去和更晚的 slot 比

2. **gap 大於 1 slot 時高機率錯配**
   - 即使 `±samplingInterval` 內還找得到 actual，也未必是模型真正對應的那格

3. **gap 更大時直接 unmatched**
   - accmonitor 可能找不到任何 actual，導致樣本被丟掉

4. **consumer 看到的時間也可能誤導**
   - callback `ueComm.ts` 目前等於 `now`
   - `commDur=30s` 很容易被解讀成「從現在開始 30 秒的預測」
   - 但模型實際上未必是在回答這個時間窗

---

## 目前可以成立的中間結論

1. NWDAF 目前確實同時使用了兩套時間體系：
   - `startTime` 體系
   - `now` / `snappedNow` 體系

2. prediction 的歷史輸入資料是依 `startTime` 體系建立的。

3. callback `ueComm.ts` 目前是 `now`，不是 prediction target slot 時間。

4. accmonitor 存下來的 prediction time 目前是 `TargetTime=snappedNow+step*si`，不是「歷史最後一格的下一格」。

5. actual ground truth 則仍以 `startTime` 為主。

6. 因此一旦 `latest aggregated slot` 和 `snappedNow` 之間出現 gap，prediction/actual pair 就可能錯位，導致 metric 對到錯答案。

---

## 這份 issue 想先釐清的不是修法，而是問題定義

在開始改 code 前，至少需要先對齊下面三件事：

1. **prediction 的語意到底是什麼**
   - 是「最後一個歷史 slot 的下一格」
   - 還是「從當下 `snappedNow` 開始的一格」

2. **對 consumer 回報的時間欄位應該代表什麼**
   - prediction target slot
   - report generation time
   - callback 發送時間

3. **accmonitor 應該拿哪個時間去做配對**
   - `latest historical slot + 1*si`
   - `resp.PredictedData[i].ts`
   - `TargetTime=snappedNow+step*si`

只要這三件事沒有統一，後續的：

- callback 語意
- metric 合法性
- retrain trigger 的可信度

都會持續受影響。

---

## 一句話總結

目前 NWDAF 的 prediction 流程不是單純「時間欄位命名不清」而已，而是：

- **歷史輸入、prediction target、callback payload、accuracy monitor ground truth lookup，分別混用了不同時間語意。**

只要 `startTime` 體系和 `now/snappedNow` 體系沒有被統一，prediction 對 consumer 的意義，以及 accmonitor 算出的 metric，都可能不是在回答同一個時間問題。
