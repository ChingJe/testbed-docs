# Hierarchical FL 論文實驗 Testbed 就緒盤點

日期：2026-09-21

狀態：Review Confirmed／Implementation Pending；待決策項目仍開放

## 1. 目的與判斷基準

本文盤點新版 testbed 是否已能執行論文所需的 E0、E1、E2a、E2b 實驗。判斷基準是：MNIST 與 CIFAR-10
各使用同一組五個 paired seeds，且同一 workload／seed 的四種情境固定資料分割、初始模型、local epochs、
FedProx／aggregation 設定及 accepted-round 總數。故完整矩陣最多包含四十個 runs。

Component contract、逐節點紀錄語意、訂閱生命週期、直接 Leaf reparenting 與 mixed-depth aggregation 以
`nwdaf-docs` 的論文實驗文件及目前 Go NWDAF／PyMTLF source 為準。Testbed 負責 component revision 整合、
設定生成、process placement、故障注入、run lifecycle、原始證據收集及離線分析交接。既有單次 E0／E1
MNIST／CIFAR-10 結果只作為 testbed pipeline 的歷史 baseline，不是這批五 seed 實驗結果。

本次 source 盤點確認 Go NWDAF 與 PyMTLF 已完成候選 payload 欄位改名、正式 subscription resource identity、
逐節點 structured evidence、直接 Leaf reparenting 及 mixed-depth training；本地 real-process runner 已跑通短程
E0、E1、E2a、E2b。這些結果尚未在實驗室四 VM testbed 驗收，因此不能直接宣稱正式實驗已可執行。

## 2. 整體結論

目前 testbed **不能直接執行正式 E0–E2b 五 seed 矩陣**。既有四 VM、network、NRF／ADRF、十一組
NWDAF↔PyMTLF placement、GPU／CPU 分工、資料工具與 lifecycle pipeline 可作基礎；主要缺口是新版 component
revision 尚未成為 parent repository 的 selected revision，設定 renderer 仍輸出舊拓樸格式，runner／evidence
checker 仍寫死 Branch replacement 舊事件與 participant 形態，且尚無五 seed 輸入及四情境離線分析能力。

## 3. 必須完成的 Testbed 調整

### 3.1 整合目前 component revisions

`5G_NWDAF_Infrastructure` 目前 parent revision 仍將 `NFs/nwdaf` 鎖定在 `be3fa57`，將 `ML/PyMTLF` 鎖定在
`bdbd2a9`；workspace 內兩個 nested repositories 雖已分別位於 `de8b385` 與 `c31b8b6`，parent 只呈現 modified
gitlinks，`components.lock.yaml` 也仍指向舊 commits。正式實驗前必須：

- 更新 parent gitlinks 與 component lock，讓乾淨 checkout 能重建目前已驗證的 component source；
- 重新建置並部署 Guest NWDAF binary 與 Host PyMTLF image；
- 重新產生 selected config，不沿用舊 generated config；
- 在正式 run 前確認 parent、NWDAF、PyMTLF 及實際 image／Guest artifact 都對應同一批已提交 revisions。

這是直接 blocker；只把 nested repository 留在較新 HEAD，不能形成可重建的 testbed 實驗版本。

### 3.2 更新 topology renderer 與 scenario contract

目前 `protocol_topology()` 固定輸出 `admission.mode: complete_required`，但新版 PyMTLF 已移除該輸入，並要求
拓樸最外層必填 `on_branch_failure`。因此以目前 renderer 配合新版 PyMTLF 產生的拓樸會在 native config
解析階段被拒絕。

Testbed 應沿用既有 `TESTBED`、scenario、renderer、checker 與 lifecycle pipeline，擴充而不是建立第二套流程：

- E1 使用 `on_branch_failure: replace_branch`；
- E2a／E2b 使用 `on_branch_failure: reparent_leaves_to_root`；
- E0 不注入故障，但仍須明確生成與其 paired comparisons 相容的 Root topology setting；
- 移除生成的舊 `admission`，不建立相容雙格式；
- 將「注入哪些 process failure」與「Root 選擇哪種修復方式」分開表達，避免以
  `fault.mode: branch-replacement` 同時承載兩種不同責任；
- 由 selected scenario 產生 E0／E1／E2a／E2b 差異，不複製另一套 renderer 或 runner。

E2a／E2b 形成 Root 直連 Leaf 後，Root 必須能取得 direct Leaf 結果。現有 artifact origin allowlist 只讓 Root
存取 Branch origins；renderer 須依已選 topology behavior 納入可能成為 Root direct children 的 Leaves，同時維持
ADRF 與原 Branch 路徑的既有權限。

### 3.3 將 fault lifecycle 擴充成情境導向

現有 runner 只解析一個 faulted Branch group，並只停止 A 的 Guest NWDAF 與 Host PyMTLF。新情境需要：

- E0：不注入故障；
- E1：在指定 accepted-round barrier 後停止 A；
- E2a：同樣停止 A，但由 Root 直接嘗試 A1／A2；
- E2b：停止 A，並讓 A2 在 Root 嘗試新訂閱前不可用；A1 保持可用。

E2b 不要求 A 與 A2 在完全相同時間停止，但必須各自保存實際 effective stop time，且 A2 必須在 Root 對它進行
direct reparent preparation 前不可用。Primary Branch 的 failure latency 以 A 的 effective stop 為起點；A2 的停止
是另一筆故障事實，不能合併成單一模糊 timestamp。

故障工具需以同一既有 fail-stop 方法處理多個明確 targets，保存各自的 Guest／container identity、freeze／kill
結果，並讓 cleanup 能恢復所有 runtime mask、restart policy 與 selected process state。不得為 E2b 建立繞過現有
provider guard、selected config identity 或 exact reset 的捷徑。

### 3.4 更新 runner 的 structured-event contract

現有 `PhaseTracker` 與 evidence checker 使用舊事件名稱與欄位，包括 `ROOT_ROUND_OUTCOME`、
`BRANCH_FAILURE_DETECTED`、`BRANCH_REPLACEMENT_READY`、`FINAL_MODEL_SAVED`、`validationLoss` 及
`validationAccuracy`，並把正常、degraded、restored participant set 固定為 Branch identities。

新版 PyMTLF 的直接來源改為：

- `MODEL_TRAINING_OPERATION`；
- `CANDIDATE_SELECTION`；
- `EDGE_CONFIRMED`／`EDGE_UNAVAILABLE`；
- `REPAIR_SELECTION`；
- `TOPOLOGY_ACCEPTANCE`；
- `ROUND_AGGREGATION`；
- `MODEL_EVALUATION` 的 `loss`／`accuracy`；
- `MODEL_ARTIFACT_SAVED`。

Runner 應依 selected scenario、Root policy 與事件中實際 direct children 解讀結果，而不是把所有 fault run 都判成
A→A*。E2a 修復後的 successful set 可包含 A1、A2、B、C；E2b 可包含 A1、B、C。實際 degraded round 數量
由 timeout 與修復時序決定，runner 不應預先要求固定輪數；它只需確認故障 barrier、accepted-round 對齊、
修復結果、首次 post-reconfiguration contribution 與 terminal model evidence 是否完整。

### 3.5 收集所有節點的原始紀錄

目前 runner 在訓練期間只增量讀 Root `observations.jsonl`，完成後再從少數 PyMTLF Docker logs 擷取 resource
字串。這不足以證明接收端實際收到的 Create／PATCH／Notify、正式 subscription resource ID、未受影響的 B／C
edges，以及 E2b 的 A2 未形成新關係。

正式 run 至少要保存：

- `run.json` 與 controller fault／lifecycle events；
- 每個實際節點自己的 `observations.jsonl`，包括已被停止的 A 或 A2；
- Root final model 與獨立 official-test evaluation；
- scenario snapshot、seed、資料配置、初始模型、component revisions、image／Guest runtime identity；
- 必要時才保存故障時間窗的精簡 diagnostics。

逐節點檔案應直接從各節點既有 private volume 收集，保持原始內容不變。Subscription、拓樸與 operation evidence
改由 structured records 解讀；Docker／journald logs 只作 diagnostics，不再作主要 protocol truth。正式 runs 前須
在多節點 testbed 確認 Slice 1 的 receiver-assigned `subscriptionResourceId` 能在發起端與接收端紀錄中正確對回
peer identity、`mlCorreId` 及 operation direction。

所有失敗、未完成或 `not recovered` runs 都須保留已產生的原始事件與模型曲線。Collection／analysis 失敗仍應能
以同一 run identity 重試，不得因此重新訓練。

### 3.6 補足五 paired seeds 的輸入與執行排程

目前正式 scenario 只有 seed 42，且每個 workload 使用一份固定的 checked-in seed model。五 seed 實驗需要先
確定五個 seed 的實際值及其語意，至少清楚控制：

- training／validation source indices；
- Leaf local-training random seed 與 minibatch order；
- 初始模型 weights；
- 同一 workload／seed 的 E0、E1、E2a、E2b 共用關係。

資料 generator 已可接受不同 partition seed，但現有共用 `datasetId` 不能讓不同 seeds 覆蓋同一目錄；需讓每個
workload／seed 有明確、可重建且可由四個情境共用的 dataset identity。初始模型目前未提供 testbed-owned 的
多 seed 生成與選擇流程，也須補上 seed-specific model preparation、component-native artifact import 及 scenario
selection。這項工作只使用 PyMTLF 已有的模型 artifact contract，不新增平行 hash／provenance 機制。

執行工具應保存 dataset、condition、seed 與 run identity 的一對一對應，支援從已完成或失敗的 run 邊界接續排程，
但不需要平行執行四十個 runs。單張 GPU 下以 serial execution 為預設，避免引入未控制的資源競爭。不得以四十份
手工重複且容易 drift 的完整 scenario 作為唯一管理方式；具體如何在現有 scenario source 表達 seed matrix，須在
詳細實作計畫決定。

### 3.7 擴充離線分析，而非修改 runtime 判斷

現有 `fl-analysis.py` 只接受一組 baseline／treatment，且依賴舊 event 欄位與 normal／degraded／restored 三分類。
新工具須能讀取兩個 workloads、四個 conditions、五個 paired seeds，並從保存的原始 evidence 離線產生：

- 每個 accepted round 的五 seed mean 與 95% CI；
- E1／E2a／E2b 相對同 seed E0 的 paired endpoint 差值；
- 預先固定 window 的 post-failure accuracy AUC；
- 依 E0 對應 round 95% CI 且連續兩個 accepted rounds 的 recovery 判定；
- failure→detection→repair instruction→edge ready→first accepted contribution 時間線；
- 各情境 reconfiguration 成功數及 `not recovered` 數；
- subscription resource Create／PUT／PATCH／DELETE 發起次數；
- requested、realized、accepted topology 與逐輪 participant set；
- E2b 的 participant／class coverage。

CI、AUC 與 recovery 都是離線分析，不應寫回 PyMTLF runtime acceptance。分析程式可以在正式訓練後繼續完善，
但原始資料 layout、必收事件與 run metadata 必須在第一個正式 run 前固定，避免因蒐集缺件而重跑長時間訓練。

## 4. 可沿用而不需重新設計的部分

下列既有 testbed 基礎可繼續使用：

- `core`、`path-a`、`path-b`、`path-c` 四台 VM、目前 network 與 process placement；
- MongoDB、NRF、ADRF，以及 Go NWDAF 位於 Guests、PyMTLF 位於 Host containers 的責任分工；
- Root 與六個 Leaves 使用同一張 GPU、Branches 使用 CPU 的 device placement；
- Root `minAvailableNodes=2`、`minTrainNodes=2`、`acceptFailures=true`、`minCompletionRate=0.66`，以及各 Branch
  要求兩個 Leaves 的現有 policy；
- FedProx `proximalMu=0.01`、`sampleWeighted` 與 Branch 每一輪回報 Root；
- 每 Leaf 8,000 筆、固定 2,000 筆 train-derived validation、完整官方 10,000 筆 test；
- MNIST 的五類／Leaf 配額與 CIFAR-10 的 all-class soft label-skew 配額；
- MNIST 24 accepted rounds／4 local epochs／第 12 輪後故障；
- CIFAR-10 40 accepted rounds／5 local epochs／第 20 輪後故障；
- 300 秒 preparation／round timeout、final model 保存、official-test evaluator；
- provider host-context guard、capacity check、selected config identity、stop／reset／recovery 與 collection retry。

Leaf template 的 `max_concurrent_jobs=2` 已可容納舊 parent subscription 尚未完全退出時的新 direct Root
subscription，不需為 E2 另加 Leaf process 或新的容量繞路。以目前需求看不需要新增 VM 或提高既有 Guest RAM；
正式執行前仍須使用當日 Host／GPU／storage capacity preflight，而不能以過去成功 run 取代。

## 5. 尚待確認的實驗決策

下列項目須在正式 runner 實作或四十個 runs 開始前確認：

1. 五個 paired seeds 的確切值，以及單一 seed 是否同時控制 partition、initialization 與 local-training order；
2. seed-specific initial model 的生成、model ID 及選擇方式；
3. 95% CI 的估計方法與 post-failure AUC common window `K`；
4. E0／E2 是否保留已部署但不被使用的 A* process。

第 4 項目前有兩種可行方式。正式情境文件要求 E2 不使用 A*，但未明確要求實體 process 必須不存在；component
本地短程驗證則在 E0／E2 未部署 A*。若正式 testbed 保留同一套十一組 process，只以
`reparent_leaves_to_root` 保證 E2 不建立 A* 關係，可以維持四條件的 VM／process placement 一致並減少動態
inventory 改動。若論文要主張 E2 中 A* 完全不存在，則 scenario、runtime inventory、NRF registration、Compose、
capacity 與 cleanup 都必須支援 condition-specific process sets。此項須先由使用者決定，不能由實作自行選擇。

E2b 目前只要求 participant／class coverage；若後續論文需要 per-class accuracy，可使用保留的 final model 與固定
validation／test data 進行離線評估。這不是啟動正式訓練前必須修改 PyMTLF 的理由。

## 6. 建議實作與驗證順序

後續可分成三個可獨立 review 的工作段落：

1. **Component integration 與 evidence compatibility**：更新 revisions／lock、topology renderer、artifact origins、
   新事件與逐節點收集，先確認 generated config 可由目前 component 解析。
2. **E0–E2b lifecycle**：加入 E2a／E2b scenario 與多 target fail-stop，在四 VM 上各跑一次短 MNIST E0、E1、
   E2a、E2b；直接核對 multi-node subscription identity、direct Leaf、mixed-depth aggregation、故障 cleanup 與
   final model，不用兩個 datasets 重複做相同接線驗證。
3. **五 seed 正式矩陣**：完成 seed-specific dataset／initial model、serial scheduling 與離線分析輸入；凍結 seeds、
   CI 與 AUC window 後，再依序執行 MNIST／CIFAR-10 的四十個正式 runs。

短程驗證只證明 testbed 整合路徑，不取代正式 rounds、資料量或五 seed evidence。只有四種情境在 real testbed
通過、原始 evidence 可完整收集與獨立重試後，才應開始長時間正式矩陣。
