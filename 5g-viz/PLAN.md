# 5g-viz 文件整理計畫

本文件記錄 `docs/5g-viz/` 的整體文件規劃方向，供後續撰寫時參考。
文件結構以建議為主，撰寫時保有彈性，不強制規定每個子目錄的確切檔名。

---

## 核心原則：兩種文件性質的分離

目前 `design/` 混放了兩種性質不同的文件：

| 性質 | 說明 | 目標位置 |
|---|---|---|
| **設計文件（canonical）** | 描述系統現狀；隨程式碼演進持續維護；讀者是「想了解系統怎麼運作的人」 | `design/` |
| **實作規劃（plan）** | 為某功能規劃時所寫；紀錄當時的決策脈絡與設計討論；完成後不再主動更新 | `plans/`（新增） |

---

## 目標目錄結構

```
docs/5g-viz/
│
├── PLAN.md                        ← 本文件
│
├── design/                        ← canonical 設計文件（依系統層級分類）
│   │
│   ├── overview/                  ← 廣義層：讀完能理解整個系統
│   │   ├── system.md              ← 系統組件、技術選型、啟動方式
│   │   ├── data-flow.md           ← 端對端資料流（VM log → Prometheus → 瀏覽器）
│   │   └── event-schema.md        ← WebSocket event types + Prometheus metrics 對照表
│   │
│   ├── backend/                   ← 後端各組件
│   │   ├── collector.md           ← SSH tail、log 目錄偵測、容錯機制
│   │   ├── parser.md              ← base regex、rule 匹配流程、rules/ 模組系統
│   │   ├── state.md               ← NF 狀態追蹤、state_snapshot
│   │   ├── profiles.md            ← profile 系統（多環境切換）
│   │   ├── api.md                 ← 全部 HTTP endpoint 說明
│   │   └── metrics.md             ← Prometheus metric 定義與 dispatch 機制
│   │
│   ├── frontend/                  ← 前端各組件
│   │   ├── topology.md            ← Cytoscape.js 拓樸圖、config-driven 載入流程
│   │   ├── event-reactions.md     ← YAML-driven event → 動畫對應機制
│   │   └── filter.md              ← 節點／邊 filter sidebar
│   │
│   ├── grafana/                   ← Grafana + Prometheus 子系統
│   │   ├── setup.md               ← grafana_setup.py 行為、dashboard 結構
│   │   ├── embed.md               ← iframe 嵌入、kiosk mode、live/replay 切換
│   │   └── rendering.md           ← 時間窗渲染語意（右邊界、over-fetch 等）
│   │
│   ├── dvr/                       ← DVR 功能（跨越前後端）
│   │   ├── overview.md            ← 前端狀態機（LIVE/PAUSED/SCRUBBING/REPLAY）、模式定義
│   │   ├── session.md             ← session recording、JSONL 格式、生命週期
│   │   └── replay.md              ← backfill、pseudo-live pipeline、Prometheus 隔離
│   │
│   ├── features/                  ← Feature 視角：跨層的完整功能描述
│   │   ├── nwdaf-ml-cycle.md      ← NWDAF ML 監控週期（UPF volume → 推論 → retrain → 換模型）
│   │   ├── subscription-chain.md  ← 5G 訂閱鏈（Consumer→NWDAF→SMF→UPF→Consumer）
│   │   └── traffic-chart.md       ← 流量圖 pipeline（events → Prometheus → Grafana iframe）
│   │
│   └── reference/                 ← 查閱層
│       ├── topology-yaml.md       ← topology.yaml 完整 schema
│       └── env-config.md          ← profiles/{name}/.env 全部變數說明
│
├── plans/                         ← 實作規劃文件（現有 design/ 內容移入）
│   │                              ← 未來新規劃也依相同分類放入對應子目錄
│   ├── backend/
│   │   └── config-driven-refactor.md    ← 原 design/backend/event-reactions-and-metric-handlers.md
│   ├── frontend/
│   │   └── topology-config-and-filter.md ← 原 design/frontend/topology-config-and-filter.md
│   ├── grafana/
│   │   ├── interactive-embed.md         ← 原 design/grafana/grafana-interactive-embed.md
│   │   ├── chart-rendering.md           ← 原 design/grafana/grafana-chart-rendering.md
│   │   └── metric-event-modeling.md     ← 原 design/architecture/metric-event-modeling.md
│   └── dvr/                             ← 原 design/dvr/ 整個目錄移入
│       ├── README.md
│       ├── recording-and-prometheus.md
│       ├── frontend-and-api.md
│       ├── runtime-and-validation.md
│       ├── replay-pseudo-live.md
│       ├── roadmap.md
│       ├── replay-pseudo-live-consistency.md
│       └── replay-pseudo-live-consistency-debug.md
│
└── notes/                         ← 不變
    ├── meetings/
    ├── impl/
    │   ├── 2026-04-10-config-driven.md
    │   ├── 2026-04-10-impl.md
    │   └── improvement-history.md       ← 原 design/architecture/5g-viz-improvements.md 移入
    └── internals/
        └── 5g-viz-internals.md
```

---

## 現有文件的處置方式

### 移入 `plans/`（內容保留，調整位置與檔名）

| 現有路徑 | 移至 |
|---|---|
| `design/backend/event-reactions-and-metric-handlers.md` | `plans/backend/config-driven-refactor.md` |
| `design/frontend/topology-config-and-filter.md` | `plans/frontend/topology-config-and-filter.md` |
| `design/grafana/grafana-interactive-embed.md` | `plans/grafana/interactive-embed.md` |
| `design/grafana/grafana-chart-rendering.md` | `plans/grafana/chart-rendering.md` |
| `design/architecture/metric-event-modeling.md` | `plans/grafana/metric-event-modeling.md` |
| `design/dvr/`（整個目錄） | `plans/dvr/` |

### 移入 `notes/impl/`

| 現有路徑 | 移至 |
|---|---|
| `design/architecture/5g-viz-improvements.md` | `notes/impl/improvement-history.md` |

### 移入 `docs/archive/`（被新 `design/` 文件取代，保留備查）

| 現有路徑 | 說明 |
|---|---|
| `design/architecture/5g-viz-architecture.md` | 由 `design/overview/` 取代 |
| `design/architecture/visualizer-plan.md` | 由 `design/overview/` 取代 |

---

## 撰寫 `design/` 文件時的注意事項

**以程式碼為準**：撰寫前先讀目前的程式碼，`plans/` 裡的對應文件只作為背景參考。

**以下 `plans/` 文件與各 `design/` 文件有直接對應關係，可供參考：**

| 目標文件 | 可參考的 plans/ 文件 |
|---|---|
| `design/backend/profiles.md` | `plans/backend/config-driven-refactor.md` §A |
| `design/backend/metrics.md` | `plans/backend/config-driven-refactor.md` §C |
| `design/frontend/event-reactions.md` | `plans/backend/config-driven-refactor.md` §B |
| `design/frontend/topology.md` | `plans/frontend/topology-config-and-filter.md` |
| `design/frontend/filter.md` | `plans/frontend/topology-config-and-filter.md` |
| `design/grafana/embed.md` | `plans/grafana/interactive-embed.md` |
| `design/grafana/rendering.md` | `plans/grafana/chart-rendering.md` |
| `design/dvr/` | `plans/dvr/` |

**語氣**：canonical 文件以現在式描述系統現況（「X 負責 Y」），不使用規劃語氣（「X 將會負責 Y」）。

**`features/` 文件的寫法**：以功能為視角貫穿多個系統層，說明「這個功能從頭到尾怎麼流」，不重複描述各 component 的內部細節（讀者可自行查閱對應的 backend/frontend/grafana 文件）。

---

## `plans/` 未來使用慣例

新增實作規劃時，依功能所屬的系統層級放入對應子目錄：

```
plans/
├── backend/    ← 後端邏輯、pipeline、API 相關規劃
├── frontend/   ← 前端 UI、拓樸圖、互動行為規劃
├── grafana/    ← Grafana/Prometheus 整合相關規劃
└── dvr/        ← DVR 功能系列規劃
```

文件命名以 feature 名稱為主，不帶日期（git history 有時間戳記）。
