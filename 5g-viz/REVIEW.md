# 5g-viz 文件審核報告

審核日期：2026-04-15  
審核方式：對照 `5g-viz/` 程式碼逐份比對  
涵蓋範圍：`design/` 下全部 22 份文件

---

## 一、需修正的正確性問題

### `reference/env-config.md` + `backend/profiles.md`：PROMETHEUS_URL（高優先）

`.env.example` 定義的是 `PROMETHEUS_BASE`，但程式碼 `main.py:310` 讀的是 `PROMETHEUS_URL`。
兩份文件都有提到，但說明不夠醒目。  
**修法**：在 `env-config.md` 第 1 節加粗警告，`profiles.md` 補交叉引用。

---

### `grafana/rendering.md`：offset 5s 方向說法不精確

文件說「把預測曲線向左對齊一個 slot」，但 PromQL `offset 5s` 的實際語意是查詢**過去 5 秒前的值**，物理意義是時間戳往前移，不是「向左」。  
**修法**：改成「因為 inference 比 ground truth 晚 5 秒產生，用 offset 5s 把預測的時間戳向前移，讓兩條線視覺對齊」，並加時間軸圖示：

```
Slot N:       Ground truth 記錄於 T
Slot N+1:     Inference 於 T+5s 產生，預測的是 T+5s 的流量

顯示時用 offset 5s：
query(T+5s, offset 5s) → 實際查 T 時點的值 → 與 ground truth 對齊
```

---

### `grafana/setup.md`：setup.sh 責任描述有誤

文件說會「檢查本機或目標環境」，但 `setup.sh` 只檢查本機（Grafana、Prometheus binary），不檢查遠端 VM。  
另：`prometheus.yml` 複製行為是無條件覆寫，不是「若不存在才複製」。

---

### `overview/system.md`：replay backfill 預設行為

文件隱含 backfill 可選，但 `main.py:357-367` 的邏輯是：**預設執行**，只有 `FORCE_BACKFILL=0` 且 Prometheus 已有該 session 資料才跳過。

---

### `backend/metrics.md`：函數名稱說明不清

文件說「`set_metric_session_id()`」，實際 `rules/nwdaf.py:21` 定義的是 `set_session_id()`。前者是 `rules/__init__.py` 動態封裝後的名稱。兩者都正確，但應說明封裝關係。

---

### `frontend/events-and-dvr.md`：event log 兩套邏輯混淆

文件說「最多保留 50 行」，但實際有兩套邏輯：
- live append 模式：超過 200 筆才刪（`events.js:664`）
- static tail（scrub/paused）：取最近 31 筆（`LOG_TAIL_SIZE = 31`）

應分開說明。

---

## 二、重要缺漏

### Parser 執行順序（`backend/parser.md` + `overview/data-flow.md`）

規則載入順序由 `pkgutil.iter_modules()` 的字母序決定（`nwdaf.py` → `nwdaf_sub.py` → `smf.py`），兩份文件均未提及。這對理解「為什麼特定 log 被哪條規則匹配」是關鍵資訊。

---

### 哪些 event 有 metric handler（`overview/event-schema.md` + `features/nwdaf-ml-cycle.md`）

目前只有 5 種 event 進入 Prometheus：`aggregated_slot`, `ml_inference`, `accuracy`, `retrain_trigger`, `model_swap`。其他（如 `upf_volume`, `smf_sub_confirmed`）只存 event log。

特別注意：`aggregated_slot` 的 `ul_thr`/`dl_thr` 欄位被 parser 提取但沒有對應 handler（靜默丟棄）。  
**建議**：`event-schema.md` 的事件型別表加「是否有 metric handler」欄。

---

### Replay pre-seed 計算公式（`dvr/replay.md`）

文件說「前一段視窗內的 metric 映射到現在之前」，但缺少公式：
- 範圍：`window_ms * speed`
- 時間映射：`now_ms - int((from_ms - event_ms) / speed)`
- 定位方式：`bisect_right()` 二分搜尋（不是線性掃描）

---

### Remote write 的 protobuf + snappy（`overview/data-flow.md`）

replay backfill 的 payload 需要 snappy 壓縮（`main.py:376-404`），這解釋了 `python-snappy` 依賴的用途，文件完全沒提。

---

### Replay 時 Prometheus 清空時機（`dvr/replay.md`）

文件說「清空 managed Prometheus TSDB」，但沒說這是在 `start.sh` 層執行，不是 FastAPI app 內部。

---

### `reference/topology-yaml.md`：replay 使用哪個 topology.yaml

replay 讀的是 **session 目錄內保存**的 `topology.yaml`，不是目前 profile 的。文件有提但不夠醒目。影響：修改 profile 後，舊 session 的重播行為不受影響（按錄製當時的拓樸）。

---

### 環境設定層缺漏

| 問題 | 位置 |
|---|---|
| `GRAFANA_PUBLIC_TOKEN` 出現在 `profiles/default/.env` 但文件未記載、程式碼也未使用（遺留變數） | `env-config.md` |
| `WS_PORT` 在 `config.py` 有定義，但 `start.sh` hardcoded `--port 8765`，兩者脫鉤，應在 `start.sh` 改用 `$WS_PORT` | `profiles.md` + `env-config.md` |
| `.env` 變數名稱被 `topology.yaml` 的 `host_env` 等欄位引用，拼字錯誤只會在 runtime 才發現，文件未說明此風險 | `topology-yaml.md` + `env-config.md` |

---

### Frontend 缺漏

| 文件 | 缺漏 |
|---|---|
| `frontend/topology.md` | Panel resizer 拖曳互動（`topology.js:97-170`）完全未提；profile badge 顯示未提 |
| `frontend/grafana-embed.md` | `kiosk` 參數代表隱藏 Grafana 頂部導覽列未說明；`refresh=5s` 只在 LIVE 和 PSEUDO_LIVE 兩種狀態下啟用，其他狀態為 `off` 未說明 |
| `frontend/events-and-dvr.md` | `_historyFetchPromise` 鎖機制（避免重複 backfill 請求）未提 |

---

### DVR 缺漏

| 文件 | 缺漏 |
|---|---|
| `dvr/overview.md` | state.py 在各層責任分工表中未列出 |
| `dvr/session.md` | `_session_events_cache` 無 eviction 機制未說明；events.jsonl 的排序保證未說明 |
| `dvr/replay.md` | `_retrain_prefix` 結構（O(1) retrain 總數查詢）未提 |

---

## 三、清晰度改善機會

### 最高優先（對理解幫助最大）

**`backend/parser.md`**：加完整 log 解析範例（輸入 → BASE dict → rule 匹配 → event dict）

```
輸入 log:
  time="2026-04-15T06:33:30Z" level="info" msg="latest aggregated slot: sub-001 group=group-test-001 ..."

BASE regex 結果:
  { "time": "...", "level": "info", "msg": "latest aggregated slot: ...", "nf": null, "cat": null,
    "extra_kv": {"sub": "sub-001", "group": "group-test-001", ...} }

匹配 rules/nwdaf.py 的 aggregated_slot 規則 → 產生 event:
  { "type": "aggregated_slot", "sub_id": "sub-001", "target": "group-test-001", ... }
```

**`frontend/events-and-dvr.md`**：加狀態機遷移圖

```
LIVE ──[pause]──→ PAUSED ──[play]──→ PLAYING ──[reach end / go-live]──→ LIVE / PAUSED
                    ↑                     ↓
                    └─────[pause]─────────┘
                    ↓
              [drag timeline]──→ SCRUBBING ──[release]──→ PAUSED
```

**`overview/data-flow.md`**：加 event-to-metric 對應表格

| Event Type | 寫入 Metrics | Labels |
|---|---|---|
| `aggregated_slot` | `nwdaf_ground_truth_ul_bytes`, `nwdaf_ground_truth_dl_bytes` | session, sub_id, target |
| `ml_inference` | `nwdaf_predicted_ul_bytes`, `nwdaf_predicted_dl_bytes` | session, sub_id, target |
| `accuracy` | `nwdaf_deviation` | session, model |
| `retrain_trigger` | `nwdaf_retrain_total`（counter +1） | session |
| `model_swap` | 刪除舊 model 的 deviation series | — |
| 其他 | 不進 Prometheus，只存 event log | — |

---

### 各文件補充建議

#### `backend/`

| 文件 | 建議 |
|---|---|
| `collector.md` | 加最新子目錄切換的 log 範例：`[collector] smf: /logs changed: dir-A → dir-B, reconnecting` |
| `state.md` | 加 `_state` 資料結構範例；加 `nf_aliases` 轉換範例（`"SMF"` → `"smf"`）；加模板解析範例（`{nf}` 替換流程） |
| `api.md` | 加 curl 範例（`/api/events` 的 `from`/`to` 參數）；說明 default limit = 50000 |
| `profiles.md` | 環境變數表格加「使用位置」欄（config.py / main.py / rules/） |
| `metrics.md` | metric 表格加「值單位」欄（bytes 等）；加 Prometheus query 範例 |

#### `frontend/`

| 文件 | 建議 |
|---|---|
| `topology.md` | 加 event → action → UI 流程圖（`{from}` 模板替換 → `nf_aliases` 轉換 → Cytoscape animation）；說明 `EDGE_COLLAPSE_THRESHOLD = 2`（同一邊第 3 條起折疊） |
| `grafana-embed.md` | 加 URL 參數表格（orgId / kiosk / theme / var-session / refresh / from / to）；補充 chart window 預設 3 分鐘、範圍 1–15 分鐘 |

#### `grafana/`

| 文件 | 建議 |
|---|---|
| `setup.md` | 補充 `prometheus.yml` 的兩個 scrape job（`prometheus` + `5g-viz`），以及 `5g-viz` 的 interval = 5s |
| `rendering.md` | 加 deviation panel 多 model 時 `topk` 只保留最新 model 的說明 |

#### `overview/`

| 文件 | 建議 |
|---|---|
| `system.md` | 補 HTTP vs WebSocket 使用場景表（live 用 WS 推播，replay 用 HTTP batch 讀取）；補 session 目錄三個檔案用途 |
| `data-flow.md` | 補 backfill pre-check 邏輯：`FORCE_BACKFILL=0` 時先 query Prometheus，有資料才跳過 |
| `event-schema.md` | 加「session label 是全域注入，live 啟動後不可切換」的限制說明 |

#### `dvr/`

| 文件 | 建議 |
|---|---|
| `replay.md` | 補 pre-seed 計算公式；說明「改 chart window 為何要重啟 pseudo-live」（pre-seed 範圍依賴 window 大小） |

#### `reference/`

| 文件 | 建議 |
|---|---|
| `env-config.md` | 加粗 `PROMETHEUS_URL` 警告；補 `GRAFANA_PUBLIC_TOKEN` 遺留說明；補「replay 時哪些變數不被讀取（SSH 相關）」 |
| `topology-yaml.md` | 補 `event_reactions` 前後端支援差異（backend `state.py` 只用 `add_class`/`remove_class`，frontend 支援所有 action）；補 compound node 子節點約束（parent 必須指向已存在的 node） |

---

## 四、各文件整體評估

| 文件 | 正確性 | 缺漏程度 | 清晰度 |
|---|---|---|---|
| `backend/collector.md` | 良好，細節有小誤 | 少數 | 需 log 範例 |
| `backend/parser.md` | 良好 | 規則順序缺 | **強烈建議加解析範例** |
| `backend/state.md` | 良好 | live/replay 差異缺 | 需資料結構範例 |
| `backend/api.md` | 良好 | limit 值缺 | 需 curl 範例 |
| `backend/profiles.md` | **PROMETHEUS_URL 問題** | WS_PORT 脫鉤缺 | 需環境變數表格 |
| `backend/metrics.md` | 函數名稱不清 | 單位 / handler 列表缺 | 需 query 範例 |
| `frontend/topology.md` | 良好 | resizer 等 UI 行為缺 | 需流程圖 / YAML 範例 |
| `frontend/events-and-dvr.md` | event log 兩套邏輯混淆 | 鎖機制缺 | **強烈建議加狀態機圖** |
| `frontend/grafana-embed.md` | 條件判斷未說明 | kiosk / refresh 條件缺 | 需 URL 參數表格 |
| `grafana/setup.md` | setup.sh 描述有誤 | prometheus.yml 內容缺 | 良好 |
| `grafana/rendering.md` | **offset 方向不精確** | 良好 | 需時間軸圖示 |
| `overview/system.md` | backfill 預設描述 | WebSocket/HTTP 場景缺 | 需場景表 |
| `overview/data-flow.md` | protobuf/snappy 未提 | parser 順序 / 公式缺 | **強烈建議加 log→event 範例** |
| `overview/event-schema.md` | 良好 | handler 列表缺 | 需 handler 對照表 |
| `dvr/overview.md` | live/replay 描述混用 | state.py 責任缺 | 良好 |
| `dvr/session.md` | 良好 | 快取機制缺 | 良好 |
| `dvr/replay.md` | 清空時機不清 | pre-seed 公式缺 | 需計算範例 |
| `features/nwdaf-ml-cycle.md` | offset 理由缺 | handler 列表缺 | 良好 |
| `features/subscription-chain.md` | 良好 | UPF 映射粒度缺 | 良好 |
| `features/traffic-chart.md` | session variable 限制 | GRAFANA_GROUPS 格式缺 | 良好 |
| `reference/env-config.md` | **PROMETHEUS_URL 高風險** | GRAFANA_PUBLIC_TOKEN 缺 | 需警告加粗 |
| `reference/topology-yaml.md` | 良好 | compound 約束 / replay 版本缺 | 需邊際情況說明 |
