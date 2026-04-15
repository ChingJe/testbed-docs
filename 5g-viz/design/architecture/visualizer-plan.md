# 5G Testbed 視覺化系統規劃

## 目標

即時視覺化 testbed 運作狀態，不修改任何元件原始碼，只讀取 log。

**觀察重點：**
- SMF：訂閱建立流程（SMF → UPF PFCP、SMF → NWDAF Nsmf Subscribe）
- NWDAF：ML inference 輸出、accuracy monitor、retrain 流程
- 流量曲線：每個 group 的 UPF 實測值 vs NWDAF 預測值（UL / DL）

---

## 技術選型

| 層 | 選型 | 理由 |
|---|---|---|
| 後端 | Python FastAPI | async/await 同時處理多條 SSH log stream，內建 WebSocket |
| SSH log streaming | asyncssh | 非同步 SSH，不 block 主迴圈 |
| 前端框架 | Vanilla JS（無 build step） | 直接開 HTML，lab 環境友好 |
| NF 拓樸圖 | Cytoscape.js（CDN） | 有 animate API，支援 pulse / edge 動畫 |
| 曲線圖 | Prometheus + Grafana | 維護性佳，Grafana 已在系統跑著 |
| 前端通訊 | 瀏覽器原生 WebSocket | 不需套件 |
| 套件管理 | uv | 管理 Python 虛擬環境與依賴 |

**Prometheus metrics 暴露：**
- 5g-viz `main.py` 透過 `prometheus_client` 在 `/metrics` 暴露 gauge / counter
- Prometheus（`~/prometheus/`）每 5 秒 scrape `localhost:8765/metrics`
- Grafana 嵌入前端頁面底部（iframe，public dashboard），每 group 一張圖

---

## 架構

```
Host machine
├── 5g-viz/
│   ├── main.py          # FastAPI + WebSocket + /metrics + /api/grafana-token
│   ├── collector.py     # asyncssh tail 5GC log，emit raw lines
│   ├── parser.py        # 載入 rules/，逐行比對，產出 Event dict
│   ├── state.py         # 維護全域狀態（NF up/down、IP↔SUPI mapping）
│   ├── config.py        # SSH 設定、IP mapping、log 路徑、Grafana config（讀 .env）
│   ├── grafana_setup.py # 啟動時自動建 datasource/dashboard/public token
│   ├── prometheus.yml   # Prometheus config 模板（setup.sh 複製到 ~/prometheus/）
│   ├── setup.sh         # 第一次部署用
│   ├── start.sh         # 每次啟動用（同時管理 Prometheus）
│   ├── rules/
│   │   ├── smf.py       # SMF：nf_up、PFCP assoc、Nsmf Subscribe
│   │   ├── nwdaf.py     # NWDAF：upf_volume、ml_inference、accuracy、retrain、aggregated_slot
│   │   ├── nwdaf_sub.py # 訂閱鏈：Consumer↔NWDAF↔SMF, UPF Notify
│   │   └── __init__.py  # 自動掃描 rules/ 下所有 module，彙整 rule list
│   └── frontend/
│       ├── index.html   # 主頁（viewport 佔滿視窗，Grafana iframe 在下方）
│       ├── topology.js  # Cytoscape.js NF 拓樸圖
│       └── events.js    # WebSocket 接收 + dispatch，首個 aggregated_slot 建立 iframe
├── ~/prometheus/        # Prometheus（二進位或 apt），--web.enable-admin-api
└── Grafana              # 系統已安裝（v11.6.1），port 3000，allow_embedding = true
```

資料流：
```
VM log file
  → asyncssh tail -F（5GC only）
    → parser.py（regex → Event）
      ├── state.py 更新狀態
      ├── WebSocket broadcast → 前端拓樸圖
      └── Prometheus gauge/counter 更新
            → Prometheus scrape /metrics（每 5s）
              → Grafana dashboard（每 group 一張圖，iframe 嵌入前端）
```

---

## Log 來源

| 來源 | VM | 路徑 | 說明 |
|---|---|---|---|
| free5gc.log | 5GC | `~/free5gc/log/<latest>/free5gc.log` | SMF 訂閱建立等事件 |
| nwdaf.log | 5GC | `~/nwdaf.log` | NWDAF 所有分析輸出 |

**注意：UPF-EES / UPF-EES2 的 upf.log 不收集。**
UPF 使用 zap logger 輸出到 stderr，不進 upf.log。UPF 相關流量資料改由 NWDAF log 取得。

---

## Log 格式

所有 log 統一格式（logrus key-value）：
```
time="2026-03-11T09:23:07.122Z" level="info" msg="..." CAT="AnLF" NF="NWDAF"
```

基礎 regex（適用所有 NF）：
```python
BASE = re.compile(
    r'time="(?P<time>[^"]+)"'
    r'\s+level="(?P<level>[^"]+)"'
    r'\s+msg="(?P<msg>[^"]+)"'
    r'(?:\s+CAT="(?P<cat>[^"]+)")?'
    r'(?:\s+NF="(?P<nf>[^"]+)")?'
    r'(?P<extra>.*)'
)
KV = re.compile(r'(\w+)="([^"]*)"')
```

---

## Parser Rules

### Rule 格式

```python
{
    "match": {
        "nf": "SMF",               # 可省略
        "cat": "Proc",             # 可省略
        "msg": re.compile(r"..."), # 可省略
    },
    "event": "sbi_call",
    "build": lambda m, base: { ... },
}
```

### free5gc.log

#### SMF 啟動
```
msg="Start SBI server (listen on 127.0.0.2:8000)"  NF="SMF"
```
→ Event: `{ type: "nf_up", nf: "SMF" }`

#### PFCP Association（SMF ↔ UPF）
```
msg="Sending PFCP Association Request to UPF[192.168.125.10]"  NF="SMF"
```
→ Event: `{ type: "sbi_call", from: "SMF", to: "UPF-EES", label: "PFCP Assoc" }`

#### Nsmf Subscribe（NWDAF → SMF）
```
msg="Nsmf subscription created"  NF="SMF"  supi="imsi-..."  notif_id="corr-1"
```
→ Event: `{ type: "sbi_call", from: "NWDAF", to: "SMF", label: "Nsmf Subscribe" }`

### nwdaf.log

#### UPF Volume Report（每 5 秒，per IP）
```
msg="UPF VOLUME: ip=10.10.0.2, startTime=..., total=N, ul=N, dl=N, ..."
NF="NWDAF"
```
→ Event: `{ type: "upf_volume", upf: "UPF-EES", ip: "10.10.0.2", ul_bytes: N, dl_bytes: N }`

#### Aggregated Slot（每 5 秒，per subscription）
```
msg="latest aggregated slot: <sub_id> <target> ts=<ts>
     ulVol=N dlVol=N totalVol=N
     ulPkts=N dlPkts=N totalPkts=N
     ulThr=N.NNNN dlThr=N.NNNN ulPktThr=N.NNNN dlPktThr=N.NNNN"
CAT="AnLF"  NF="NWDAF"
```
`target` 格式：`group=<groupId>`、`supi=<supi>`、或 `supis=[s1,s2,...]`

→ Event: `{ type: "aggregated_slot", sub_id, target, ts, ul_vol, dl_vol, ul_thr, dl_thr }`
→ **更新 Prometheus ground truth gauge**

#### ML Inference（每 5 秒）
```
msg="ML inference: sub=X group=group-test-001 steps=1 ulVol=N dlVol=N confidence=80 commDur=5s"
CAT="AnLF"  NF="NWDAF"
```
→ Event: `{ type: "ml_inference", sub_id, target, ul_vol, dl_vol, confidence }`
→ **更新 Prometheus predicted gauge**

#### Accuracy Monitor（每 ~50 秒）
```
msg="Accuracy [model_url]: deviation=0.1234, accuracy=88%, samples=10"
CAT="AnLF"  NF="NWDAF"
```
→ Event: `{ type: "accuracy", model, deviation, accuracy, samples }`

#### MTLF Retrain
```
msg="Triggering retraining due to accuracy degradation for model: ..."  CAT="MTLF"
msg="Accuracy-triggered retraining completed successfully"              CAT="MTLF"
msg="Model hot-swap completed successfully: new modelId=..."            CAT="MTLF"
```
→ Events: `retrain_trigger` / `retrain_done` / `{ type: "model_swap", model_id }`
→ `retrain_trigger` **遞增 Prometheus counter**（供 Grafana annotation 使用）

#### Nnwdaf Subscribe / Notify（訂閱鏈）
```
msg="Consumer subscribed"    → sbi_call: Consumer → NWDAF
msg="Nnwdaf Notify sent"    → sbi_call: NWDAF → Consumer
msg="UPF Notify received"   → sbi_call: UPF → NWDAF
```
（詳見 `rules/nwdaf_sub.py`）

---

## Prometheus Metrics

`main.py` 透過 `prometheus_client` 維護，掛載到 `/metrics`：

| Metric | Type | Labels | 來源 event |
|---|---|---|---|
| `nwdaf_ground_truth_ul_bytes` | Gauge | `sub_id`, `target` | `aggregated_slot` |
| `nwdaf_ground_truth_dl_bytes` | Gauge | `sub_id`, `target` | `aggregated_slot` |
| `nwdaf_predicted_ul_bytes` | Gauge | `sub_id`, `target` | `ml_inference` |
| `nwdaf_predicted_dl_bytes` | Gauge | `sub_id`, `target` | `ml_inference` |
| `nwdaf_retrain_total` | Counter | — | `retrain_trigger` |
| `nwdaf_deviation` | Gauge | `model` | `accuracy` |

**注意：** Grafana 查詢預測值時加 `offset 5s`，讓預測線對齊 ground truth 的相同時間窗口（預測是 steps=1，針對下一個 slot）。

---

## Grafana Dashboard

由 `grafana_setup.py` 在每次 `./start.sh` 啟動時自動建立/更新：

1. 查詢或建立 Prometheus datasource（不 hardcode UID）
2. 依 `GRAFANA_GROUPS` config 建立 panel（每個 group 一張圖）
3. 取得 public dashboard token（GET 既有或 POST 新建）

每張圖 4 條線：
```promql
sum by (target)(nwdaf_ground_truth_ul_bytes{target="group=..."})          # GT UL
sum by (target)(nwdaf_ground_truth_dl_bytes{target="group=..."})          # GT DL
sum by (target)(nwdaf_predicted_ul_bytes{target="group=..."} offset 5s)   # Pred UL（對齊）
sum by (target)(nwdaf_predicted_dl_bytes{target="group=..."} offset 5s)   # Pred DL（對齊）
```

Retrain annotation：
```promql
changes(nwdaf_retrain_total[1m]) > 0
```

前端嵌入：首個 `aggregated_slot` event 到達時，前端 fetch `/api/grafana-token` 取得 token 後動態建立 iframe。

---

## WebSocket Event Schema

後端 → 前端，統一格式：
```json
{ "type": "event_type", "time": "2026-03-11T09:23:07Z", ...payload }
```

| type | 用途 | 主要欄位 |
|---|---|---|
| `nf_up` | SMF 啟動 | `nf` |
| `sbi_call` | NF 間通訊箭頭 | `from`, `to`, `label` |
| `upf_volume` | UPF 流量（per IP，拓樸圖用） | `upf`, `ip`, `ul_bytes`, `dl_bytes` |
| `aggregated_slot` | per-subscription 聚合流量（Prometheus 用） | `sub_id`, `target`, `ul_vol`, `dl_vol` |
| `ml_inference` | NWDAF 推論輸出 | `sub_id`, `target`, `ul_vol`, `dl_vol`, `confidence` |
| `accuracy` | Accuracy monitor check | `model`, `deviation`, `accuracy`, `samples` |
| `retrain_trigger` | 觸發重訓 | — |
| `retrain_done` | 重訓完成 | — |
| `model_swap` | 模型熱更換 | `model_id` |
| `smf_sub_confirmed` | SMF 訂閱建立確認 | — |
| `state_snapshot` | 新 client 連線全量狀態 | `nf_status` |

---

## 前端拓樸圖（Cytoscape.js）

### 節點

| 節點 ID | 顯示名稱 | 備註 |
|---|---|---|
| `consumer` | Consumer | 訂閱發起方 |
| `nwdaf` | NWDAF | compound，包含 AnLF / MTLF |
| `nwdaf_anlf` | AnLF | NWDAF 子節點 |
| `nwdaf_mtlf` | MTLF | NWDAF 子節點 |
| `smf` | SMF | |
| `upf_ees` | UPF-EES | |
| `upf_ees2` | UPF-EES2 | |

gNB 不顯示。AccMon 不是獨立節點（AnLF → MTLF 的行為）。

### Edge 動畫

| 類型 | 顏色 | 持續 |
|---|---|---|
| Nnwdaf Subscribe / Notify | `#ce93d8`（紫） | 1200ms |
| Nsmf Subscribe | `#ce93d8`（紫） | 1200ms |
| PFCP Assoc | `#4fc3f7`（藍） | 1000ms |
| UPF Notify | `#80cbc4`（青綠） | 800ms |
| AnLF → MTLF accuracy | `#00e5ff`（青，細） | 800ms |
| MTLF → AnLF provision | `#a5d6a7`（綠） | 4000ms |

---

## 設定（.env）

```
SSH_5GC_HOST=127.0.0.1
SSH_5GC_PORT=2222
SSH_5GC_KEY=/path/to/private_key

LOG_FREE5GC_DIR=~/free5gc/log
LOG_NWDAF=~/nwdaf.log

UPF_EES_API_IPS=192.168.127.10=UPF-EES,192.168.127.11=UPF-EES2
UPF_DATA_SUBNETS=10=UPF-EES,100=UPF-EES2

GRAFANA_BASE=http://<host-ip>:3000
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASS=admin
GRAFANA_GROUPS=group-test-001,group-test-002
GRAFANA_PUBLIC_TOKEN=        # 啟動時自動填入
PROMETHEUS_BASE=http://localhost:9090
```

---

## 實作 Phases

### ✅ Phase 1：骨架（後端 + WebSocket + 基本前端）
- SSH tail、parser、WebSocket broadcast、前端基礎

### ✅ Phase 2：NF 拓樸圖
- Cytoscape.js 節點佈局、edge flash、node pulse、持久訂閱狀態
- 完整訂閱鏈：Consumer → NWDAF → SMF → UPF → NWDAF → Consumer
- retrain / model_swap / accuracy 動畫

### ✅ Phase 3：Prometheus + Grafana 曲線圖
- `/metrics` endpoint，5 個 metrics（4 Gauge + 1 Counter）
- `grafana_setup.py`：啟動時自動建 datasource / dashboard / public token
- Grafana iframe 嵌入前端，token 由後端動態提供
- `start.sh`：同時管理 Prometheus（含 admin API）與 5g-viz
- `setup.sh`：第一次部署精靈
- 預測線 `offset 5s` 對齊 ground truth 相同時間窗口

### 🔲 Phase 4：Accuracy / Deviation 視覺化

**設計決策：**
- `accuracy` event 是 model 層級（兩個 group 共用同一組 model），無須對應 group
- 只顯示 `deviation`（越高越差），不顯示 accuracy %（簡化 y-axis）
- 新增一張 deviation panel，與現有 group 流量圖**並排在同一排**

**Grafana 佈局（以 2 groups 為例）：**
```
[group-test-001 流量, w=8, x=0]  [group-test-002 流量, w=8, x=8]  [Deviation, w=8, x=16]
```
panel 寬度動態計算：`panel_width = 24 // (len(GRAFANA_GROUPS) + 1)`

**新增 Prometheus metric：**

| Metric | Type | Labels | 來源 event |
|---|---|---|---|
| `nwdaf_deviation` | Gauge | `model` | `accuracy` |

**Grafana deviation panel PromQL：**
```promql
nwdaf_deviation   # 所有 model，legend = {{model}}
```

**實作範圍：**

1. **`main.py`**
   - 新增 `_deviation = Gauge("nwdaf_deviation", ..., ["model"])`
   - `_update_metrics()` 處理 `accuracy` event：`_deviation.labels(model=...).set(deviation)`

2. **`grafana_setup.py`**
   - `_build_panel()` 的 `w` 和 `x` 改用 `panel_width` 參數（由呼叫端傳入）
   - 新增 `_build_deviation_panel(panel_id, x, w, datasource_uid)`
   - `setup()` 中計算 `panel_width = 24 // (len(GRAFANA_GROUPS) + 1)`，並在 panels list 末尾加入 deviation panel

### 🔲 Phase 5（待定）
- Nnwdaf Notify group 標示（等 NWDAF log 加上 group 資訊後）

---

## 已知問題 / 限制

1. **Nnwdaf Notify 無法區分 group**：等 NWDAF log 加入 group 資訊後再處理。
2. **Prometheus scrape 週期誤差**：UPF 每 5 秒回報，Prometheus scrape interval 也設 5 秒，誤差最多 5 秒。
