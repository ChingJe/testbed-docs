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

原始問題是 `retrain` annotation 曾沿用：

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

1. `counter + idelta()` 方案曾經可以運作，但語意較曲折。
2. replay 較容易暴露這個問題，因為回填與 pseudo-live 都要顧及 sample pattern。
3. 問題不是 replay 專屬；只要 annotation 想要穩定表達某個離散事件，live 也會受同一套建模
   限制影響。
4. `retrain_done` 現已接入 Prometheus，和 `retrain_trigger` 一樣用 pulse metric 表達。
5. Replay backfill 與 `MetricPlayer` 也都已同步補上 retrain pulse 與 `model_swap` cut-off。

---

## 待評估方向

### 1. 維持現狀：counter + idelta / increase

- 變更最小。
- retrain_done 新增一個 counter，annotation 也用 idelta。
- 繼續承受 sample density 與 query 語意帶來的複雜度，replay 仍需額外照顧。

### 2. Gauge pulse

- 每次事件發生時設為 1，再回到 0（寬度 = 1 個 scrape interval，5000ms）。
- 比 counter 直覺；annotation query 用 `metric > 0`，語意明確。
- Pulse reset 需要非同步機制：在 `main.py` 的 `_process_queue`（asyncio context）中，
  於 `_update_metrics(event)` 之後為 retrain 事件排 `asyncio.create_task(reset_after_delay())`。
- Replay backfill 需為每個 retrain 事件寫 sample pair：`(event_ts, 1.0)` 與
  `(event_ts + 5000ms, 0.0)`。

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
- Replay 路徑：backfill / pseudo-live 都為 retrain 事件寫 `(ts, 1.0)` + `(ts + 5000ms, 0.0)` sample pair。
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
| 4 | Replay backfill 更新 | `main.py` | ✅ |
| 5 | Grafana annotation 更新 | `grafana_setup.py` | ✅ |
| 6 | sMAPE panel 標示 | `grafana_setup.py` | ✅ |
| 7 | Backfill pulse 寬度修正 | `main.py` | ✅ |
| 8 | MetricPlayer pseudo-live annotation 補寫 | `metric_player.py` | ✅ |
| 9 | Grafana session 選單修正 | `grafana_setup.py` | ✅ |
| 10 | Replay / pseudo-live deviation gap 修正 | `main.py`、`metric_player.py` | ✅ |

---

## 實作紀錄（2026-04-16）

### 最終落地結果

實作已補齊到 live、replay backfill、pseudo-live 三條路徑一致：

- live：`retrain_trigger` / `retrain_done` 透過 pulse gauge + asyncio reset 顯示 annotation。
- replay backfill：改為寫入 `start/done` pulse sample pair，pulse 寬度固定為一個 scrape interval
  `5000ms`，避免 annotation 重複。
- pseudo-live：`MetricPlayer` 的 pre-seed 與 emit loop 共用同一套 event → metric 映射，
  因此 `retrain_start_event`、`retrain_done_event`、`retrain_total` 與 deviation cut-off
  都會一起生效。
- Grafana：annotation query 改查 pulse gauge；session variable 改為查 retention 內仍有樣本的
  session，而不是直接讀 Prometheus label index。

### 這次補完的三個關鍵修正

#### 1. Backfill annotation 重複

初版 backfill 使用 `ts_ms + 8000` 作為 pulse off 樣本，會跨過兩個 5 秒步距。最後已改成：

```text
(ts_ms, 1.0)
(ts_ms + 5000, 0.0)
```

這讓 pause / scrub 狀態的 retrain start / done annotation 各只打一條線。

#### 2. Pseudo-live session 與 annotation 缺口

`MetricPlayer` 現在會：

- 在 `retrain_trigger` 寫 `nwdaf_retrain_start_event`
- 在 `retrain_done` 寫 `nwdaf_retrain_done_event`
- 在每個 mapped timestamp 寫一筆 `nwdaf_retrain_total`
- pre-seed 結束時再補一筆 `now_ms` 錨點樣本

因此 pseudo-live 一開始就能被 Grafana session variable 選到，annotation 也不再只在 backfill 有。

#### 3. Deviation cold-start gap

live 路徑在 `model_swap` 會移除舊 `nwdaf_deviation{session,model}` series；replay remote write
無法重播 exporter 的 `remove()`，所以最終採用 `NaN` cut-off sample 模擬「series terminate」語意：

- replay backfill：在 `model_swap` 時對所有 active models 寫 `NaN`
- pseudo-live：在 `MetricPlayer._build_metric_series_for_event()` 做同樣的 `NaN` cut-off

結果是舊 model 的 deviation 線會在 swap 當下截斷，後續保留新 model 部署後的 cold-start 空窗。

### Replay 兩個模式功能對照

| 項目 | Backfill（pause） | Pseudo-live（play） |
|---|---|---|
| GT / Pred / Deviation chart 線 | ✅ | ✅ |
| sMAPE panel 標示 | ✅ | ✅ |
| Retrain Start annotation（紅線） | ✅ | ✅ |
| Retrain Done annotation（綠線） | ✅ | ✅ |
| `model_swap` 後 deviation gap | ✅（`NaN` cut-off） | ✅（`NaN` cut-off） |

---

## 結論

這次重構最後不是只把 retrain annotation 從 counter 推導改成 pulse gauge，而是順手把 replay /
pseudo-live 兩條路徑裡原本不完整的建模也補齊：

- retrain start / done 改為顯式 metric event
- session variable 改為查實際仍有樣本的 session
- replay / pseudo-live 補上 live `model_swap` 的終止語意

因此目前 live、backfill、pseudo-live 三條圖表路徑，對 retrain 與 model deployment 的呈現已經對齊。
