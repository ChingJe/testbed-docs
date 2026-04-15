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
│   ├── architecture/                    ← 跨層 / cross-cutting 規劃
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
| `design/architecture/metric-event-modeling.md` | `plans/architecture/metric-event-modeling.md` |
| `design/dvr/`（整個目錄） | `plans/dvr/` |

### 移入 `notes/impl/`

| 現有路徑 | 移至 |
|---|---|
| `design/architecture/5g-viz-improvements.md` | `notes/impl/improvement-history.md` |

### 移入 `docs/archive/5g-viz/`（被新 `design/` 文件取代，保留備查）

| 現有路徑 | 說明 |
|---|---|
| `design/architecture/5g-viz-architecture.md` | 由 `design/overview/` 取代 |
| `design/architecture/visualizer-plan.md` | 由 `design/overview/` 取代 |

---

## 遷移時一併處理的同步項目

### 1. 修正 Markdown 相對連結

只搬檔案位置不夠，凡是文件內有相對連結者，都要在遷移時一併改寫。
特別是 `design/dvr/` 與 `design/grafana/` 目前已互相引用；搬到 `plans/` 後，原本的 `../` 或同層路徑很可能失效。

**原則：**

- 同一主題目錄內互連，優先維持同層相對路徑。
- 跨目錄引用時，以搬遷後的新目錄結構重算路徑。
- 若某文件已被新的 canonical 文件取代，舊文件不再指向已不存在的舊位置；必要時改指向新 `design/` 文件或保留純文字說明。

### 2. 更新索引文件

`docs/5g-viz/` 結構調整後，至少要同步更新下列索引入口：

- `docs/README.md`：更新 `5g-viz/` 的目錄說明，避免仍描述舊的 `design/architecture|backend|frontend|grafana|dvr` 結構。
- `docs/5g-viz/PLAN.md`：本文件在結構異動後持續保持與實際目錄一致。
- 若新增 `design/README.md`、`plans/README.md` 或其他導覽頁，也應在同一次整理中補上。

### 3. 保留 archive 來源脈絡

為避免不同子系統的 retired 文件混放在同一層，`5g-viz` 的 archive 檔案統一放在：

```text
docs/archive/5g-viz/
```

若後續 archive 內容變多，可再依性質細分為：

```text
docs/archive/5g-viz/design/
docs/archive/5g-viz/plans/
```

### 4. 明確區分 cross-layer 文件

部分文件雖與某個子系統關係密切，但本質上是跨層議題，不宜硬塞進單一子目錄。
例如：

- metrics event modeling
- event schema
- live / replay 共用的資料語意

這類規劃文件應優先放在 `plans/architecture/`；canonical 文件則依內容拆到 `design/overview/`、`design/backend/`、`design/grafana/` 等最終落點。

### 5. `event_reactions` 視為 cross-layer config

`event_reactions` 雖直接驅動前端動畫，但也會影響後端 state snapshot 的生成，因此在文件上不應只被視為前端細節。

建議寫法：

- `design/frontend/event-reactions.md`：描述前端如何執行 reaction actions、如何對應到 Cytoscape 視覺效果。
- `design/reference/topology-yaml.md`：定義 `event_reactions` schema、模板欄位、action 類型。
- `design/backend/state.md`：補充後端如何使用同一份 reaction config 來維護 `state_snapshot`。

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
| `design/overview/event-schema.md` | `plans/architecture/metric-event-modeling.md` |
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
├── architecture/ ← 跨層資料模型、event schema、系統級設計決策
└── dvr/        ← DVR 功能系列規劃
```

文件命名以 feature 名稱為主，不帶日期（git history 有時間戳記）。

---

## 建議執行順序

若目標是**先把舊規劃文件與未來 canonical 文件明確分開**，可以把「先搬現有 `design/`」作為第一步。
這樣在撰寫階段會更容易辨識哪些是舊規劃、哪些是新設計文件。

但要注意：此作法會讓整理過程中出現一段「新 `design/` 尚未補齊」的過渡期，因此需要同時管理好索引與連結修正。

### 建議方案 A：先整體搬移，再重建 `design/`

這個方案適合本次情境，因為目前 `design/` 內本來就混有大量 plan 性質文件。

1. **先將現有 `design/` 內容搬到 `plans/`**
   - 依本文件上方的對應表搬移
   - `design/dvr/` 直接移到 `plans/dvr/`
   - `design/architecture/metric-event-modeling.md` 移到 `plans/architecture/`
   - 被新 canonical 文件完全取代的總覽文件，改移到 `docs/archive/5g-viz/`

2. **在同一次整理中修正舊文件的相對連結與索引**
   - 修正 `plans/` 內互相引用的 Markdown 路徑
   - 更新 `docs/README.md`
   - 視需要新增 `plans/README.md`，說明 `plans/` 是歷史規劃文件集合

3. **建立新的 `design/` 骨架**
   - 新增 `design/overview/`、`design/backend/`、`design/frontend/`、`design/grafana/`、`design/dvr/`、`design/features/`、`design/reference/`
   - 至少放入最小可讀的 stub 或導覽頁，避免 `design/` 目錄重新出現後仍是空殼

4. **逐步撰寫新的 canonical 文件**
   - 以程式碼為準描述現況
   - 舊 `plans/` 只作背景材料，不直接沿用其規劃語氣

### 建議方案 B：先寫新 `design/`，再搬舊文件

為避免整理過程中出現大量暫時失效的引用，也可以採取較保守的順序：

1. **先建立新的 canonical 文件骨架**
   - 先新增 `design/overview/`、`design/backend/`、`design/frontend/`、`design/grafana/`、`design/dvr/`、`design/features/`、`design/reference/`
   - 至少放入最小可讀的 stub 或初版內容，避免目標位置為空

2. **撰寫／補齊新的 `design/` 文件**
   - 以程式碼為準整理現況
   - 需要時參考舊 `design/` 與未來 `plans/` 內容，但不直接複製規劃語氣

3. **搬移舊規劃文件到 `plans/`**
   - 保留原內容
   - 同步修正檔名、同層連結與跨檔引用

4. **將被取代的舊總覽文件移入 `docs/archive/5g-viz/`**
   - archive 前先確認已有新的 canonical 文件能接手其用途

5. **最後更新索引與導覽**
   - 更新 `docs/README.md`
   - 補齊必要的 `README.md` / 導覽頁
   - 以 `rg` 檢查舊路徑是否仍殘留在文件中

### 本計畫的建議採用方式

若你希望撰寫期間視覺上就明確區分「舊規劃」與「新 canonical」，本計畫建議優先採用**方案 A**。

實務上可進一步收斂成兩個階段：

1. **結構整理 commit**
   - 搬移 `design/` → `plans/`
   - 修正連結
   - 更新索引

2. **內容重寫 commit(s)**
   - 逐步補齊新的 `design/` canonical 文件
