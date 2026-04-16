# Metric Event Modeling Notes

本文件整理 DVR / replay 實作過程中發現的 metrics 建模問題，以及 2026-04-10 開會提出的兩個
Grafana 顯示問題。這些議題影響 live 與 replay 兩條路徑，不應被視為 DVR 單一路線的後續工作，
因此從 DVR 主線抽離並整理到 `plans/architecture/`。

## 範圍

主要討論：

- `retrain` 事件是否應該繼續以 counter 推導
- event metric、counter、gauge pulse 的語意差異
- live / replay / backfill / pseudo-live 是否應共享同一套事件建模
- retrain 結束 annotation 缺失問題（2026-04-10 開會）
- deviation panel sMAPE 標示問題（2026-04-10 開會）

---

## 待解決問題（2026-04-10 開會）

### 問題 1：Retrain 結束的線、斷掉的點

Grafana 右下角 deviation panel 觀察到的現象：

- 目前只有 retrain **開始**的紅色 annotation（對應 `retrain_trigger`），沒有結束標記。
- Retraining 進行中，舊 model 繼續推論與 accuracy check，deviation 曲線**不中斷**。
- Model swap 完成後（`model_swap` event），`_on_model_swap` 把舊 model 的 deviation label set
  從 Prometheus 移除；新 model 要等到下一次 accuracy 取樣（約 50s）才會回報第一筆數值，造成
  曲線出現**短暫 sampling gap**。
- 觀看者看到「中間斷了一段」但沒有任何視覺標記說明原因，難以解釋。

**事件時序（正確理解）**：

```
retrain_trigger  →  [Daisy training, 持續數分鐘]  →  retrain_done  →  model_swap
    ↑                                                      ↑               ↑
 紅色 annotation（已有）                              需要加綠色      deviation gap 從這裡開始
```

**根本原因**：
- `retrain_done` event 在 `rules/nwdaf.py` 已被識別（log 規則存在），但 `METRIC_HANDLERS`
  沒有對應 handler，也沒有 Prometheus metric。
- Grafana 無法標記 retrain 結束時間點，觀看者無法將「sampling gap」與「model swap」對應。

### 問題 2：Deviation panel 指標標示

- `_build_deviation_panel` 的 title 目前是 `"Model Deviation"`，未說明這是 sMAPE。
- Y 軸沒有 label，legend 只顯示 model 名稱，觀看者無法辨識縱軸的指標語意。

**此問題與 metric 建模無關，只涉及 `grafana_setup.py` 的 panel 設定。**

---

## retrain_done 的 log 捕捉確認

`rules/nwdaf.py` 已有規則，捕捉 Daisy 非同步 callback：

```python
# MTLF: retraining completed (async mode — Daisy callback received)
{
    "match": {
        "nf": "NWDAF",
        "cat": "MTLF",
        "msg": re.compile(r"Async training complete: taskId=\S+ modelUrl=\S+"),
    },
    "event": "retrain_done",
    "build": lambda m, base: {},
},
```

目前缺失：
- `METRIC_HANDLERS` 沒有 `"retrain_done"` 入口 → event 進得來，但完全不接 Prometheus。
- 沒有對應 Prometheus metric。
- Grafana annotation 沒有對應 query。

**覆蓋範圍說明**：`retrain_trigger` 的 comment 標注「both ADRF and non-ADRF paths」，即有無 ADRF
資料取回，最終都送 Daisy 進行訓練；`retrain_done` 捕捉的是 Daisy async callback，涵蓋兩條路徑。
因此「Async training complete」是可靠的訓練完成信號。

**備注**：即便缺少 retrain_done 的某些邊界情況，`model_swap`（「Model hot-swap completed」）
也可作為訓練完成的後備信號 —— hot-swap 完成代表新 model 必然已就緒。

---

## 背景

目前 `retrain` annotation 仍沿用：

```promql
idelta(nwdaf_retrain_total{session="$session"}[15s]) > 0
```

這種做法的問題在於：

- 事件本質上是離散事件，但目前用 counter 的變化來推導。
- 取樣密度會影響 annotation 是否能被偵測到。
- replay / backfill / pseudo-live 為了讓 annotation 成立，需要額外照顧 sample pattern。

因此它較像是「現有系統可運作的折衷表示法」，而不是語意最直接的建模。

---

## 與 DVR 的關係

下列內容仍屬 DVR 主線，留在 `plans/dvr/`：

- replay pseudo-live 如何把目前的 metric 寫回 Prometheus
- pre-seed 如何配合現有 dashboard / annotation

下列內容屬於更高一層的 metrics 建模議題，留在本文件：

- `retrain` 是否應該改為 event metric
- retrain_done 如何加入建模
- 若改動，如何同時適用於 live 與 replay
- 是否要保留 `nwdaf_retrain_total` 只做統計用途

---

## 目前已知事實

1. 現有 `counter + idelta()` 方案可以運作，但語意較曲折。
2. replay 較容易暴露這個問題，因為回填與 pseudo-live 都要顧及 sample pattern。
3. 問題不是 replay 專屬；只要 annotation 想要穩定表達某個離散事件，live 也會受同一套建模
   限制影響。
4. `retrain_done` 已有 log 捕捉規則（`rules/nwdaf.py`），但尚未接入 Prometheus。
5. Replay backfill（`main.py _build_replay_metric_series`）目前只處理 `retrain_trigger`，
   若加入 retrain_done event metric，backfill 也需同步更新。

---

## 待評估方向

### 1. 維持現狀：counter + idelta / increase

- 變更最小。
- retrain_done 新增一個 counter，annotation 也用 idelta。
- 繼續承受 sample density 與 query 語意帶來的複雜度，replay 仍需額外照顧。

### 2. Gauge pulse

- 每次事件發生時設為 1，再回到 0（寬度 > 1 個 scrape interval，約 8s）。
- 比 counter 直覺；annotation query 用 `metric > 0`，語意明確。
- Pulse reset 需要非同步機制：在 `main.py` 的 `_process_queue`（asyncio context）中，
  於 `_update_metrics(event)` 之後為 retrain 事件排 `asyncio.create_task(reset_after_delay())`。
- Replay backfill 需為每個 retrain 事件寫 sample pair：`(event_ts, 1.0)` 與
  `(event_ts + 8000ms, 0.0)`。

### 3. 獨立 event metric（語意最直接）

- 將 `retrain_start` 與 `retrain_done` 明確表示為事件，而不是從 counter 變化推導。
- 語意最直接，也較符合 debug 與 replay 的需求。
- 需要單獨設計 label 與 cardinality 控制；實作複雜度較高。

---

## 實作方向決定

採用**方案 2（Gauge pulse）**：

- 保留 `nwdaf_retrain_total` Counter 供統計與趨勢查詢使用。
- 新增 `nwdaf_retrain_start_event` 與 `nwdaf_retrain_done_event` 兩個 Gauge pulse，
  專供 Grafana annotation 使用。
- Live 路徑：pulse reset 在 `main.py` 的 asyncio context 中排程（handler 本身同步）。
- Replay 路徑：backfill 時為 retrain 事件寫 `(ts, 1.0)` + `(ts + 8000ms, 0.0)` sample pair。
- Grafana annotation query 改為 `nwdaf_retrain_start_event{session="$session"} > 0`
  與 `nwdaf_retrain_done_event{session="$session"} > 0`。

**sMAPE 標示**（問題 2，獨立處理）：

純粹修改 `grafana_setup.py` 的 `_build_deviation_panel`，不涉及 metric 建模：

- title → `"Model sMAPE"`
- `fieldConfig.defaults.custom` 加 `axisLabel: "sMAPE"`
- `legendFormat` → `"{{model}} sMAPE"`

---

## 實作任務

| # | 任務 | 檔案 | 狀態 |
|---|---|---|---|
| 1 | 加 `nwdaf_retrain_start_event` Gauge | `rules/nwdaf.py` | ✅ |
| 2 | 加 `nwdaf_retrain_done_event` Gauge | `rules/nwdaf.py` | ✅ |
| 3 | Pulse reset 機制（live 路徑） | `main.py` | ✅ |
| 4 | Replay backfill 更新 | `main.py` | ✅（有 bug，見下） |
| 5 | Grafana annotation 更新 | `grafana_setup.py` | ✅ |
| 6 | sMAPE panel 標示 | `grafana_setup.py` | ✅ |
| 7 | Backfill pulse 寬度修正 | `main.py` | ⬜ |
| 8 | MetricPlayer pseudo-live annotation 補寫 | `metric_player.py` | ⬜ |
| 9 | Grafana session 選單修正 | `grafana_setup.py` | ⬜ |

---

## 實作紀錄（2026-04-16）

### 已完成（任務 1–6）

初版實作完成。Live 路徑（`_process_queue` → gauge pulse → asyncio reset）與 Grafana
dashboard 設定（annotation query、sMAPE panel）運作正常。

Replay backfill 路徑（`_build_replay_metric_series`）也已更新，寫入 retrain start/done
pulse 的 sample pair。

---

### 測試發現的問題

#### Bug 1：Backfill 模式下 annotation 線重複兩次

**現象**：pause 狀態時，紅色與綠色 annotation 線各出現兩次，時間間距約 5 秒。

**根因**：Grafana annotation 以 5s 為步距掃描時間軸。Backfill 寫入的 pulse：
```
(ts_ms,        1.0)   ← on
(ts_ms + 8000, 0.0)   ← off，8s 後
```
8 秒跨越兩個 5s 步距區間，兩個取樣點都讀到值 1.0，因此畫出兩條線。

**修正方向（任務 7）**：將 off 樣本改為 `ts_ms + 5000`（剛好一個 scrape interval）。
數學上，任意步距對齊情況下，`[ts_ms, ts_ms+5000)` 範圍內只會有一個步距邊界落入，
確保恰好一個取樣點看到值 1.0。

---

#### Bug 2：Pseudo-live（play）模式所有線消失且不回來

**現象**：按下 play 後，Grafana 圖上的 annotation 線與 chart 曲線全部消失，即使
emit loop 持續運作也不恢復。

**根因分析**（兩個獨立問題）：

**① Annotation 線消失（任務 4 遺漏）**

Replay 有兩條路徑：
- Backfill（pause）：由 `_build_replay_metric_series` 寫歷史樣本到 Prometheus
- Pseudo-live（play）：由 `MetricPlayer._build_metric_series_for_event` 即時寫樣本

任務 4 只補了 backfill 路徑。`metric_player.py` 的
`_build_metric_series_for_event` 完全沒有寫 `nwdaf_retrain_start_event` 或
`nwdaf_retrain_done_event`，pseudo_session 對這兩個 metric 永遠是空的，
Grafana annotation query 永遠回傳空，annotation 線不出現。

**② Chart 曲線（GT / Pred / Deviation）消失（pre-existing 問題）**

Grafana session 下拉選單的來源 query：
```promql
label_values(nwdaf_ground_truth_ul_bytes, session)
```
Play 開始時，pseudo_session（`_live_...`）尚無 `nwdaf_ground_truth_ul_bytes`
（preseed 若為空則 GT 資料尚未寫入），session 選單找不到 `_live_...`。
Grafana 收到 URL 參數 `var-session=_live_...` 但選單中無此選項，
自動 fallback 到第一個有效 session（原始 session）。
原始 session 的資料時間戳為歷史時間，不在 `now-3m ~ now` 視窗內，所有 panel 為空。

此外，template variable 的 `refresh` 設為 1（只有 dashboard 載入時才重新查詢），
即使之後 `_live_...` 有了資料，選單也不更新，session 不切換，chart 線不回來。

此問題與本次 annotation 變更無關，為 pre-existing 問題，但兩者疊加導致 play 畫面全空。

#### Bug 3：Retrain 後舊 model 的 deviation 線被平推到下一個 model 第一筆 accuracy

**現象**：在 replay / pseudo-live 中，`retrain_done` / `model_swap` 之後，前一個 model 的
deviation 線沒有立刻斷掉，而是以最後一筆值持續水平延伸，直到下一個 model 出現第一筆
`accuracy` 為止。這和 live mode 看到的冷啟動空窗不一致。

**根因**：live mode 在 `model_swap` 會直接刪除 exporter 內舊的
`nwdaf_deviation{session,model}` series，Prometheus scrape 後就會出現斷點；但 replay backfill
與 pseudo-live 只有重播數值樣本，沒有重播這個「series 被終止」的語意，所以 query 仍會拿舊
model 的最後一筆 deviation 當成目前可見 series。

**修正方向**：在 replay backfill 與 pseudo-live 的 `model_swap` 處，為當前舊 model 寫一筆
`NaN` cut-off 樣本，強制 Grafana 在 swap 時把線段截斷，保留部署後 cold start 的空窗。

---

### Replay 兩個模式功能對照

| 項目 | Backfill（pause） | Pseudo-live（play） |
|---|---|---|
| GT / Pred / Deviation chart 線 | ✅ | ❌ session 選不到 `_live_...`（pre-existing）|
| sMAPE panel 標示 | ✅ | ✅ |
| Retrain Start annotation（紅線） | ⚠️ 出現但重複（Bug 1） | ❌ metric 未寫入 |
| Retrain Done annotation（綠線） | ⚠️ 出現但重複（Bug 1） | ❌ metric 未寫入 |

---

### 補充修正計畫

**任務 7：Backfill pulse 寬度修正（`main.py`）**

`_build_replay_metric_series` 中 `nwdaf_retrain_start_event` 與
`nwdaf_retrain_done_event` 的 off 樣本時間戳從 `ts_ms + 8000` 改為
`ts_ms + 5000`（定義為常數 `_BACKFILL_PULSE_MS = 5000`）。

**任務 8：MetricPlayer pseudo-live annotation 補寫（`metric_player.py`）**

在 `_build_metric_series_for_event` 中，為 retrain 事件補寫 pulse 樣本：

```python
_PULSE_MS = 5000  # one Prometheus scrape interval

# retrain_trigger → nwdaf_retrain_start_event pulse
# retrain_done   → nwdaf_retrain_done_event pulse
# 各寫 (ts_ms, 1.0) 與 (ts_ms + _PULSE_MS, 0.0)
```

Preseed 與 emit loop 共用同一個 `_build_metric_series_for_event`，
補一處即同時覆蓋兩個場景。

**任務 9：Grafana session 選單修正（`grafana_setup.py`）**

兩處調整：

1. Template variable query 從 `nwdaf_ground_truth_ul_bytes` 改為
   `nwdaf_retrain_total`。`MetricPlayer._prepare_state_and_preseed` 無論
   preseed 是否為空，**一定**會為 pseudo_session 寫一筆 `nwdaf_retrain_total`
   錨點樣本，確保 `_live_...` 一定出現在選單中。

2. Template variable `refresh` 從 `1`（dashboard 載入時）改為 `2`
   （time range 改變時）。Play 時 Grafana 切換時間窗口到 `now-3m ~ now`，
   觸發選單重新查詢，session 正確切換到 `_live_...`。

---

## 結論

這是 DVR 實作過程中發現的延伸建模問題，同時也對應 2026-04-10 開會提出的顯示改善需求。
決定採用 Gauge pulse event metric，兼顧語意清晰與 replay 相容性。

初版實作（任務 1–6）已完成，但測試發現 replay 的兩個模式未完整覆蓋：

- **Backfill**：pulse 寬度過寬導致 annotation 重複（任務 7 修正）
- **Pseudo-live**：annotation metric 未接入 MetricPlayer（任務 8），
  session 選單無法找到 pseudo_session（任務 9，pre-existing 問題）
- **Deviation gap**：replay / pseudo-live 缺少 live `model_swap` 的 series terminate 語意，
  需補 `NaN` cut-off 樣本，否則舊 model 會被平推到新 model 第一筆 accuracy。

任務 7–9 為補充修正，完成後 live / backfill / pseudo-live 三條路徑才全部對齊。
