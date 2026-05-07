# DVR Mode: Offline Log to Session Import

日期：2026-05-07

定位：此文件描述的是 DVR / replay 主流程完成後的後續新增實作，用來補上「未事先錄製 live session 時，仍可由既有 log 生成 replay session」的能力。它不是原始 DVR 基線的一部分，而是建立在既有 session artifact、replay loader 與 Grafana backfill 已經存在的前提上。

本文件規劃一個新功能：在沒有事先開啟 5g-viz live recording 的情況下，直接把既有 log 檔轉成可 replay 的 session 目錄。

## 0. 實作收斂

此功能已於 2026-05-07 完成第一輪落地，現況如下：

- 已新增 `import_logs.py` 作為離線匯入 CLI。
- 已新增 `session_import.py` 負責 `log -> session artifact` 的主流程。
- 已新增 `session_io.py`，承接原本散在 `main.py` 內的 session / JSONL / time helper。
- replay 主流程無需為此功能另開新格式，仍直接讀 `events.jsonl`、`meta.json`、`topology.yaml`。
- `nwdaf.log` 為必要輸入；`free5gc.log` 為 optional enrichment，目前只補 `SMF -> UPF` 的 `Nupf_EventExposure_Subscribe`。
- 與 live session 已知不一致的 `topology_suppressed` 問題已修正，方式是讓 importer 重播必要的 runtime state advance。
- `nf_up` 已確認為歷史殘留並已移除，不再作為 replay / import 的支援項。
- importer 預設會在 session 開頭加入 5 秒 lead-in，避免 replay 一按播放就立刻湧出第一批事件；可用 `--lead-in-seconds` 調整或關閉。

因此本文件後續內容應視為：

- 已完成設計的實作說明
- 為何這樣實作的設計依據
- 後續若要擴充時可沿用的邊界與驗證標準

## 1. 問題定義

目前 replay 模式依賴 `sessions/<session_id>/` 下的既有錄製結果。這代表若使用者想事後重播某次實驗，必須在實驗進行當下就先開著 5g-viz，讓 live pipeline 同步錄製 `events.jsonl`。

這個前提太強，會造成兩個實際問題：

- 若當次實驗保留了 `nwdaf.log`，但沒有啟動 5g-viz，事後無法補做 replay。
- 若想把既有 log 當成分析素材，只使用 replay / Grafana / topology，而不需要連 VM live tail，現在沒有入口。

因此需要一個「離線匯入」流程，能將原始 log 轉為正式 session artifact，之後完全沿用現有 replay 能力。

## 2. 目標

### 2.1 功能目標

提供一個離線 CLI，輸入一組原始 log 檔，輸出：

```text
sessions/<session_id>/
  meta.json
  events.jsonl
  topology.yaml
```

輸出格式必須與 live recording 產物相容，讓使用者可直接：

```bash
./start.sh --replay sessions/<session_id>
```

### 2.2 設計目標

- 最大化複用既有 parser / rule / replay / Grafana backfill 邏輯。
- 不重做第二套 event schema。
- 不把離線匯入邏輯硬塞進 live runtime。
- 匯入失敗時，不污染既有 session 目錄。
- 匯入結果要可重現、可驗證、可攜帶。

## 3. 非目標

以下不在第一階段範圍內：

- 不重建 VM SSH tail 流程。
- 不直接從原始 log 在前端播放。
- 不改 replay 主流程去支援新的資料格式。
- 不嘗試補齊目前 parser 尚未支援的所有 log pattern。
- 不做跨 session 的自動 merge、切片或壓縮。
- 不把匯入器做成背景服務或 API；第一版以 CLI 為主。

## 4. 現況事實

### 4.1 Live recording 的現有資料流

目前 live 模式的核心流程是：

```text
collector -> parser.parse_line() -> event
                                -> append events.jsonl
                                -> update Prometheus metrics
                                -> broadcast WebSocket
```

也就是說，真正被持久化的只有 parser 產出的 event，而不是原始 log。

### 4.2 Replay 的現有輸入契約

目前 replay 啟動時需要：

- `events.jsonl`
- `meta.json`
- `topology.yaml`

其中：

- `events.jsonl` 是 replay timeline、event log、state rebuild、Grafana backfill 的主要來源。
- `meta.json` 提供 `session_id`、`start_time`、`end_time`、`grafana_groups`、`profile`。
- `topology.yaml` 決定 topology node、event reactions 與 replay 視覺行為。

### 4.3 已可直接複用的程式

以下能力已存在，應直接複用：

- `parser.parse_line()`：將單行 log 轉為標準 event。
- `rules/*`：既有事件辨識與 event payload 建構。
- `main.py` 的 replay 載入流程：讀取 session 目錄並回放。
- `_build_replay_metric_series()`：由 event list 重建 Prometheus 時序資料。
- 前端 DVR / timeline / topology replay：只依賴 event list 與 topology config。

結論：新功能不是重做 replay，而是新增一個「原始 log -> 現有 session artifact」的轉換器。

## 5. 方案選擇

### 5.1 推薦方案

新增一個獨立 CLI，例如：

```bash
uv run python import_logs.py \
  --profile default \
  --source nwdaf=/path/to/nwdaf.log
```

執行後產生新的 session 目錄，再由既有 `./start.sh --replay ...` 使用。

### 5.2 為什麼不用直接改 `start.sh`

若把匯入流程硬加進 `start.sh`，會讓這個腳本同時負責：

- live 啟動
- replay 啟動
- log 匯入

這會把「資料生成」和「服務啟動」耦合在一起。離線匯入本質上是 session build step，不是 runtime mode。第一版維持為獨立 CLI，責任邊界更乾淨，也更容易測試。

後續若真的需要 convenience command，再考慮在 `start.sh` 補一層薄 wrapper。

## 6. Session 匯入契約

### 6.1 輸入

第一版支援：

- 多個 `--source <tag>=<path>` 參數。
- `nwdaf` 應視為 v1 的必要來源。
- 其他來源先視為可選補充；就目前程式碼來看，`free5gc` 只補少量非核心拓樸事件。
- 每個來源是一個純文字 log 檔。

範例：

```bash
uv run python import_logs.py \
  --profile default \
  --source nwdaf=/data/run-01/nwdaf.log
```

### 6.2 輸出

匯入成功後建立：

```text
sessions/<generated-or-specified-session-id>/
  meta.json
  events.jsonl
  topology.yaml
```

### 6.3 `meta.json` 最低必要欄位

第一版應至少寫入：

```json
{
  "session_id": "20260507T000000123",
  "profile": "default",
  "grafana_groups": ["group-test-001", "group-test-002"],
  "start_time": "2026-05-07T10:00:00.000+08:00",
  "end_time": "2026-05-07T10:30:00.000+08:00",
  "event_count": 1234,
  "imported_from_logs": true,
  "log_sources": [
    {"source": "nwdaf", "path": "/data/run-01/nwdaf.log"}
  ]
}
```

其中 `imported_from_logs` 與 `log_sources` 為新增欄位，方便來源追蹤；replay 不必依賴它們。

## 7. 詳細設計

### 7.1 CLI 介面

目前已落地的 CLI 介面：

- `--profile <name>`
- `--nwdaf-log <path>`
- `--free5gc-log <path>`，選填
- `--session-id <id>`，選填；未提供時自動產生
- `--output-dir <path>`，選填；預設為 `sessions/<session_id>`
- `--lead-in-seconds <seconds>`，選填；預設 `5.0`
- `--strict`
- `--json`

目前沒有落地通用 `--source`、`--dry-run`、`--grafana-group` 這類較泛化參數，因為現階段真實需求已可由上述介面直接覆蓋。

```bash
uv run python import_logs.py --profile default --nwdaf-log /path/to/nwdaf.log
```

再把 `--free5gc-log` 保留為選填擴充。這個介面比通用 `--source` 更符合當前使用情境。

### 7.2 匯入流程

建議流程：

1. 解析 CLI 參數。
2. 驗證 profile 與 `profiles/<profile>/topology.yaml` 存在。
3. 驗證每個 `--source` 檔案可讀。
4. 為本次匯入設定 session id。
5. 呼叫 `rules.set_metric_session_id(session_id)`，確保 parser 相關 state 以新 session 為單位初始化。
6. 逐檔逐行讀取 log，呼叫 `parser.parse_line(line, source=tag)`。
7. 對成功 parse 的 event 收集到記憶體。
8. 根據 `event["time"]` 做全域排序。
9. 推導 `start_time` / `end_time` / `event_count`。
10. 建立暫存輸出目錄。
11. 複製 profile 的 `topology.yaml`。
12. 寫出 `events.jsonl`。
13. 寫出 `meta.json`。
14. 原子 rename 到正式 session 目錄。

### 7.3 為什麼需要全域排序

live 模式是靠到達順序 append，但離線匯入可能會同時讀多個檔：

- `nwdaf.log`
- `free5gc.log`（若有提供）

這些檔案之間的事件交錯順序不能假設與檔案讀取順序一致，因此匯入器必須以 event timestamp 做 merge/sort。前端 replay 也會排序，但輸出時先排序可讓：

- `events.jsonl` 更容易人工檢查
- `meta.start_time` / `end_time` 更可信
- replay 前後結果更穩定

### 7.4 落盤策略

匯入器不應直接寫正式目錄再半途失敗。建議：

- 先寫到 `sessions/.tmp-<session_id>-<pid>/`
- 全部成功後 rename 成 `sessions/<session_id>/`

若失敗：

- 保留 `--keep-temp` 作為未來除錯選項
- 第一版預設自動清理暫存目錄

### 7.5 解析統計

CLI 結束時應輸出統計摘要：

- 每個 source 總行數
- 每個 source 成功 parse 行數
- 每個 event type 數量
- 無法解析行數
- 缺少 `time` 或 timestamp 無效的 event 數量
- 最終輸出 session 路徑

這份統計對離線匯入特別重要，因為使用者通常會想知道「這批 log 到底轉到了多少事件」。

## 8. 模組拆分建議

### 8.1 新增模組

建議新增：

- `import_logs.py`

若邏輯稍大，再拆出：

- `session_import.py`

責任分工：

- `import_logs.py`：CLI 入口與參數解析
- `session_import.py`：純匯入流程與資料結構

### 8.2 應從 `main.py` 抽出的共用 helper

目前 `main.py` 已有一些可重用函式，但放在 runtime 檔案內不利於 CLI 共用。建議視需要抽出：

- session id 產生
- JSON 讀寫 helper
- `events.jsonl` 讀寫 helper
- ISO8601 timestamp parse helper
- profile topology 載入 helper

可新增共用模組，例如：

- `session_io.py`

目標不是大規模重構，而是把離線匯入真正需要共用的幾個 helper 抽出。

## 9. 與既有系統的整合點

### 9.1 與 replay 的整合

整合方式應該是零修改或極少修改 replay 主流程：

- replay 繼續只讀 session 目錄
- backfill 繼續只讀 event list
- frontend 無需知道該 session 是 live 錄製還是 log 匯入

### 9.2 與 Grafana 的整合

匯入器不需要直接寫 Prometheus。Grafana 相關資料仍在 replay 啟動時由既有 backfill 流程產生。

這個切分很重要，因為它讓匯入步驟保持純 artifact generation，不碰 runtime side effect。

### 9.3 與 topology 的整合

匯入器必須複製對應 profile 的 `topology.yaml` 到 session 內，否則 replay 時 topology event reactions 可能與原始環境不一致。

### 9.4 與 live side effects 的整合

雖然 replay 最終消費的是 event list，但 live 錄製出來的 `events.jsonl` 並不是純 parser 輸出。live pipeline 在 parse 之後，還會執行一小部分會反過來影響後續 event payload 的 stateful side effects。

目前最重要的例子是：

- `accuracy_policy` 在 parse 階段會依 `_POLICY_TOPOLOGY_STATE` 決定 `topology_suppressed`
- `model_swap` 在 live pipeline 的 metric handler 中會清空 `_POLICY_TOPOLOGY_STATE`

這代表「離線重新 parse 一遍 log」若只做 pure parse，而不重播這些 side effects，匯入結果可能會與原 live session 的 payload 不完全一致。

已知具體影響：

- 在 `model_swap` 後的第一批 `accuracy_policy` event，離線 parse 可能會產生與 live session 不同的 `topology_suppressed`
- 網頁上會反映成 topology 是否多畫一次 self edge / pulse 的差異

因此，匯入器的目標不應只是「從 log 解析出合理 event」，而是要盡量重建「live 錄製當下實際寫入 `events.jsonl` 的 payload」。

此問題已實作修正。現在 importer 在每筆 `parser.parse_line()` 成功後，會呼叫共用的 runtime state advance API；已知必要路徑至少包含：

- `model_swap -> clear policy topology state`

這讓 `model_swap` 後第一批 `accuracy_policy` 的 `topology_suppressed` 能與既有 live session 對齊。

## 9.5 為什麼 `nwdaf.log` 幾乎足夠

依目前規則覆蓋來看，replay 核心真正依賴的大部分事件都來自 NWDAF log：

- subscription chain 的大部分事件
- `upf_volume`
- `aggregated_slot`
- `ml_inference`
- `accuracy_scope`
- `accuracy_policy`
- `retrain_trigger`
- `retrain_done`
- `model_swap`
- ADRF 相關事件

相對地，`free5gc.log` 目前只補一類事件：

- `sbi_call`：SMF -> UPF subscription created

這代表：

- 若只提供 `nwdaf.log`，replay/Grafana/大部分 topology 行為仍應成立。
- 若再加 `free5gc.log`，主要是額外補完整 `SMF -> UPF` subscription edge。

因此 v1 規劃應以 `nwdaf.log` 為唯一必要輸入，把 `free5gc.log` 定位成 optional enrichment。

**後續設計判斷補充**：

經實際比對同一組 `log + live session` 後，目前 `free5gc.log` 的有效用途幾乎只剩：

- 補 `SMF -> UPF` 的 `Nupf_EventExposure_Subscribe` topology event

而先前的 `nf_up` 已確認屬於歷史殘留，且已移除，原因是：

- 只會把 `SMF` 節點標成 `up`
- 沒有對應的 `down` 或 reset 邏輯
- 對 Grafana、replay metrics、核心時間軸沒有實質價值
- 在 replay / import 場景中反而容易造成「節點從開頭一路亮到尾」的誤導

因此目前的設計決策應記錄為：

- `free5gc.log` 保留為 optional enrichment source
- 其短期價值只在 `SMF -> UPF` subscription edge
- `nf_up` 已視為歷史殘留並移除
- 後續不再為 `nf_up` 增加任何 importer 或 replay 特例

## 10. 風險與邊界條件

### 10.1 Parser coverage 不完整

若原始 log 含有目前 rules 尚未支援的 pattern，匯入器只能忽略那些行。這不是匯入器特有問題，而是整個 event model 的既有限制。

因應方式：

- 匯入摘要中明確回報未解析行數
- 保留 `--strict` 模式，讓高要求使用者在 coverage 不足時直接 fail

### 10.2 多檔時間戳不完全同步

不同 log 檔若由不同程序寫入，timestamp 可能存在小幅抖動。這是離線匯入無法完全消除的事實，但與現有 replay 模型相容，因為 replay 本來就依 event `time` 工作。

### 10.3 記憶體消耗

第一版使用「全讀入記憶體後排序」最簡單可靠。以目前 session 規模，通常可接受。若未來 session 非常大，再考慮外部排序或 streaming merge。

### 10.4 `topology_suppressed` 與 parser state

`rules.nwdaf` 目前有 session 內部狀態，例如 policy topology change suppression。匯入器必須在每次匯入開始前重設 session state，避免跨匯入污染。

### 10.5 live side effects 與 payload 對齊風險

目前已知 `rules.nwdaf` 不只是純 parser，它還和 live runtime 中的後續 handler 共享狀態。最明確的案例是：

- `_policy_topology_changed()` 會更新 `_POLICY_TOPOLOGY_STATE`
- `model_swap` 的 live handler 會清空 `_POLICY_TOPOLOGY_STATE`

若 importer 只逐行呼叫 `parser.parse_line()`，卻不重播這些 side effects，就可能出現：

- event 數量看起來一致
- event 數值欄位也一致
- 但部分 event payload 的旗標欄位不同

目前已觀察到的具體例子是：

- `model_swap` 後第一批 `accuracy_policy`
- live session 中 `topology_suppressed=true`
- pure parse 匯入結果則可能變成 `topology_suppressed=false`

這會讓 replay topology 比原 live session 多演出幾次 policy self edge / pulse。從 UX 角度看，這是重要差異，因為 `topology_suppressed` 原本就是為了防止 policy 類事件在畫面上重複刷出大量 self edge。

**最終實作方式**：

1. 在 `rules` 層抽出 importer 可共用的 runtime state advance API
2. 明確避免讓 importer 直接呼叫 `main._update_metrics()`
3. importer 在 `parser.parse_line()` 成功得到 event 後，立刻執行必要的 state-advance
4. 目前已知需要的 `model_swap` 路徑已被覆蓋

**為什麼不直接讓 importer 呼叫 `main._update_metrics()`**：

- importer 不應碰 Prometheus metrics 更新
- importer 只需要重播「會影響後續 payload 的狀態」
- 因此應把「payload-related state mutation」和「metrics side effects」拆開

**實際採用的程式結構**：

- `rules.nwdaf`
  - 保留 parser rule
  - 暴露 payload-related runtime state advance
- `rules.__init__`
  - 提供跨 rule 的 `advance_runtime_state(event)` dispatcher
- `main.py`
  - live mode 維持 metrics update
- `session_import.py`
  - parse 成功後
  - 依既定順序呼叫 state-advance API

**驗收標準**：

- 原本 4 筆 `accuracy_policy` 的 `topology_suppressed` 與 live session 對齊
- 重新匯入後，不再出現已知 payload 差異
- 網頁 replay 不會比 live session 多出額外的 policy self edge / pulse

### 10.6 僅用 `nwdaf.log` 的視覺缺口

若匯入時只提供 `nwdaf.log`，目前可預期會少掉：

- `SMF -> UPF` 的那條 subscription edge

這個缺口不影響 replay 的時間軸、event log、Grafana backfill 與大多數 NWDAF 相關 topology 互動，但應在文件與 CLI 輸出中明確告知使用者。

### 10.7 Session ID 與 replay metadata 不一致

若使用者手動指定 `--session-id`，必須保證：

- 目錄名
- `meta.json["session_id"]`
- event 所屬 session context

三者一致。

### 10.8 replay 開頭事件過於擁擠

若 `meta.start_time` 與第一筆 event 完全相同，replay 在按下 play 後會立刻連續噴出第一批事件，觀感上比 live 錄製 session 更急促。

此問題已透過 importer 預設 `lead_in_seconds=5.0` 修正：

- session `meta.start_time` 預設會比第一筆 event 早 5 秒
- replay timeline 會先經過一小段空白 lead-in
- 可透過 `--lead-in-seconds` 調整或設為 `0`

## 11. 驗證計畫

### 11.1 單元測試

至少補以下測試：

- 僅提供 `nwdaf.log` 時可正確轉出可 replay session
- 單一 log 檔可正確轉出 `events.jsonl`
- 多個來源檔案可依 timestamp 正確排序
- 無法解析行會被忽略並統計
- 缺少 `topology.yaml` / source 檔時會 fail
- 匯入 session 可被 replay 流程成功讀取

### 11.2 整合驗證

建議驗證流程：

1. 用一組已知可 replay 的 live session 作為基準。
2. 找出當次原始 log。
3. 以新匯入器生成另一個 session。
4. 比較：
   - event type 分布
   - 關鍵事件數量
   - event payload 是否一致（至少抽查 `topology_suppressed`、`policy_label`、`scope` 等 topology-sensitive 欄位）
   - `model_swap` 前後的 `accuracy_policy` 是否與 live session 對齊
   - replay timeline 起訖時間
   - Grafana 主要 panel 是否能正常 backfill 顯示

### 11.3 人工驗證項目

- `./start.sh --replay sessions/<imported_id>` 可正常啟動
- 前端 timeline 可看到完整事件
- topology 動畫與 state rebuild 正常
- Grafana panel 有資料
- `session-info`、`sessions` API 顯示 metadata 正確

## 12. 實作階段建議

### Phase 1：最小可用匯入器

狀態：已完成

- 新增 CLI
- 先支援 `nwdaf.log` 單檔匯入
- 產生完整 session 目錄
- 可成功 replay

### Phase 2：共用 helper 整理

狀態：已完成

- 抽出 session / JSONL / time parse 共用 helper
- 降低 `main.py` 與匯入器之間的重複

### Phase 2.5：optional source enrichment

狀態：已完成，但範圍已收斂

- 補 `free5gc.log` 作為選填輸入
- 補 `SMF -> UPF` 這類非核心拓樸事件
- `nf_up` 已在後續清理中移除，不再視為保留項

### Phase 2.6：live-payload fidelity 修正

狀態：已完成

- 修正 importer 與 live session 在 `topology_suppressed` 上的不一致
- 至少覆蓋 `model_swap -> clear policy topology state` 這條已知路徑
- 用同一組實驗的 `log + live session` 做 payload-level regression compare
- 將驗收標準從「事件數量接近」提升到「拓樸敏感欄位也對齊」
- 明確抽出 importer 可用的 state-advance API，避免直接耦合 `main._update_metrics()`

### Phase 3：可用性強化

狀態：部分完成

- `--strict` 已完成
- 匯入統計摘要已完成
- `--lead-in-seconds` 已完成
- `--dry-run` 尚未實作
- 更細緻的錯誤訊息仍可持續補強

### Phase 4：後續 convenience 層

狀態：未開始

- 視需要加 `start.sh --import-logs` wrapper
- 視需要補文件與操作指南

## 13. 建議的最終落地形態

第一版完成後，使用者心智模型應該是：

```text
live 實驗當下有開 5g-viz
-> 直接使用 sessions/<id>

live 實驗當下沒開 5g-viz，但保留了原始 log
-> 先跑 import_logs.py 生成 sessions/<id>
-> 再用既有 replay 模式播放
```

這個模型延續既有 session/replay 架構，不引入第二條資料產品線，是目前最穩定、最省變更、也最容易驗證的方案。

## 14. 實作決策摘要

- 功能歸類放在 `plans/dvr/`。
- 以「離線產生 session artifact」為核心，而不是修改 replay 格式。
- 第一版使用獨立 CLI，不修改 `start.sh` 主責。
- 最大化複用 `parser.py`、`rules/*`、session 目錄格式與 replay/backfill 流程。
- `nwdaf.log` 是必要輸入；`free5gc.log` 僅作 optional enrichment。
- `nf_up` 已移除，不再保留為 replay / import 支援項。
- importer 需重播最小必要的 live-side runtime state，不能只做 pure parse。
- imported session 預設加入 5 秒 replay lead-in，以改善開頭播放體驗。
- 驗收標準以「可直接 replay」為最終判準。
