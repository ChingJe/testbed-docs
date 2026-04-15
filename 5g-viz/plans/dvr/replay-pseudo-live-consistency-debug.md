# Replay Pseudo-Live Consistency: Debug Plan

本文件是 [replay-pseudo-live-consistency.md](replay-pseudo-live-consistency.md) 的配套診斷計畫，目的是定位 Pause（backfill）與 Play（pseudo-live）圖表不一致的根因。

## 問題摘要

replay mode 下，使用者在同一個 scrub 位置 Pause 看到的 Grafana 圖表，與 Play 後看到的圖表數值不同。設計上 pre-seed 保留了原始事件的相對間距與值，理論上兩條路徑應產生一致的圖表，但實測觀察到明顯差異。

## 根因可能在三個層次

| 層次 | 描述 | 意義 |
|---|---|---|
| **A. 寫入層** | backfill 和 pre-seed 寫進 Prometheus 的 raw sample 值或 timestamp 分佈本身就不同 | 需要修 MetricPlayer 或 backfill 邏輯 |
| **B. 查詢層** | raw sample 相同，但 Grafana range query 的 step / evaluation 導致不同結果 | 需要對齊 Grafana query 參數或 timestamp quantization |
| **C. 渲染層** | query 結果相同，但 Grafana 視覺化 / interpolation 差異 | 需要調整 panel 設定或 iframe 參數 |

確認是哪一層後，修復方向完全不同。以下診斷步驟從最底層（A）往上排查。

---

## 前置準備

### 環境

- 5g-viz replay mode 能正常啟動
- Prometheus 運行中（含 `--web.enable-remote-write-receiver` 和 `--web.enable-admin-api`）
- 有一個含足夠 metric event 的 session（至少 1 分鐘，建議 3 分鐘以上）

### 選定觀察位置

選一個 session 中段的 scrub 位置 `T_scrub`，確保前後都有 metric event。記下：

- `SESSION_ID`：replay 的 session ID
- `T_scrub`：ISO 8601 格式（例如 `2026-04-15T06:33:30+00:00`）
- `T_scrub_epoch`：對應的 epoch seconds（例如 `1744695210`）

---

## Step 1：比對 Prometheus raw samples（定位 A 層）

目標：確認 backfill session 和 pseudo_session 寫入 Prometheus 的 **raw sample 值** 是否一致。

### 1.1 查 backfill 資料

Replay 啟動後，backfill 已完成。直接用 Prometheus range query API 拉 raw samples：

```bash
# 拉 backfill session 在 T_scrub 前後 1 分鐘的 ground truth UL
curl -s 'http://localhost:9090/api/v1/query_range' \
  --data-urlencode 'query=nwdaf_ground_truth_ul_bytes{session="<SESSION_ID>"}' \
  --data-urlencode 'start=<T_scrub_epoch - 60>' \
  --data-urlencode 'end=<T_scrub_epoch>' \
  --data-urlencode 'step=1s' \
  | python3 -m json.tool > /tmp/backfill_raw.json
```

記錄：每個 step 點的 `[timestamp, value]` pair。

### 1.2 觸發 pseudo-live pre-seed

在前端 scrub 到 `T_scrub`，按 Play。從 server log 或 `/api/replay/play` 的 response 取得 `PSEUDO_SESSION` 值。

**立即** Pause（目的是只看 pre-seed 資料，不混入 emit loop 資料）。

### 1.3 查 pseudo-live 資料

```bash
# 拉 pseudo_session 的 ground truth UL（時間範圍用 now-relative，因為 pseudo-live 映射到 now 附近）
# 先確認 pseudo_session 有資料
curl -s 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=count(nwdaf_ground_truth_ul_bytes{session="<PSEUDO_SESSION>"})' \
  | python3 -m json.tool

# 拉 range 資料（pre-seed 寫入的 timestamp 在 play 按下的那一刻的 now-3min ~ now 附近）
curl -s 'http://localhost:9090/api/v1/query_range' \
  --data-urlencode 'query=nwdaf_ground_truth_ul_bytes{session="<PSEUDO_SESSION>"}' \
  --data-urlencode 'start=<play_start_epoch - 240>' \
  --data-urlencode 'end=<play_start_epoch + 10>' \
  --data-urlencode 'step=1s' \
  | python3 -m json.tool > /tmp/pseudo_raw.json
```

### 1.4 比對

比對 `/tmp/backfill_raw.json` 和 `/tmp/pseudo_raw.json`：

1. **兩組的 sample 數量是否一致？**（同一段 session 歷史應產生相同數量的 metric sample）
2. **value 是否完全相同？**（只是 timestamp 不同，值應該一樣）
3. **timestamp 間隔是否一致？**（backfill 間隔 ~5s，pseudo-live 映射後也應 ~5s / speed）

**判讀**：
- 若 value 不同 → **A 層問題**，需要 debug `_build_metric_series_for_event` 和 `_build_replay_metric_series` 的值計算邏輯
- 若 sample 數量不同 → **A 層問題**，需要 debug 事件篩選邏輯（METRIC_EVENT_TYPES 過濾等）
- 若 value 和數量都一致 → A 層沒問題，繼續 Step 2

---

## Step 2：比對 Grafana 實際 query 參數（定位 B 層）

目標：確認 Grafana 對兩條路徑發出的 range query 是否使用相同的 `step` 和 `from/to`。

### 2.1 取得 Grafana query 參數

兩種方式（擇一）：

**方式 A：Query Inspector**

暫時移除 Grafana iframe URL 中的 `&kiosk`（修改前端 `_grafanaUrl()` 或直接在瀏覽器開 Grafana dashboard），然後：

1. Pause 狀態 → 打開任一 panel 的 Query Inspector → 記錄 Request 中的 `start`, `end`, `step` 參數
2. Play → 打開 Query Inspector → 記錄同樣的參數

**方式 B：Prometheus query log**

在 Prometheus 設定中開啟 query log（`--query.log-file`），然後觀察兩次操作分別發出的 range query。

### 2.2 比對

比對兩次的 `step` 值：

- 若 step 不同 → 可能是 Grafana 對 absolute timestamp 和 `now-Nm` 選擇了不同的 step resolution
- 若 step 相同 → B 層的 step 差異不是原因

同時檢查：用相同的 step 對兩組 raw data 做手動 evaluation，看結果是否一致。可以用以下 Python script：

```python
# 模擬 Prometheus range query evaluation
# 輸入：samples = [(ts_ms, value), ...], query_start_ms, query_end_ms, step_ms
# 在每個 step 點，取 timestamp <= step_point 且在 5 分鐘 lookback 內的最新 sample

def evaluate_range(samples, start_ms, end_ms, step_ms):
    lookback_ms = 5 * 60 * 1000
    results = []
    t = start_ms
    while t <= end_ms:
        best = None
        for ts, val in samples:
            if ts <= t and ts >= t - lookback_ms:
                if best is None or ts > best[0]:
                    best = (ts, val)
        results.append((t, best[1] if best else None))
        t += step_ms
    return results
```

用此 script 分別對 backfill 和 pseudo-live 的 raw samples 做 evaluation，比對結果。

**判讀**：
- 若手動 evaluation 結果不同 → **B 層問題**，需要調整 timestamp quantization 或 Grafana query step alignment
- 若手動 evaluation 結果相同但 Grafana 顯示不同 → 繼續 Step 3

---

## Step 3：比對 Grafana 渲染結果（定位 C 層）

目標：排除 Grafana 視覺化層的差異。

### 3.1 用 Query Inspector 比對 raw data

在 Grafana Query Inspector 中切到 "Data" tab，看 Grafana 渲染前的 raw data points：

1. Pause 狀態 → 記錄 Data tab 的值
2. Play → 記錄 Data tab 的值

### 3.2 檢查 panel 設定

確認兩次查詢使用完全相同的 panel 設定：
- `connect null values` 設定
- `fill below to` / `stack` 設定
- tooltip mode
- y-axis scale

**判讀**：
- 若 raw data 相同但圖表看起來不同 → C 層（渲染），可能是 Grafana 的時間軸對齊、interpolation 或 auto-scaling
- 若 raw data 就不同 → 回到 Step 2 深入比對 query evaluation

---

## 額外排查：Emit Loop 漂移（如果 A/B/C 都沒發現問題）

如果純 pre-seed 階段沒問題，但 Play 一段時間後開始出現差異，問題可能在 emit loop 的 wall-clock jitter。

### 驗證方式

1. Play 開始後**立即 Pause**（只看 pre-seed） → 截圖 Grafana
2. 從同一位置再次 Play，**等 30 秒後 Pause** → 截圖 Grafana
3. 比對兩張圖的「重疊歷史區間」（pre-seed 部分應相同，emit loop 部分可能有漂移）

---

## 2026-04-15 Runtime 診斷結果

以下結果來自實際對本機 replay / Prometheus / Grafana runtime 的驗證。

### 測試條件

- 診斷日期：`2026-04-15`
- Replay session：`20260413T110900802`
- Session 範圍：`2026-04-13T19:09:00.802+08:00` ~ `2026-04-13T19:27:08.841+08:00`
- Scrub 位置：`2026-04-13T11:17:06.293Z`（=`2026-04-13T19:17:06.293+08:00`）
- Chart window：`3 min`
- 主要觀察 panel：
  - `sum by (target)(nwdaf_ground_truth_ul_bytes{session="$session",target="group=group-test-002"})`
  - `sum by (target)(nwdaf_predicted_ul_bytes{session="$session",target="group=group-test-002"} offset 5s)`
- `nwdaf_retrain_total` 本次**刻意排除**，後續由 [../architecture/metric-event-modeling.md](../architecture/metric-event-modeling.md) 處理；本文件只判讀主圖（GT / Pred / Deviation）一致性

### Step 1 結果：A 層對主圖 metric 正常

對同一個 scrub 位置，分別查：

- 原始 replay backfill session：`session="20260413T110900802"`
- 每次 `Play` 產生的新 `pseudo_session`

並直接打 Prometheus `/api/v1/query_range` 做 `step=1s` 查詢，比對 3 分鐘視窗內的主圖值。

#### 觀察

- `nwdaf_ground_truth_*`：backfill 與 pseudo-live 的 Prometheus `query_range(step=1s)` 結果一致
- `nwdaf_predicted_*`：在 Prometheus raw query 層一致
- `nwdaf_deviation`：先前離線重建比對也一致
- 代表性驗證：
  - backfill query：`sum by (target)(nwdaf_ground_truth_ul_bytes{session="20260413T110900802",target="group=group-test-002"})`
  - pseudo query：`sum by (target)(nwdaf_ground_truth_ul_bytes{session="<pseudo_session>",target="group=group-test-002"})`
  - 結果：`181` 個點對 `181` 個點，`diff = 0`

#### 判讀

對主圖 metric 而言，backfill 與 pseudo-live 的 raw sample 值沒有分岔；A 層不是本次主要根因。

### Step 2 結果：B 層已確認是主因

這一步改用 Grafana 自己的 datasource query API：`/api/ds/query`，避免只靠 Prometheus API 推測。

#### 觀察 1：在手動 `/api/ds/query` 測試中，Grafana 可被迫以 `step = 1s` 查詢

- backfill：Grafana `executedQueryString` 顯示 `Step: 1s`
- pseudo-live：Grafana `executedQueryString` 也顯示 `Step: 1s`

這只說明在當時手動構造的 API 測試條件下，不是「backfill 用 1s、pseudo-live 用 15s」這種粗粒度 step 差異；它**不等於**實際瀏覽器匯出路徑也一定是 `1s`。

#### 觀察 2：同樣 `step = 1s`，Grafana frame 值仍然不同

代表性結果如下：

```text
GT UL / backfill first 12:
8423, 11207, 11207, 11207, 11207, 11207, 4174, 4174, 4174, 4174, 4174, 51731

GT UL / pseudo-live first 12:
11207, 11207, 11207, 11207, 11207, 4174, 4174, 4174, 4174, 4174, 51731, 51731

Pred UL / backfill first 12:
3546, 4163, 4163, 4163, 4163, 4163, 3426, 3426, 3426, 3426, 3426, 2931

Pred UL / pseudo-live first 12:
3426, 3426, 3426, 3426, 3426, 2931, 2931, 2931, 2931, 2931, 7377, 7377
```

這不是 sample 值本身變了，而是 evaluation grid 對到的「最近一筆 sample」不同，整段序列往前或往後平移。

#### 觀察 3：Prometheus raw 一致，但 Grafana datasource query 不一致

同一組 session / target：

- 直接打 Prometheus `/api/v1/query_range` → backfill 與 pseudo-live 一致
- 經 Grafana `/api/ds/query` → backfill 與 pseudo-live 不一致

這將問題明確收斂到 Grafana query layer，而不是 MetricPlayer / backfill 寫入邏輯。

#### 觀察 4：intermittent 現象來自 query timing / phase

使用同一個 scrub 點重複 `Play` 多次，得到不同 `pseudo_session`。實測發現：

- 有些 `Play` 後 mismatch 較小
- 有些 `Play` 後 mismatch 較大
- 即使固定同一個 `pseudo_session` 不變，只改 Grafana 查詢的 `to` 時刻，diff 也會改變

代表性結果：

- 對同一個 `pseudo_session`，只把 Grafana query `to` 往後推 `1.5s`
- GT UL 的 `diff` 由 `37` 變成 `73`

這說明「有時一致、有時不一致」不是因為 raw sample 每次 `Play` 都不同，而是因為 `now-relative` 查詢的 evaluation 時刻跟 sample phase 的相對位置會變。

### Step 3 結果：C 層不是主要根因

因為在 Grafana render 前的 datasource frame 值就已經不同，所以不需要把根因歸到 panel interpolation、auto-scale 或 tooltip/rendering。

換句話說，問題在 render 之前就已經發生。

### 造成 B 層問題的具體原因

本次確認的實際機制如下：

1. `Pause / backfill` 路徑走 absolute trailing range，前端 `_trailingGrafanaRange()` 會先把 `from/to` 做 `Math.floor()` 後再送進 Grafana，見 `frontend/events.js`。
2. `Play / pseudo-live` 路徑走 `now-3m ~ now` relative range，Grafana 每次 refresh 都以當下 wall clock 重新決定 query 邊界。
3. 主圖資料是 sparse gauge-like sample（約每 5 秒一筆）；Prometheus range evaluation 在每個 step 點取 lookback 內最新 sample。
4. 當 absolute range 與 `now-relative` range 的時間格線相位不同，即使 raw sample 完全相同，也會在某些 step 點取到不同的最新 sample。
5. `Pred UL / DL` 額外帶 `offset 5s`，所以對 phase shift 更敏感，視覺差異比 ground truth 更明顯。

### 本次結論

- 主圖（`nwdaf_ground_truth_*` / `nwdaf_predicted_*` / `nwdaf_deviation`）的 raw sample 在 A 層沒有發現不一致
- 問題主因是 B 層：Grafana query 的時間軸對齊與 evaluation phase
- intermittent 的原因是 `Play` 使用 `now-relative` 視窗；每次 `Play` 與每次 refresh 的 wall-clock phase 不同，因此有時看起來接近、有時差很多
- `retrain_total` 的不一致另案處理，不納入本次主因

### 修復方向建議

若要讓 Pause / Play 的圖更穩定，修復方向應優先放在時間對齊，而不是 sample 值重建：

1. 讓 backfill absolute query 與 pseudo-live mapped timestamp 共用同一種時間量化策略
2. 避免 Pause / Play 使用兩種 phase 明顯不同的 query 邊界
3. 若保留 `now-relative` 模式，需接受它是「近似 live」視圖，而非與 backfill 完全等價的歷史重建

---

## 2026-04-15 後續補充：CSV 證據與 5s 粒度修正

上一節的 runtime 診斷主要來自 Prometheus API 與手動構造的 Grafana `/api/ds/query` 測試。之後使用者補充了**實際從瀏覽器匯出的 CSV**，這讓判讀可以再收斂一步，也修正了「user-visible 路徑是否真的是 `1s`」這件事。

### 補充證據 1：修正前，實際匯出路徑曾落在 `15s` 粒度

使用者提供：

- `docs/5g-viz/tmp/group-test-001-data-2026-04-15 16_56_22.csv`
- `docs/5g-viz/tmp/group-test-001-data-2026-04-15 16_56_48.csv`

觀察結果：

- 兩份 CSV 都只有 `13` 個點
- 時間欄位是固定每 `15s` 一筆
- 同位置逐點比對是 `0 / 13` 相等

這代表先前的「`/api/ds/query` 可跑出 `step = 1s`」並不能代表實際 user-visible 匯出路徑；在這組案例裡，面板實際顯示/匯出看到的是 `15s` 級別的序列。

### 補充證據 2：為什麼會出現「整張圖 pattern 類似，但數值幾乎全不同」

同一段 replay 歷史中的 raw metric sample 仍然大致是 `5s` 一筆，但相鄰 `5s` sample 的數值可能差很多。例如同一段 `GT DL` raw sample 內可看到：

- `11:13:41` 附近約 `14.6 KiB`
- `11:13:46` 附近約 `208 KiB`
- `11:13:56` 附近約 `18.4 KiB`

當 Grafana 面板不是直接用 `5s` 顯示，而是先自動降採樣成 `15s` 粒度時，只要 Pause / Play 的取樣 phase 不同，每一個 `15s` checkpoint 都可能挑到完全不同的 `5s` raw sample。這會導致：

- 圖的大致高低起伏還有些相似
- 但 tooltip / CSV 上的實際值可以幾乎整列都不同

因此，這個現象不是單純的視覺插值，也不只是少數 sample 被平移，而是**面板在較粗粒度下對同一批 `5s` raw sample 做了不同 phase 的重取樣**。

### 修正內容

為了讓 user-visible 面板真的以 `5s` 粒度顯示，而不是讓 Grafana 自動降採樣，已在 `5g-viz/grafana_setup.py` 做以下修改：

- 將 Prometheus datasource 的 `jsonData.timeInterval` 固定為 `5s`
- 將主圖 panel targets 的 `interval` 固定為 `5s`
- 將 panel `maxDataPoints` 提高到 `4096`

目標是避免 `3 min` 視窗再次被自動壓成 `15s` 級別。

### 補充證據 3：修正後，CSV 已回到 `5s` 粒度且 Pause / Play 對齊

使用者之後提供較新的兩份 CSV：

- `docs/5g-viz/tmp/group-test-001-data-2026-04-15 17_17_48.csv`
- `docs/5g-viz/tmp/group-test-001-data-2026-04-15 17_18_08.csv`

觀察結果：

- 兩份 CSV 都是固定每 `5s` 一筆
- 第一份有 `27` 個點，第二份有 `28` 個點
- 在重疊區間內逐點比對，值序列為 `27 / 27` 完全相等

具體來說，兩份 CSV 的差異只剩下：

- 一份使用 pseudo-live 的現在時間軸
- 一份使用 replay 的歷史時間軸
- 其中一份尾端多一筆，表示匯出視窗長度沒有完全一樣

但在真正重疊的資料區間內，值本身已經對齊。

### 更新後的判讀

綜合上述補充，這次問題可更精確地分成兩段：

1. **修正前**
   主圖 panel 的 user-visible 路徑可能落在 `15s` 粒度，導致 Pause / Play 因 phase 差異而整列值幾乎都不同。

2. **修正後**
   主圖 panel 已回到 `5s` 粒度；在最新 CSV 驗證中，Pause / Play 的重疊區間值序列一致。

這表示主圖的一致性問題，至少在目前驗證到的案例中，已可藉由將 Grafana 面板粒度固定到 `5s` 來大幅改善，且最新證據顯示修正有效。

---
