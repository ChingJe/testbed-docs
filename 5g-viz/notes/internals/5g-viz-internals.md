# 5g-viz 內部運作原理

> 個人筆記，記錄 Prometheus、Grafana、5g-viz 三者的互動流程與設計原因。

---

## 整體架構

```
5GC VM (free5gc + NWDAF)
  │
  │  SSH tail（asyncssh，長連線）
  ▼
5g-viz（FastAPI，port 8765）
  │
  ├── parser.py     將 log 行解析成結構化 event
  ├── state.py      維護拓樸圖當前狀態
  ├── /ws           WebSocket，推送 event 給瀏覽器
  ├── /metrics      Prometheus scrape endpoint
  └── /             靜態前端（Cytoscape.js 拓樸圖 + Grafana iframe）
         ▲
         │  iframe embed
Grafana（port 3000）
  │
  │  PromQL 查詢（每 5s refresh）
  ▼
Prometheus（port 9090）
  │
  │  HTTP scrape（每 5s）
  └─── localhost:8765/metrics
```

---

## 資料流：從 NWDAF log 到 Grafana 曲線圖

### 1. Log 產生（5GC VM）

NWDAF 每 5 秒處理一次 UPF 通知，AnLF 聚合後輸出兩種關鍵 log：

```
# 實際流量（ground truth）
msg="latest aggregated slot: <sub_id> group=<group> ts=<ts>
     ulVol=N dlVol=N ..."  CAT="AnLF"

# ML 推論結果（預測值）
msg="ML inference: sub=<sub_id> group=<group> steps=1
     ulVol=N dlVol=N confidence=N"  CAT="AnLF"
```

兩條 log 在同一個處理週期內（相差 <1s）被輸出。

### 2. 5g-viz 收集與解析

`collector.py` 透過 SSH 對 5GC VM 執行 `tail -F`，把每一行 log 放進 asyncio Queue。

`parser.py` 從 Queue 取出，依序比對 `rules/` 底下的正則規則：
- `aggregated_slot` → 解析出 `sub_id`, `target`, `ul_vol`, `dl_vol`
- `ml_inference` → 解析出 `sub_id`, `target`, `ul_vol`, `dl_vol`, `confidence`

### 3. Prometheus metrics 更新

`main.py` 的 `_update_metrics()` 在收到 event 後立即更新 Gauge：

```python
# aggregated_slot event
nwdaf_ground_truth_ul_bytes{sub_id=..., target="group=group-test-001"}.set(ul_vol)
nwdaf_ground_truth_dl_bytes{sub_id=..., target="group=group-test-001"}.set(dl_vol)

# ml_inference event
nwdaf_predicted_ul_bytes{sub_id=..., target="group=group-test-001"}.set(ul_vol)
nwdaf_predicted_dl_bytes{sub_id=..., target="group=group-test-001"}.set(dl_vol)

# retrain_trigger event
nwdaf_retrain_total.inc()
```

這些 Gauge 存在 Python 程序的記憶體裡，`/metrics` endpoint 呼叫 `generate_latest()` 序列化成 Prometheus text format 回傳。

### 4. Prometheus scrape

Prometheus 每 5s 對 `localhost:8765/metrics` 發 HTTP GET，把當前的 Gauge 值存入自己的 TSDB（time series database，在 `~/prometheus/data/`）。

Prometheus 是 **pull-based**：主動來拿，不是 5g-viz 推過去。

### 5. Grafana 查詢與顯示

Grafana dashboard 每 5s 對 Prometheus 發 PromQL 查詢，畫出時間序列圖。

---

## 為什麼預測線需要 `offset 5s`

### 問題

`aggregated_slot` 和 `ml_inference` 在同一個處理週期（時間 T）被記錄，但它們描述的是不同的時間窗口：

```
時間 T 記錄的內容：
  ground truth → slot T 的實際流量
  prediction   → 對 slot T+5s 的預測（steps=1，往前一格）
```

如果直接畫，x 軸時間 T 上同時出現「T 的實際值」和「T+5s 的預測值」，兩條線比較的不是同一個東西。

### 修法

在 PromQL 對預測值加 `offset 5s`：

```promql
sum by (target)(nwdaf_predicted_ul_bytes{target="group=..."} offset 5s)
```

`offset 5s` 的意思是：「在 x 軸時間 T 上，顯示 T-5s 時的 Gauge 值」。

效果：T-5s 記錄的預測（針對 slot T 的預測）被顯示在 x 軸的 T 位置，和 ground truth for T 對齊。

```
x 軸時間 T 上顯示的：
  GT UL           = 實際 slot T 的上行流量
  Pred UL offset  = T-5s 時做的預測，目標就是 slot T    ← 正確比較
```

---

## Prometheus 的角色

### 為什麼需要 Prometheus，不直接用 MongoDB？

NWDAF 把 UPF 流量存在 MongoDB，但要用 MongoDB 畫曲線圖需要一個 exporter 中間層，而且 MongoDB 是應用程式自己的 DB，不適合給外部工具直接查。

Prometheus 是**專為 metrics 設計的**：
- 自帶 TSDB，查詢效率遠高於一般 DB
- Grafana 原生支援 Prometheus 資料來源
- PromQL 提供豐富的時間序列操作（offset、rate、sum、avg 等）
- pull-based 架構：不需要修改 5g-viz 推送邏輯，Prometheus 自己來拿

### Gauge vs Counter

| metric | 類型 | 原因 |
|---|---|---|
| `nwdaf_ground_truth_ul_bytes` | Gauge | 每個 slot 的量，可增可減 |
| `nwdaf_predicted_ul_bytes` | Gauge | 同上 |
| `nwdaf_retrain_total` | Counter | 只增不減，累計重訓次數 |

### 重啟清空

每次 `./start.sh` 啟動時，`grafana_setup.setup()` 會呼叫 Prometheus admin API 刪除所有 `nwdaf_*` series：

```
POST /api/v1/admin/tsdb/delete_series?match[]={__name__=~"nwdaf_.+"}
POST /api/v1/admin/tsdb/clean_tombstones
```

這樣每次重啟都是乾淨的起點，不會有前一次測試的殘留資料污染圖表。

---

## Grafana 的角色

### allow_embedding

瀏覽器的安全機制：預設情況下網頁無法把其他 origin 的頁面嵌入 `<iframe>`，因為 HTTP response 帶有：

```
X-Frame-Options: SAMEORIGIN
```

5g-viz 跑在 port 8765，Grafana 在 port 3000，port 不同就是不同 origin，所以需要在 Grafana 設定 `allow_embedding = true` 來移除這個限制。

注意：這只移除瀏覽器層面的 iframe 限制，Grafana 的登入驗證不受影響。

### Public Dashboard

Grafana dashboard 預設需要登入才能看，嵌入 iframe 後瀏覽器看到的是登入頁（黑畫面）。

解法：Grafana 10+ 的 Public Dashboard 功能，可以對特定 dashboard 產生一個無需登入的存取 token：

```
GET /public-dashboards/{accessToken}
```

5g-viz 啟動時把 token 存在記憶體，前端透過 `/api/grafana-token` 取得，再動態建立 iframe。

為什麼不存在 `.env`？因為 token 是在 dashboard 建立後才能取得（先有 dashboard 才有 token），而 dashboard 是 `./start.sh` 啟動時由 `grafana_setup.py` 建立的。每次重啟 5g-viz 都會重新取得 token（GET 既有的或 POST 建立新的），所以前端永遠拿得到有效 token，不需要手動管理。

### Dashboard 自動建立

`grafana_setup.setup()` 在每次 5g-viz 啟動時執行：

1. 查詢 Prometheus datasource UID（避免 hardcode，不同機器 UID 不同）
2. 根據 `GRAFANA_GROUPS` config 動態建立 panel（每個 group 一張圖，4 條線）
3. 取得 public dashboard token

這樣新增 group 只需改 `.env` 的 `GRAFANA_GROUPS` 再重啟即可。

---

## 啟動序列

```
./start.sh
  │
  ├─ 1. 找 prometheus binary（apt 優先，~/prometheus/ fallback）
  ├─ 2. pkill 現有 prometheus（確保乾淨啟動）
  ├─ 3. 啟動 prometheus --web.enable-admin-api（背景執行）
  ├─ 4. 等待 Prometheus /- /ready（最多 10s）
  └─ 5. uv run uvicorn main:app（前景執行）
           │
           └─ lifespan 啟動：
                ├─ grafana_setup.setup()
                │    ├─ 清空 Prometheus nwdaf_* 資料
                │    ├─ 查/建 Prometheus datasource
                │    ├─ 建立/更新 Grafana dashboard
                │    └─ 取得 public dashboard token
                ├─ asyncio task: _process_queue()（event 處理迴圈）
                └─ asyncio task: collector.start()（SSH tail 迴圈）
```

---

## WebSocket 與前端

瀏覽器連上 `/ws` 後：

1. 立即收到 `state_snapshot`（當前拓樸圖狀態，讓後連的使用者也能看到完整拓樸）
2. 持續接收即時 event（sbi_call、upf_volume、ml_inference 等）
3. 第一個 `aggregated_slot` event 觸發建立 Grafana iframe
4. WebSocket 斷線後 3s 自動重連

Grafana iframe 是整個 dashboard（不是單一 panel），因為 Public Dashboard 不支援 `d-solo`（單 panel 嵌入）。
