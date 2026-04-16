# 5g-viz 文件重構計畫

狀態：提案中  
最後更新：2026-04-16  
本檔範圍：第二階段文件重構藍圖。本檔只定義分析、資訊架構、phase、mapping、驗收與交接方式，不直接改寫既有文件正文。

## 1. 背景與目標

這一輪不是要補 correctness，也不是要補更多 implementation detail。

這一輪的核心目標是：把 `5g-viz` 的文件從「實作導向的系統說明」往前補上一層「人類導向的理解層」，讓讀者能先從下列角度理解系統，再決定是否進入 deeper reference：

- 實際使用時會看到什麼
- 主畫面各區塊在做什麼
- live 與 replay 在使用行為與畫面行為上有什麼差異
- 常見操作時，畫面、資料、圖表分別會怎麼變
- event、snapshot、metrics、Prometheus、Grafana、pseudo-live 之間的關係

目前 repo 已經有不少深層、技術正確、維護者友善的文件。真正缺的是這一層：

- 「我剛打開 UI，我到底在看什麼？」
- 「為什麼剛剛會出現這個現象？」
- 「如果我只是想理解系統，不想先讀 `main.py` / `events.js` / `state.py`，我應該從哪裡開始？」

本計畫是根據實際 repo 內容規劃，不是只根據檔名猜測。

## 2. 規劃前實際查看了哪些 docs 與 code areas

### 已閱讀 docs

- `docs/README.md`
- `docs/5g-viz/design/README.md`
- `docs/5g-viz/design/overview/README.md`
- `docs/5g-viz/design/overview/system.md`
- `docs/5g-viz/design/overview/data-flow.md`
- `docs/5g-viz/design/frontend/README.md`
- `docs/5g-viz/design/frontend/topology.md`
- `docs/5g-viz/design/frontend/events-and-dvr.md`
- `docs/5g-viz/design/frontend/grafana-embed.md`
- `docs/5g-viz/design/backend/README.md`
- `docs/5g-viz/design/backend/api.md`
- `docs/5g-viz/design/backend/state.md`
- `docs/5g-viz/design/backend/metrics.md`
- `docs/5g-viz/design/dvr/README.md`
- `docs/5g-viz/design/dvr/overview.md`
- `docs/5g-viz/design/dvr/session.md`
- `docs/5g-viz/design/dvr/replay.md`
- `docs/5g-viz/design/features/README.md`
- `docs/5g-viz/design/features/traffic-chart.md`
- `docs/5g-viz/design/features/subscription-chain.md`
- `docs/5g-viz/notes/internals/5g-viz-internals.md`
- `docs/5g-viz/notes/impl/2026-04-10-impl.md`
- `docs/5g-viz/notes/meetings/2026-04-10-meeting.md`
- `5g-viz/README.md`

### 已閱讀 code / runtime areas

#### 啟動與模式切換

- `5g-viz/start.sh`
- `5g-viz/setup.sh`
- `5g-viz/main.py`

#### 前端 UI / 事件 / DVR / Grafana

- `5g-viz/frontend/index.html`
- `5g-viz/frontend/events.js`
- `5g-viz/frontend/topology.js`

#### 後端 event / state / session / replay

- `5g-viz/state.py`
- `5g-viz/collector.py`
- `5g-viz/parser.py`
- `5g-viz/metric_player.py`

#### metrics / Grafana / config / rules

- `5g-viz/config.py`
- `5g-viz/grafana_setup.py`
- `5g-viz/rules/__init__.py`
- `5g-viz/rules/nwdaf.py`
- `5g-viz/rules/nwdaf_sub.py`
- `5g-viz/rules/smf.py`
- `5g-viz/profiles/default/topology.yaml`

#### session artifact 樣本

- `5g-viz/sessions/20260415T060245034/meta.json`
- `5g-viz/sessions/20260415T060245034/events.jsonl`

### 規劃時特別確認過的實際系統行為

- UI 實際上有六個最重要的可觀察區塊：
  - 連線 / 模式狀態
  - DVR controls
  - filter sidebar
  - topology
  - Grafana 區塊
  - event log
- live mode 主要靠 `/ws` 推事件，但「回到 live」會另外用 `/api/state` 拉權威 snapshot。
- replay mode 正常前端資料來源不是 WebSocket，而是 `/api/events` 與 `/api/session-info`。
- 新分頁 / 新 live client 只會先拿到 `state_snapshot`，不會自動重播完整歷史動畫。
- replay 的 chart 其實有兩條資料路徑：
  - 原始 session backfill
  - replay 播放中的 pseudo-live remote write
- 並不是所有 event 都會進 Grafana；只有一部分 metric event types 會寫 Prometheus。
- `event_reactions` 同時影響：
  - 前端 topology 視覺反應
  - backend `state_snapshot` 的持久狀態

## 3. 現況問題診斷

目前文件的強項是技術正確、實作對齊、細節充分；弱項是 onboarding 與使用者視角不足。

### 最影響理解的問題

1. 入口文件是 implementation-first，不是 reader-first。  
   `design/overview/system.md`、`data-flow.md`、`design/dvr/overview.md` 很快就進入元件、模式、pipeline，而不是先幫讀者建立「這是什麼系統、我會看到什麼畫面」。

2. 文件骨架主要沿著 code / subsystem 切，不是沿著讀者問題切。  
   現有結構以 `frontend`、`backend`、`dvr`、`grafana`、`overview` 為主，對維護者合理，但不適合作為第一層閱讀路徑。

3. 缺少一份真正的「How to read the screen」文件。  
   程式碼顯示 UI 區塊很明確，但文件沒有一份從畫面區塊出發的主文件。

4. live vs replay 雖然到處都有講，但沒有被整理成一個讓人容易抓住的心智模型。  
   相關資訊分散在 `system.md`、`data-flow.md`、`events-and-dvr.md`、`metrics.md`、`dvr/replay.md`，讀者需要自己拼。

5. 很多使用者最常困惑的現象，目前是以 internal mechanism 的形式存在，而不是以 FAQ / troubleshooting / scenario 形式存在。  
   實際從 code 確認的例子包括：
   - 新分頁只看到當前 state，不會重播完整歷史
   - live pause 時 backend 其實還在往前跑
   - replay paused chart 與 replay playing chart 底層不是同一組 series
   - topology 有反應，不代表 Grafana 一定同步有線

6. 高層文件過早引入低層詞彙。  
   例如 `state_snapshot`、`event_reactions`、remote write、pre-seed、pseudo-live、`MetricPlayer` 這些詞，在讀者還沒建立 observable behavior 前就出現。

7. human-facing layer 與 reference layer 沒有被明確切開。  
   `design/*` 雖然被標成 canonical，但實際上多數更像 deep technical reference；同時旁邊還有 `notes/`、`plans/`，會增加閱讀入口混亂。

8. feature docs 已經存在，但仍偏 internal path explanation，不夠像 operator scenario。  
   例如讀者更可能問：
   - 為什麼 topology 動了但 Grafana 沒動？
   - replay 時我應該看 topology 還是看 chart？
   - Pause 到底 freeze 了什麼？
   目前這種提問方式還不是文件主骨架。

9. `notes/*` 很有價值，但不適合作為第一層 canonical 說明。  
   它們混合了歷史決策、實作過程與現況，是重要背景，但不應成為 onboarding 主路徑。

## 4. 重構原則

1. 每篇高層文件先從使用者可觀察到的現象與操作開始。

2. 先建立心智模型，再講內部機制。

3. 不用 function name、internal state name、handler name 當高層章節骨架。

4. 保留技術正確性，但避免在第一屏就掉進低層細節。

5. 保留既有 deep design docs，但把它們視為 reference layer，而不是 onboarding layer。

6. 以 scenario / workflow / screen-reading 語言取代 subsystem-first 語言。

7. 優先增加新的 reader path，而不是一開始就大規模搬動現有檔案。

8. `plans/*` 與 `notes/*` 保留，但應清楚標示其角色：
   - `plans/*`：規劃歷史
   - `notes/*`：內部筆記 / 背景
   - `design/*`：深層設計 / reference
   - 新增的人類導向層：主要閱讀入口

9. 高層文件的敘述語氣採直接陳述式，不採導覽式第二人稱。
   - 避免使用第二人稱，如「你會看到」「如果你想」「你現在看到的是」。
   - 避免假設讀者的視覺感受、體感或主觀反應，如「第一眼會覺得」「直覺上會看到」。
   - 優先直接說明物件是什麼、用途是什麼、與其他部分的關係是什麼。

10. 高層文件不使用寫給作者或維護者自己的階段性口吻。
   - 避免使用「目標不是取代...」「這一層刻意不做...」這類偏規劃備忘錄式寫法作為正文主敘述。
   - 若需要說明文件層級或閱讀定位，應直接陳述其角色，例如「本目錄收錄...」「本文件說明...」「`design/*` 保留為 reference」。

## 5. 新的資訊架構建議

### 建議的邏輯結構

```text
docs/5g-viz/
  README.md
  start-here/
    README.md
    what-is-5g-viz.md
    live-vs-replay.md
    screen-tour.md
    first-concepts.md
  ui-workflows/
    topology.md
    event-log.md
    event-history.md
    grafana.md
    dvr-controls.md
  mental-model/
    README.md
    events-snapshots-metrics.md
    live-vs-replay-data-paths.md
    ui-only-vs-metric-events.md
  troubleshooting/
    common-scenarios.md
  reference/
    README.md
    ...（初期先指向既有 design/*）
  design/
  plans/
  notes/
```

### 重要過渡策略

前 2 到 3 個 phase 不建議先把 `design/*` 全部實體搬動。

比較合理的做法是：

- 先新增 human-facing layer
- 先建立新的閱讀入口
- 先讓 `design/*` 被重新定位為 reference layer
- 等高層文件真的成形後，再決定是否有必要做 physical move / rename

這樣可以避免一開始就因為大規模搬檔案造成 link churn。

### 新讀者建議閱讀順序

1. `docs/5g-viz/README.md`
2. `start-here/what-is-5g-viz.md`
3. `start-here/live-vs-replay.md`
4. `start-here/screen-tour.md`
5. `ui-workflows/*`
6. `mental-model/*`
7. `troubleshooting/common-scenarios.md`
8. `reference/README.md` 與其指向的 deep docs

### very short outline examples

以下只是未來文件骨架示例，不是正文草稿。

#### 示例：`start-here/screen-tour.md`

- 第一次打開畫面會看到什麼
- Header 與模式 / 連線狀態
- Topology 區塊
- Grafana 區塊
- DVR controls
- Event log
- Live 與 replay 在畫面上哪裡不同
- 下一步該讀哪份文件

#### 示例：`ui-workflows/dvr-controls.md`

- Pause freeze 了什麼、沒 freeze 什麼
- Play 重播的是什麼
- Go Live 真的恢復了什麼
- Chart window 會改變什麼
- Replay 時為什麼更複雜
- 常見誤解

## 6. 舊文件到新架構的 mapping

| 現有文件 / 類別 | 未來角色 | 規劃中的處理方式 |
|---|---|---|
| `5g-viz/README.md` | setup / run reference | 保留為 repo-level setup 文件；從新 gateway 連過去，但不作為主要概念入口。 |
| `design/overview/system.md` | human overview + reference | 抽出人類導向部分到 `start-here/what-is-5g-viz.md`；保留組件級內容在 reference。 |
| `design/overview/data-flow.md` | mental-model bridge + reference | 把 reader-facing 部分拆到 `start-here/live-vs-replay.md` 與 `mental-model/live-vs-replay-data-paths.md`；原檔保留深層資料流。 |
| `design/dvr/overview.md` | user-facing DVR 概念 + reference | 觀察者視角的部分進 `ui-workflows/dvr-controls.md`；跨層細節保留原檔。 |
| `design/dvr/replay.md` | mental model + troubleshooting | 把 replay 需要 backfill / pseudo-live 的使用者層說明抽到 `mental-model/*` 與 troubleshooting；原檔保留 runtime 細節。 |
| `design/dvr/session.md` | first concepts + reference | 把「session 是什麼」放進 `start-here/first-concepts.md`；`meta.json` / `events.jsonl` / `topology.yaml` 細節保留原檔。 |
| `design/frontend/topology.md` | workflow doc + reference | 新增 `ui-workflows/topology.md`；Cytoscape / filter / reaction 細節仍留在 reference。 |
| `design/frontend/events-and-dvr.md` | workflow doc + reference | 新增 `ui-workflows/dvr-controls.md`、`ui-workflows/event-log.md`、`ui-workflows/event-history.md`；state-machine 細節保留原檔。 |
| `design/frontend/grafana-embed.md` | workflow doc + reference | 新增 `ui-workflows/grafana.md`；iframe / session / window mechanics 保留原檔。 |
| `design/features/traffic-chart.md` | scenario support + reference | 作為 Grafana / troubleshooting 文件的支援 reference。 |
| `design/features/subscription-chain.md`、`nwdaf-ml-cycle.md` | scenario examples | 保留為具體 feature flow 例子；從 workflow / mental-model 文件往下連。 |
| `design/backend/*.md`、`design/grafana/*.md`、`design/reference/*.md`、`design/overview/event-schema.md`、`design/overview/architecture.md` | deep reference layer | 暫時保留原位；後續補 index / link-in / audience framing。 |
| `notes/*` | historical / internal | 保留，但明確標成非 onboarding 主入口。 |
| `plans/*` | planning history | 保留，但不放進主閱讀路徑。 |

## 7. 分階段執行方案

這些 phase 不是套模板切出來的，而是根據 repo 現況切的：

- deep technical docs 已經很多
- 真正缺的是 human-facing layer
- 主要風險是 path confusion 與 duplication，不是缺 internal detail

### Phase 1：入口層與導覽層

#### 目標

先建立一條新讀者可用的閱讀入口，讓人不用先進 `design/*` 也能知道：

- 這個系統是什麼
- 主畫面有哪些區塊
- live / replay 差在哪裡
- 下一步應該讀哪一層

#### 範圍

- 新增 `docs/5g-viz/README.md`
- 新增 `start-here/` index 與最小 onboarding set
- 從新入口把既有 deep docs 重新串成清楚的路徑
- 明確區分：
  - human-facing docs
  - design/reference docs
  - notes
  - plans

#### 預計產出

- `docs/5g-viz/README.md`
- `docs/5g-viz/start-here/README.md`
- `docs/5g-viz/start-here/what-is-5g-viz.md`
- `docs/5g-viz/start-here/live-vs-replay.md`
- `docs/5g-viz/start-here/screen-tour.md`
- 必要的 index / link 更新

#### 不要做的事

- 不要重寫 backend / grafana deep docs
- 不要開始擴寫 implementation detail
- 不要在這一 phase 大規模搬動既有檔案

#### 驗收標準

新讀者不打開 `design/*` 也能回答：

- 5g-viz 是做什麼的？
- 畫面上主要區塊是什麼？
- live 與 replay 在使用上有什麼不同？
- 如果我要看 deeper implementation / reference，該往哪裡走？

#### 風險 / 注意事項

- 風險：和現有 overview docs 重複。
- 處理方式：Phase 1 文件要刻意短、刻意 reader-facing，重點是建立路徑，不是複製 deep explanation。

### Phase 2：UI 與工作流程層

#### 目標

把系統真正最常被看的東西，按照畫面與操作流程講清楚。

#### 範圍

- topology 在看什麼
- event log 在看什麼
- event 何時進前端、前端如何保留與補抓歷史
- Grafana 在看什麼
- DVR controls 在做什麼
- 需要直接由畫面與操作理解的跨區塊行為

#### 預計產出

- `docs/5g-viz/ui-workflows/topology.md`
- `docs/5g-viz/ui-workflows/event-log.md`
- `docs/5g-viz/ui-workflows/event-history.md`
- `docs/5g-viz/ui-workflows/grafana.md`
- `docs/5g-viz/ui-workflows/dvr-controls.md`

#### 不要做的事

- 不要把 workflow 文件變成 API reference
- 不要用 `events.js` / `topology.js` / function 名稱當章節骨架
- 不要在還沒把 observable behavior 講清楚前，就先講 remote write / internal state

#### 驗收標準

讀者能清楚說出：

- topology 顯示的是什麼
- event log 顯示的是什麼
- Grafana 顯示的是什麼
- Pause / Play / Scrub / Go Live / Chart Window 各自做什麼
- 哪些操作在 live 與 replay 下有不同的畫面與資料行為

#### 風險 / 注意事項

- 風險：變成 frontend deep docs 的第二份副本。
- 處理方式：每篇 workflow doc 固定用下面順序：
  - 區塊或機制是什麼
  - 主要用途與操作
  - 常見誤解
  - 如需細節，再往下連 reference

### Phase 3：心智模型與 troubleshooting 層

#### 目標

回答「為什麼會這樣」的問題，且回答方式要從使用者現象出發，而不是先丟 internal mechanism。

#### 範圍

- event / snapshot / metrics 的關係
- Prometheus / Grafana / topology 各自的角色
- live 與 replay 為什麼資料路徑不同
- replay 為什麼需要 backfill / pseudo-live
- 哪些 event 只影響 UI，哪些也會進 Grafana
- 常見使用者情境與誤解

#### 預計產出

- `docs/5g-viz/mental-model/events-snapshots-metrics.md`
- `docs/5g-viz/mental-model/live-vs-replay-data-paths.md`
- `docs/5g-viz/mental-model/ui-only-vs-metric-events.md`
- `docs/5g-viz/troubleshooting/common-scenarios.md`

#### 不要做的事

- 不要擴寫與目前 5g-viz 無關的 infra runbook
- 不要把 troubleshooting 寫成 raw internal debug note
- 不要假裝 live / replay 在所有層都完全等價，因為 code 實際上不是這樣

#### 驗收標準

文件可以直接回答這些已由 code 確認的問題：

- topology 有反應但 Grafana 沒線
- 新分頁為什麼只看到當前狀態
- replay 為什麼需要 backfill / pseudo-live
- replay paused chart 與 replay playing chart 為什麼不是同一組 series
- 哪些 event 只影響 UI，哪些會進 Prometheus / Grafana

#### 風險 / 注意事項

- 風險：又退回 implementation-first 寫法。
- 處理方式：每個 scenario 固定用：
  - 症狀
  - 最可能的解釋
  - 需要時再補 under-the-hood
  - deeper reference link

### Phase 4：reference re-anchoring 與 legacy cleanup

#### 目標

讓現有 deep docs 在新架構下更容易被找到、被正確使用，同時降低重複與入口混亂。

#### 範圍

- 新增 `reference/README.md` 或等價的 reference index
- 把 `design/*` 明確重新定位為 deep technical reference
- 從新的人類導向文件向下補精準 reference links
- 視需要決定是否值得做 physical move / rename
- 視需要替 `notes/*`、`plans/*` 補 audience framing

#### 預計產出

- 更清楚的 reference layer index
- 舊文件到新閱讀路徑的橋接連結
- 必要時少量 preface / banner / note
- 若有充足理由，再做選擇性路徑清理

#### 不要做的事

- 不要全面重寫所有 deep docs
- 不要只為了「看起來整齊」就搬大量檔案
- 不要在沒有明顯收益時破壞既有連結

#### 驗收標準

- 讀者可以從任一 human-facing doc 明確跳到 1 到 2 份對應 deep docs。
- `design/*` 不再是 accidental onboarding entry。
- `notes/*`、`plans/*` 與 canonical / reader-facing 路徑的角色清楚分開。

#### 風險 / 注意事項

- 風險：過度搬檔造成 link churn。
- 處理方式：先做 logical reclassification，再視 Phase 1 到 3 的實際結果決定要不要 move。

## 8. 為什麼這樣切 phase，而不是其他切法

這份切法刻意不是按 `frontend` / `backend` / `grafana` / `dvr` 切，因為那樣只會複製現有問題。

也刻意不是一開始就先「重寫所有 overview」，因為 repo 現況不是 overview 太少，而是缺少一條真正 reader-facing 的閱讀路徑。

這樣切 phase 的理由是：

1. 目前 repo 已經有足夠多的 deep technical docs，可作為後續連結落點。
2. 最大缺口是入口層，不是 reference layer。
3. UI / workflow 應該先於深層 mental model，因為讀者要先知道自己在看什麼。
4. troubleshooting 需要 Phase 1 與 Phase 2 的共通詞彙，否則容易再次寫成 implementation note。
5. reference cleanup 放最後，才能避免在 reader path 尚未成形前就先做高 churn 搬動。

## 9. 重構進度表

這個區塊用來記錄實際重構進度，而不是只記規劃。

後續 session 若有執行任何 phase，應同步更新：

- 各 phase 狀態
- 已新增 / 已調整的文件
- 本輪完成項
- 下一步建議

### Phase status tracker

| Phase | 名稱 | 狀態 | 最近更新 | 備註 |
|---|---|---|---|---|
| 1 | 入口層與導覽層 | Completed | 2026-04-16 | `docs/5g-viz/README.md` 與 `start-here/` 初版已建立，並完成第一輪語氣收斂 |
| 2 | UI 與工作流程層 | Completed | 2026-04-16 | `ui-workflows/` 初版已建立，並完成 `event-history.md`、`state_snapshot` 與 `pre-seed` 補強 |
| 3 | 心智模型與 troubleshooting 層 | Completed | 2026-04-16 | `mental-model/` 與 `troubleshooting/common-scenarios.md` 初版已建立 |
| 4 | reference re-anchoring 與 legacy cleanup | In Progress | 2026-04-16 | 已新增 `reference/README.md`、`notes/README.md`，並開始收斂 `design/*` 與索引頁的定位 |

### Artifact tracker

| 類別 | 項目 | 狀態 | 備註 |
|---|---|---|---|
| 計畫 | `plans/docs/5g-viz-restructure-plan.md` | Active | 正式規劃檔；後續應持續更新 phase 狀態與決策 |
| 入口 | `docs/5g-viz/README.md` | Drafted | Phase 1 初版已建立 |
| 入口 | `docs/5g-viz/start-here/README.md` | Drafted | Phase 1 初版已建立 |
| 入口 | `docs/5g-viz/start-here/what-is-5g-viz.md` | Drafted | Phase 1 初版已建立 |
| 入口 | `docs/5g-viz/start-here/live-vs-replay.md` | Drafted | Phase 1 初版已建立 |
| 入口 | `docs/5g-viz/start-here/screen-tour.md` | Drafted | Phase 1 初版已建立 |
| workflow | `docs/5g-viz/ui-workflows/README.md` | Drafted | Phase 2 初版已建立 |
| workflow | `docs/5g-viz/ui-workflows/topology.md` | Drafted | Phase 2 初版已建立 |
| workflow | `docs/5g-viz/ui-workflows/event-log.md` | Drafted | Phase 2 初版已建立 |
| workflow | `docs/5g-viz/ui-workflows/event-history.md` | Drafted | 取代較不穩定的 `common-workflows.md`，聚焦 event buffer / history fetch / scrub 行為 |
| workflow | `docs/5g-viz/ui-workflows/grafana.md` | Drafted | Phase 2 初版已建立 |
| workflow | `docs/5g-viz/ui-workflows/dvr-controls.md` | Drafted | Phase 2 初版已建立 |
| mental-model | `docs/5g-viz/mental-model/README.md` | Drafted | Phase 3 初版已建立 |
| mental-model | `docs/5g-viz/mental-model/events-snapshots-metrics.md` | Drafted | 說明 event / snapshot / metrics 的分工 |
| mental-model | `docs/5g-viz/mental-model/live-vs-replay-data-paths.md` | Drafted | 說明 live / replay 的資料來源與 chart 路徑差異 |
| mental-model | `docs/5g-viz/mental-model/ui-only-vs-metric-events.md` | Drafted | 說明 UI-only event 與 metric event 邊界 |
| troubleshooting | `docs/5g-viz/troubleshooting/common-scenarios.md` | Drafted | 整理跨區塊常見現象與最常見解釋 |
| reference | `docs/5g-viz/reference/README.md` | Drafted | deep reference 導覽與主題索引 |
| notes | `docs/5g-viz/notes/README.md` | Drafted | notes 的定位、用途與邊界 |
| 索引 | `docs/README.md` | Updated | 已補 `5g-viz` 入口與 `plans/docs/` 區塊 |
| 索引 | `docs/5g-viz/design/README.md` | Updated | 已補「先看新入口」導向 |
| 索引 | `docs/5g-viz/plans/README.md` | Updated | 已補 docs 規劃區塊 |
| 索引 | `docs/5g-viz/README.md` | Updated | 已加入 `ui-workflows/` 閱讀入口 |
| 索引 | `docs/5g-viz/start-here/README.md` | Updated | 已加入 workflow 層導向 |

### Current checkpoint

| 日期 | 狀態摘要 | 下一步 |
|---|---|---|
| 2026-04-16 | 規劃藍圖完成；Phase 1 入口層已建立；Phase 2 `ui-workflows/` 初版已建立；Phase 3 `mental-model/` 與 `troubleshooting/` 初版已建立；Phase 4 已開始收斂 reference / design / notes 的定位；寫作語氣約束已納入計畫；`common-workflows.md` 已替換為 `event-history.md` | 依審核結果決定：繼續完成 Phase 4 的 reference re-anchoring，或做最小整理後進入維護模式 |

### Update rule

每次後續 session 若有實際推進，至少更新以下三處之一：

- `Phase status tracker`
- `Artifact tracker`
- `Current checkpoint`

若 phase 邊界或定義改變，則還要同步更新：

- `分階段執行方案`
- `決策紀錄區塊`

## 10. 後續 session 接手說明

後續 session 若要接手，建議照以下順序：

1. 先讀本檔。
2. 確認目前獲准執行的是哪一個 phase。
3. 若 decision log 尚未更新，不要自行推翻本檔的 phase 邊界。
4. 在寫新文件前，再次對照實際 code path，特別是：
   - `main.py`
   - `frontend/events.js`
   - `frontend/topology.js`
   - `state.py`
   - `metric_player.py`
   - `rules/nwdaf.py`
   - `profiles/default/topology.yaml`
5. 優先往下連 existing reference docs，不要複製第二份 deep explanation。
6. 若後續實作發現本計畫的結構太細或太粗，先更新本檔，再開始新增 / 改寫文件。
7. 後續新增的人類導向文件，需遵守本檔「重構原則」中的語氣約束，特別是：
   - 不使用第二人稱作為主要敘述方式
   - 不假設讀者主觀感受
   - 不在正文中使用寫給作者自己的階段性規劃口吻

### Session continuation checklist

- Approved phase：
- 本次新增 / 修改了哪些文件：
- 新發現的 open questions：
- 是否改了 mapping / phase 邊界：
- 是否更新 decision log：
- 是否更新 progress tracker：

## 11. 決策紀錄區塊

### Active decisions

| 日期 | 決策 | 理由 | 狀態 |
|---|---|---|---|
| 2026-04-16 | 先在 `design/*` 之上新增 human-facing layer，而不是先重寫 `design/*` | 現況真正缺的是 explanation layer，不是 reference 細節 | Proposed |
| 2026-04-16 | 延後大規模 physical move / rename | 先避免 path churn，等新 reader path 成形後再判斷是否值得搬 | Proposed |
| 2026-04-16 | 用 start-here / ui-workflows / mental-model / troubleshooting 切，而不是按 subsystem 切 | 比較貼近真實讀者問題與實際 UI / runtime 行為 | Proposed |

### Progress log

| 日期 | Phase | 摘要 | By |
|---|---|---|---|
| 2026-04-16 | Planning | 已完成 repo/doc audit，並新增正式重構藍圖 | Codex |

### Open questions

- `docs/5g-viz/README.md` 未來是否應成為唯一 landing page，還是與 `design/README.md` 長期共存但服務不同 audience？
- 完成 Phase 1 到 3 後，是否真的有必要做實體 `reference/` 搬移，還是用 logical index 就足夠？
- `features/*` 應長期保留為獨立 scenario docs，還是有些應被吸收到 workflow / mental-model 文件中？
