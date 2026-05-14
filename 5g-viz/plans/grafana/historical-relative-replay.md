# Grafana Historical Relative Replay Test Plan

本文件用來驗證一個簡化方向：在 replay mode 中，完全捨棄 pseudo-live / `MetricPlayer` / replay 倍速，改為直接使用原始 session 的 backfill 資料，並透過 Grafana 的 historical relative time range 來達成「播放時圖表平滑滑動」。

> 狀態：測試計畫，尚未實作。

---

## 背景

目前 replay `PLAYING` 的 Grafana 體驗是靠 pseudo-live pipeline 達成：

- backend 將歷史 metric remap 到現在時間
- 每次 Play 建立新的 `_live_...` pseudo session
- 前端 iframe 切到 `from=now-<window>&to=now`

這條路徑雖然能避免播放期間反覆 reload iframe，但代價是：

- backend 複雜度高
- 與 paused backfill 有一致性落差
- 需要額外的 `/api/replay/*`、cleanup、pre-seed、remote write emit loop
- replay speed、chart window、pseudo session lifecycle 全部耦合在一起

本次要驗證的替代方案是：

- replay 只保留原始 session backfill
- 播放時不再建立 pseudo session
- Grafana 直接查原始 `session=<orig_session>`
- 若 playhead 對應到「距今 `offset`」的位置，則 iframe 使用概念上等價於下列語意的相對時間窗：

```text
var-session=<orig_session>
from=now-(offset + window)
to=now-offset
refresh=5s
```

實作上不應直接依賴 `now-(offset + window)` 這種概念式字串，而是前端先把它換算成 Grafana 實際可接受的 relative time string。

若這種 historical relative window 在 Grafana 中能穩定 auto-refresh，便可能用更簡單的方式取代 pseudo-live。

---

## 為何放在 `plans/grafana`

這份工作雖然會影響 DVR 設計，但眼前要先驗證的核心問題不是 DVR 狀態機，而是：

1. Grafana 是否接受這種 relative `from/to`
2. Grafana 是否會在不重設 iframe 的情況下平滑推進視窗
3. Prometheus retention / session availability 是否足以支撐這條路徑

因此這份文件先放在 `plans/grafana/`，視為一個圖表時間窗與資料保留策略驗證。若驗證通過，再回頭收斂到 DVR 實作規劃。

---

## 目標

驗證以下假設是否成立：

1. Grafana iframe 可使用 `from=now-...&to=now-...` 查詢過去某段固定歷史視窗，而非只能 `to=now`
2. 在 `refresh=5s` 下，historical relative window 會隨真實時間平滑前進，不需重新設定 iframe `src`
3. 在 replay `1x` 播放語意下，這種平滑前進可取代 pseudo-live 的主要使用者體感需求
4. 只要原始 session 仍存在於 Prometheus，Grafana 就能直接查到 replay 圖表
5. 若 session 不存在於 Prometheus，但使用者有 session 目錄，則可以先 backfill 再使用此模式
6. 若 session 因 Prometheus retention 被清除，系統可以偵測並明確降級，而不是默默壞掉

---

## 非目標

- 不驗證 replay 倍速是否保留
- 不追求與現行 pseudo-live 完全相同的 API 介面
- 不在本階段處理 live DVR
- 不在本階段處理 `MetricPlayer` 移除後的正式重構
- 不在本階段處理 chart rendering 右邊界缺線等既有 Grafana 問題

---

## 核心假設與限制

### A. 此方案只自然支援 `1x`

historical relative window 的推進速度跟真實時間綁在一起，因此若要完全捨棄 pseudo-live，預期也會一併捨棄 replay 倍速。

### B. 原始 session timestamp 必須仍存在於 Prometheus

此方案不是把歷史資料重新映射到現在，而是直接查原始時間戳。因此只要對應資料已被 retention 清掉，就無法再播放。

### C. replay `Play` 與 `Pause` 將使用不同時間窗語意

- `Play`：historical relative window，固定 URL、依 auto-refresh 前進
- `Pause / Scrub`：absolute trailing window，直接固定在目前 playhead

這代表仍可能保留一次性的 mode switch，但不再需要 pseudo session。

---

## 測試前提

### Session 可用性前提

需要準備至少 3 種 session：

1. **Prometheus 已存在**
   原始 `session=<orig_session>` 已可直接在 Grafana / Prometheus 查到
2. **Prometheus 不存在，但本地有 session 目錄**
   可用 replay backfill 補回 Prometheus
3. **Prometheus 不存在，本地也沒有 session 目錄**
   預期無法播放，需驗證失敗行為

### 時間跨度前提

需要至少覆蓋以下時間距離：

1. 幾分鐘前
2. 幾小時前
3. 超過一天前

目的是驗證 `now-<offset>` 的計算與 Grafana 顯示在短期與跨日情境下都可用。

### Query 前提

需至少驗證下列圖表元素：

1. ground truth 線
2. prediction 線（含 `offset 5s`）
3. deviation / accuracy 類 panel
4. retrain annotations

---

## 驗證問題

### 1. Grafana URL 語法

需確認目前嵌入路徑是否穩定接受：

```text
/d/<uid>?...&var-session=<orig_session>&from=now-<A>&to=now-<B>&refresh=5s
```

其中：

- `A > B`
- `A - B = chart window`
- `B` 代表 playhead 距離現在的 offset

### 2. 視窗是否會自動滑動

若 URL 固定不變，Grafana 是否會在 refresh 時自動把：

```text
from=now-10m&to=now-7m
```

重新解釋為新的真實時間範圍，並讓畫面平滑向右推進。

### 3. 是否仍會 reload iframe

需區分兩件事：

1. **Grafana panel 內部 query refresh**
2. **整個 iframe reload**

本測試的關鍵是驗證：固定 `src` 時，Grafana 是否只做 panel refresh，而不會像現在 absolute `from/to` 那樣每次都重設 iframe。

### 4. `Pause -> Play` 是否可重建相對視窗

暫停後，前端可取得目前 playhead。需驗證 resume 時只要重新計算新的 `offset`，再設定一次新的 historical relative window，是否就足夠。

### 5. Prometheus retention 邊界

需確認：

1. 當 session 還在 retention 內時，一切正常
2. 當 session 已被 retention 清掉時，Prometheus / Grafana 會呈現什麼訊號
3. 系統之後應以哪種方式檢查「session 是否仍可播放」

---

## 測試矩陣

| Case | Session in Prometheus | Local session dir | Playback | 預期 |
|---|---|---:|---:|---|
| A1 | yes | yes/no | play | 直接使用 historical relative window 成功播放 |
| A2 | yes | yes/no | pause/scrub | 切回 absolute trailing window 成功 |
| B1 | no | yes | replay startup | backfill 成功後可播放 |
| B2 | no | yes | replay startup | backfill 失敗時明確顯示 Grafana unavailable |
| C1 | no | no | replay startup | 明確拒絕，無法播放圖表 |
| D1 | evicted by retention | yes/no | replay startup/play | 明確偵測 session 已失效，不進入假播放 |

---

## 詳細測試步驟

### 1. 基礎語法驗證

目標：確認 Grafana 真的接受 historical relative window。

步驟：

1. 選一個已存在於 Prometheus 的舊 session
2. 手動在 iframe 或 browser URL 上設定：

```text
var-session=<orig_session>
from=now-<offset + 3m>
to=now-<offset>
refresh=5s
```

3. 觀察 Grafana 是否成功顯示該段歷史資料
4. 切換不同 offset，涵蓋分鐘、小時、跨日案例

成功標準：

- 能正確顯示對應歷史資料
- panel 不出現 query parse error

---

### 2. 自動滑動驗證

目標：確認固定 historical relative URL 時，Grafana 畫面會自動前進。

步驟：

1. 選定一個 3 分鐘視窗
2. 使用固定 URL：

```text
from=now-<offset + 3m>
to=now-<offset>
refresh=5s
```

3. 保持 iframe `src` 不變，連續觀察至少 30~60 秒
4. 同時用 browser devtools 或 log 觀察是否有 iframe full reload

成功標準：

- 曲線位置會向右推進
- 不需要重設 iframe `src`
- 沒有可感知的整頁 iframe 閃爍

---

### 3. Replay `1x` 體感驗證

目標：確認此模式是否已足以取代 pseudo-live 的主要使用者需求。

步驟：

1. 讓 topology replay 以 `1x` 播放
2. Grafana 固定使用對應 playhead 的 historical relative URL
3. 觀察 topology 與 chart 的主觀同步感
4. 在中途 Pause，再 Resume

成功標準：

- chart 的推進感接近 live
- play 期間不需要週期性重設 iframe
- Pause / Resume 只需在切換當下更新一次 URL

備註：

- 這裡不要求秒級完全同步
- 若結果是「足夠觀察用途」即可視為通過

---

### 4. Pause / Scrub 切換驗證

目標：確認 historical relative window 與現有 absolute trailing window 可以自然切換。

步驟：

1. 在 replay 播放中按 Pause
2. Grafana 改回：

```text
from=<playhead - window>
to=<playhead>
refresh=off
```

3. Scrub 到另一個時間點
4. 再按 Play，重新計算 offset，切回：

```text
from=now-(offset + window)
to=now-offset
refresh=5s
```

成功標準：

- Pause 後圖表停住
- Scrub 後圖表正確顯示該歷史點附近資料
- Resume 後圖表重新開始平滑前進

---

### 5. Query 語意驗證

目標：確認這個方案不會破壞現有 panel query 語意。

檢查項目：

1. `var-session=<orig_session>` 在 historical relative window 下仍正確過濾資料
2. prediction 線的 `offset 5s` 仍對齊
3. retrain annotations 仍能顯示
4. accuracy / deviation panel 不因 relative `to` 非 `now` 而失真

成功標準：

- 至少主要面板都仍可讀
- 沒有出現只在 pseudo-live 才能運作的 query 假設

---

### 6. Retention 驗證

目標：確認此方案對 Prometheus retention 的依賴程度，並定義降級行為。

步驟：

1. 先確認 Prometheus 目前 retention 設定與實際保留範圍
2. 選取一個接近 retention 邊界的 session
3. 驗證 session 在：
   - 還存在時的 replay 行為
   - 被清除後的 replay 行為
4. 驗證是否可用 session existence check 提前檢測

建議檢查方向：

1. 啟動 replay 前先 query `nwdaf_session_anchor{session="<orig_session>"}`
2. 若查無資料，再決定是否觸發 backfill
3. 若沒有 session dir 可 backfill，則直接顯示不可播放

成功標準：

- 能明確知道 session 是「還在」、「可補」、「不可補」哪一種
- 不會出現 replay UI 可以操作，但 Grafana 永遠空白的半失敗狀態

---

### 7. 性能與穩定性觀察

目標：確認這種方案是否真的比 pseudo-live 更簡單且更穩。

觀察項目：

1. backend 是否不再需要 `MetricPlayer`
2. 是否可移除 `/api/replay/play|pause|speed`
3. Prometheus 是否不再需要 pseudo session cleanup
4. replay 啟動流程是否只剩「檢查 session -> 必要時 backfill -> 查原始 session」
5. chart window 改變時是否只需重算 relative `from/to`

成功標準：

- 主要複雜度明顯下降
- 沒有引入同等級的新邏輯分支

---

## 結果判定

### 通過

若符合以下條件，可進入後續重構規劃：

1. historical relative window 可穩定使用
2. `1x` replay 體感足夠接近 live
3. Pause / Scrub / Resume 切換自然
4. session availability / retention 可明確檢查
5. 無需 pseudo-live 才能成立的關鍵 query 行為

### 不通過

若出現以下任一情況，則不建議移除 pseudo-live：

1. Grafana 其實不會平滑前進，仍會產生明顯重載感
2. historical relative window 在實際 auto-refresh 下不穩定
3. panel query、annotation 或 prediction 行為異常
4. retention 使此方案在實務上過於脆弱
5. `1x` 以外雖可放棄，但連 `1x` 都無法達成可接受體驗

---

## 若驗證通過，下一步的重構方向

1. 移除 replay pseudo-live 與 `MetricPlayer`
2. 移除 replay speed 控制
3. 保留 replay 的 session backfill 能力
4. 前端播放中改為 historical relative Grafana window
5. 新增 session existence / retention-aware 檢查
6. 重寫 replay runtime 文件，明確宣告「replay 僅支援 `1x` + Pause/Scrub」

---

## 建議實驗分支

可用獨立分支進行驗證，例如：

```text
experiment/replay-without-pseudolive
```

此分支的目標不是立刻刪除所有 pseudo-live 程式碼，而是先做最小改動驗證：

1. 保留現有 backfill
2. 暫時跳過 pseudo-live 切換
3. 直接用 historical relative URL 驗證 Grafana 行為

這樣比較容易把「Grafana 能不能取代 pseudo-live」和「整體重構是否完成」兩件事拆開。

---

## 實驗 branch 最小範圍

本次實驗 branch 採用更保守的落地方式，先不重構 backend replay pipeline，只在使用路徑上停止使用 pseudo-live。

### 保留

1. replay session 載入
2. replay 啟動時的 Prometheus backfill
3. replay `Pause / Scrub` 的 absolute trailing window
4. 既有 topology replay 與 event log 播放

### 暫停使用

1. `MetricPlayer`
2. `/api/replay/play`
3. `/api/replay/pause`
4. `/api/replay/speed`
5. `_live_...` pseudo session 切換
6. replay speed 對 Grafana 的同步

### 改用

replay `PLAYING` 時，Grafana 一律查原始 session：

```text
var-session=<orig_session>
from=now-<A>
to=now-<B>
refresh=5s
```

其中：

1. `B` 代表目前 playhead 距離現在的 offset
2. `A - B` 等於目前 chart window
3. 前端根據 replay session 的原始 timestamp 與當前 wall clock 動態計算 `A/B`

### 本次實驗不處理

1. Prometheus 跨重啟保留 session 的正式方案
2. pseudo-live backend 程式碼刪除
3. replay speed 的替代設計
4. historical relative window 的正式 API 命名與抽象

### 成功條件

若以下條件成立，即可視為這個 branch 的實驗成功：

1. `./start.sh --replay sessions/<id> --force-backfill` 後，原始 session 可正常顯示圖表
2. replay `Play` 時不再呼叫 pseudo-live API
3. Grafana 在固定 iframe `src` 下可持續 auto-refresh，且畫面不會像 absolute window 那樣反覆閃爍
4. replay `Pause / Scrub / Resume` 仍可用
5. speed 即使被停用，也不影響「用戶可透過 pause/scrub 觀察歷史」這個核心目的
