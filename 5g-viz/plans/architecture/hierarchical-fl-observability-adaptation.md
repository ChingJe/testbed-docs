# Hierarchical FL Observability Adaptation Assessment

> 狀態：架構評估／候選方向，尚未進入 implementation planning，也不是 confirmed design。
>
> 盤點日期：2026-08-28

## 1. 目的

新版 testbed 已具備 static hierarchical federated learning（HFL）的受控流程。下一步若要讓
`5g-viz` 呈現這類實驗，需要重新確認三個觀測面的資料來源與責任：

- topology：Root、Branch、Leaf 之間目前發生哪些互動；
- event log：各 participant 正在執行哪個 round／phase，以及成功、失敗或等待狀態；
- Grafana：local training、lower／upper aggregation、等待與 artifact transfer 等實驗成本。

本文件記錄初步盤點、候選架構、預計影響範圍與尚待決策項目。它不承諾特定 metric names、ports、
scrape interval、process count 或 implementation slices；這些內容必須在正式計畫中依當時 source、deployment
inventory 與實驗需求重新確認。

## 2. 現有 5g-viz 資料路徑

目前三個畫面並不是共用同一條傳輸路徑。

### 2.1 Topology 與 event log

Live mode 的既有流程是：

```text
remote process log
  -> 5g-viz collector（AsyncSSH + tail -F）
  -> parser
  -> structured event
  -> events.jsonl
  -> WebSocket /ws
  -> browser topology reactions + event log
```

WebSocket 位於 `5g-viz backend -> browser`，不是 process 到 `5g-viz` 的資料傳輸協定。拓樸的節點、邊與
event reactions 由 profile topology configuration 定義，WebSocket 只負責把已解析事件與 state snapshot
送到前端。

### 2.2 Live Grafana metrics

既有 live 路徑主要由 `5g-viz` 將 log event 投影為 in-process Prometheus metrics，再開放 `/metrics` 供
Prometheus scrape：

```text
log -> event -> 5g-viz metric handler -> 5g-viz /metrics
    -> Prometheus scrape -> Grafana -> embedded iframe
```

這個集中式轉接模型源自既有 Go NWDAF 沒有提供對應 metrics endpoint 的限制。它能運作，但若直接套用到大量
HFL participants，將使 `5g-viz` 必須從文字 log 重建所有 process 已經知道的測量值。

### 2.3 Replay metrics

可攜 replay 的 canonical artifact 仍是 session 目錄，而不是 Prometheus TSDB：

- `meta.json`
- `events.jsonl`
- `topology.yaml`

Replay 會從 session events 重建 metric samples，透過 Remote Write 回填帶有原始時間戳的資料。Live session
目前也會以 Remote Write 建立 session anchor，但一般 live metrics 仍主要走 scrape path。

## 3. 新 HFL 架構的候選分工

初步建議將「事件」與「metrics」分開處理。

```text
PyMTLF structured events
  -> VM journal / Host Docker logs
  -> 5g-viz collector + parser
  -> events.jsonl
  -> WebSocket
  -> topology + event log

PyMTLF native metrics
  -> candidate A: /metrics -> Prometheus scrape
  -> candidate B: Remote Write -> Prometheus receiver
  -> Grafana
  -> 5g-viz embedded chart area
```

這個方向保留現有 `5g-viz` 的 topology／event pipeline，但讓真正擁有測量值的 PyMTLF process 原生產生
metrics，避免把 log parser 當成主要 metrics transport。Scrape 與 Remote Write 都保留為正式候選；本評估不先
指定其中一個為預設方案。

### 3.1 Scrape 與 Remote Write 的選擇

Scrape 的優點是沿用 Prometheus 標準 pull model，process 只需暴露 endpoint，target health 與 `up` 語意也較直接。
它需要處理 endpoint reachability、target discovery 與 scrape interval，短暫狀態若建模不當也可能落在兩次 scrape
之間。

Remote Write 可由 process 在 measurement 產生時直接送出帶原始 timestamp 的 sample，不必等待下一次 scrape；
對低頻、離散且希望保留精確時間點的 HFL round／phase measurement 具有吸引力。其實作成本在目前範圍內不應先被
視為過高或低優先級，但正式方案仍需清楚處理：

- Remote Write encoding 與 batching；
- retry、backoff 與 receiver outage buffering；
- duplicate／out-of-order sample handling；
- process shutdown 時的 pending delivery；
- session、staleness 與 endpoint configuration。

這些責任可以透過 PyMTLF 共用 instrumentation／sender、中央 agent 或其他共用封裝收斂，不代表必須為每個 role
維護不同實作。反過來，scrape 所需的 endpoint lifecycle、port allocation 與 target discovery 也不是零成本。

正式計畫應以小型 proof of concept 比較兩條路徑，至少評估：

- measurement 到 Prometheus 可查詢的延遲；
- process／deployment 複雜度；
- Prometheus 暫時不可用時的資料完整性；
- exact timestamp、duplicate、out-of-order 與 staleness 語意；
- live 與 replay 能否共用 metric contract；
- Grafana refresh 後的實際使用者可見延遲。

最終可以選擇純 scrape、純 Remote Write，或依 metric 類型採混合模式；目前不排除任何一種。

## 4. 預計修改範圍

### 4.1 PyMTLF component

PyMTLF 預計需要提供兩類觀測輸出。

第一類是原生 metrics instrumentation，並透過 `/metrics` scrape endpoint、Remote Write sender 或最後選定的混合
模式輸出。候選測量包括：

- local training duration；
- Branch lower aggregation duration；
- Root upper aggregation duration；
- participant wait／straggler time；
- artifact count 與 bytes；
- sample count、round participation、success／failure；
- retry、timeout 與 cleanup 結果。

第二類是供 topology 與 event log 使用的 structured events，至少需能辨識：

- session／run identity；
- Root、Branch、Leaf role 與 participant identity；
- round、phase、tier、direction 與 status；
- event timestamp；
- replay 重建所需的測量值。

具體 metric names、types 與 labels 必須另行設計。Artifact digest、完整 URL、任意 error message 等高基數值不應作為
Prometheus labels，應保留在 event record。

### 4.2 5G_NWDAF_Infrastructure

Deployment／orchestration 預計需要：

- 為各 participant 配置可識別的 metrics transport identity；
- 若採 scrape，配置不衝突的 endpoint 並確保 Prometheus 到 endpoint 的 network reachability；
- 若採 Remote Write，配置 receiver location、sender lifecycle 與必要的 delivery state；
- 將 session／run identity 與 participant identity 注入 process；
- 從 selected configuration、manifest 或 Compose runtime inventory 解析 scrape target／Remote Write source 與 log source；
- 將 metrics transport、journal unit、container identity 與實際 run 綁定。

不應為 `5g-viz` 另外手工維護一份 Root／Branch／Leaf inventory。觀測設定應重用 deployment 已解析的 canonical
inventory，避免 topology、collector 與實際 runtime 漂移。

### 4.3 5g-viz collector 與 parser

現有 collector 只支援 AsyncSSH 後執行 `tail -F`。新 testbed 至少需要兩類 source adapter：

- VM journal：透過 SSH 讀取 exact systemd units；
- Host Docker：讀取 selected Compose project 中的 exact containers。

Collector 應保留結構化 source envelope，例如：

```json
{
  "kind": "remote_journal",
  "machine": "<machine>",
  "service": "<exact-unit-or-container-role>",
  "source": "<logical-participant>",
  "line": "<raw-message>"
}
```

實作時還需處理：

- journal cursor 與 reconnect 後的 gap／duplicate；
- exact unit、invocation 與 selected run fencing；
- `sudo -n journalctl` 所需權限；
- exact container ID、start time 與 restart 後 identity；
- 多機時鐘偏差與 arrival order；
- collector reconnect、deduplication 與 bounded buffering。

跨機 phase duration 不應只靠兩台機器的 wall-clock timestamp 相減；重要 duration 應由執行該 phase 的 process
使用本地 monotonic clock 計算後輸出。

Parser 則需將 HFL structured events 正規化為 `5g-viz` event schema，並保留 exact participant identity，讓同一台
VM 或 Host 上的多個 process 不會被合併成單一來源。

### 4.4 Topology profile 與 frontend reactions

Topology 本身仍可沿用 config-driven 模型，預計修改：

- Root／Branch／Leaf nodes 與 hierarchy edges；
- local training、lower aggregation、upper aggregation 與 artifact transfer reactions；
- participant waiting、failure、round complete 等 state／pulse 樣式；
- event filter 與 Event Log 顯示欄位。

現有 WebSocket 和 frontend reaction engine 原則上可以重用；主要工作是 event identity、profile configuration
與 HFL reaction definitions，而不是重新建立一套前端即時傳輸。

### 4.5 Prometheus 與 Grafana

Prometheus 端需依最終選擇新增 HFL participant scrape targets、啟用／配置 Remote Write receiver ingress，或同時
支援兩者。`5g-viz /metrics` 可繼續用於 `5g-viz` 自身健康與 collector 狀態；若後續加入 node exporter、cAdvisor
或 GPU exporter，這些 operational metrics 仍可獨立使用 scrape，不必強迫所有資料採用同一種 transport。

Grafana dashboard 需要依新實驗目標重新設計。候選 panels 包括：

- per-round end-to-end duration；
- Leaf local training duration comparison；
- Leaf completion spread／straggler；
- Branch lower aggregation 與 Root upper aggregation duration；
- Branch／Root wait time；
- artifact bytes by tier／direction；
- participant／round status、retry 與 failure；
- sample count 與 participation。

WAPE 或 degradation 可保留為輔助 quality diagnostic，但不應直接沿用為 HFL 成本觀測的主要 dashboard。若實驗
round 數量少，bar、timeline、state panel、table 或 heatmap 可能比連續曲線更適合。

### 4.6 Replay 與 session artifacts

無論 live 使用 scrape 或 Remote Write，都不會自動使 replay 可攜。若 `events.jsonl` 沒有保存完整測量值，離開
原本 Prometheus TSDB 後就無法重建 Grafana charts。

候選預設是讓 structured events 同時保留 replay 所需測量值：

```text
events.jsonl
  -> deterministic event-to-metric projector
  -> Remote Write historical samples
  -> Prometheus
  -> Grafana replay
```

這表示 live metrics 的 authoritative producer 是 PyMTLF instrumentation，transport 可以是 scrape、Remote Write
或混合模式；replay projector 則必須能從同一份 measurement contract 得到等價 samples。正式設計需驗證：

- live transport 與 replay projection 是否具有一致的 metric names、labels 與 values；
- event 是否記錄足夠的 original timestamp 與 measurement fields；
- replay backfill 的 duplicate、overwrite 與 session isolation；
- session artifact schema versioning 與向後相容策略。

另一個候選方案是每次實驗保存 Prometheus snapshot／export，但這會擴大 artifact lifecycle 與 portability 成本，
目前不列為優先方向。

## 5. Repository 影響

若正式實作，預計至少涉及：

- `nwdaf-docs`：PyMTLF observability contract 與 component implementation plan；
- PyMTLF owning source repository：structured events、native metrics instrumentation 與候選 transport；
- `5G_NWDAF_Infrastructure`：metrics transport configuration、runtime inventory、networking、session identity 與 integration tests；
- `5g-viz`：collector adapters、event parser、profile、topology reactions、replay projector、Grafana dashboard 與 tests；
- `testbed-docs`：5g-viz adaptation plan、testbed integration plan、operations 與 validation records。

Go NWDAF 不一定需要修改。只有在 topology 需要呈現 Go NWDAF 內部的 exact SBI interaction，或需要 Go process
提供額外 timing／metrics 時，才應把它加入正式 scope。

## 6. 候選實作順序

以下只是用來估計工作面積的候選順序，不代表已核准的 slices：

1. 定義 HFL event／metric／identity contract，並以單一 participant 比較 scrape 與 Remote Write proof of concept。
2. 在 deployment inventory 中加入 metrics transport configuration 與 log source discovery。
3. 完成 VM journal／Host Docker collector adapters、run fencing 與 parser normalization。
4. 建立 HFL topology profile、event reactions 與 Event Log 呈現。
5. 依 PoC 結果建立 Prometheus scrape／Remote Write integration 與 HFL Grafana dashboard。
6. 完成 replay event-to-metric projector 與 live／replay consistency verification。
7. 執行完整 static hierarchy 實驗，驗證 topology、events、metrics、replay 與 failure handling。

在進入 implementation 前，應依 workspace policy 為跨 repository 工作建立正式 plan，明確列出 source ownership、
branch／revision、conformance map、acceptance evidence 與 repository-by-repository commit boundaries。

## 7. 尚待決策

正式計畫前至少需要決定：

1. PyMTLF 採 scrape、Remote Write 或混合模式，以及共用 instrumentation／sender 的 library 與 lifecycle ownership。
2. 若採 scrape，Prometheus 如何從 canonical deployment inventory 取得 targets，以及是否需要 file-based service
   discovery。
3. 若採 Remote Write，receiver、batching、retry、buffering、shutdown flush 與 delivery observability 如何處理。
4. Live scrape／push latency、Grafana refresh interval 與可接受的顯示延遲。
5. HFL metric names、types、bounded labels 與 session retention policy。
6. Structured event encoding、schema version 與跨 process identity contract。
7. Replay 是由 events deterministic projection，或另存 metrics artifact；目前偏向前者。
8. 哪些 measurements 屬於 protocol／artifact cost，哪些需要額外 exporter 才能代表 CPU、memory、GPU 或 network
   resource cost。
9. Go NWDAF 是否只作為 topology participant，或也需要新增 observability instrumentation。

## 8. 初步結論

適配 HFL 的主要成本不在 WebSocket 或拓樸繪圖本身，而在：

- 讓 PyMTLF 原生產生可信的 structured events 與 metrics；
- 從 runtime inventory 正確識別、蒐集並隔離多個 participants；
- 重新設計 HFL Grafana dashboard；
- 保持 live native metrics 與 portable replay 的資料語意一致。

候選架構保留 `5g-viz` 作為 event aggregation、topology、session recording 與 replay orchestration 層；PyMTLF
則原生產生 metrics，並保留 scrape 與 Remote Write 兩條正式候選路徑。Remote Write 不被視為低優先級，scrape
也不被預設為唯一正解；兩者應在正式計畫或 PoC 中依延遲、可靠性、部署成本與 replay consistency 再做選擇。
Replay backfill 仍可由中央 projector 使用 Remote Write，無須和 live transport 做出相同選擇。
