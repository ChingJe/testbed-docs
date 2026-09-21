# Hierarchical FL 論文實驗 Testbed 實作順序與 Slice 安排

日期：2026-09-21

狀態：Implementation；Slice 1 已提交，Slice 2 計畫已確認並提交、待實作，Slice 3 尚未開始；本文只安排工作邊界，不授權正式實驗執行

依據：[Testbed 實驗就緒盤點](./Testbed%20Experiment%20Readiness%20Inventory.md)。本文把該盤點第 3 節的七項缺口合併為三個依序推進、各自交付 review 的實作 slice。Go NWDAF／PyMTLF 的協定及訓練行為仍由 component source 與 `nwdaf-docs` 擁有；本計畫只安排 testbed 的設定、部署、控制、證據與分析工作。既有單次 E0／E1 run 不計入新的五 seed 實驗。

## 1. 七項缺口與 Slice 對照

| 盤點項目 | 主責 Slice | 交付重點 |
| --- | --- | --- |
| 3.1 Component revisions | Slice 1 | 可重建的 selected revisions 與實際部署版本 |
| 3.2 Topology renderer／scenario contract | Slice 1；Slice 2 使用 | 新版 native topology 可解析；四種情境從同一設定流程選取 |
| 3.3 Fault lifecycle | Slice 2 | E0 無故障、E1／E2a 單一故障、E2b 多目標故障與完整 cleanup |
| 3.4 Structured-event contract | Slice 1；Slice 2 收集 | 保存 accepted-round／topology 原始事件，實際 participant 與修復結果事後判讀 |
| 3.5 逐節點原始紀錄 | Slice 1；Slice 2 驗收 | 收集與重試能力；四情境的跨節點證據完整性 |
| 3.6 五 paired seeds | Slice 3 | 每個 workload／seed 的資料、初始模型、run 對應與序列排程 |
| 3.7 離線分析 | Slice 3 | 四情境、五 seed 的可重算彙整，不改 runtime acceptance |

各 slice 延伸現有 `TESTBED` + scenario + generated config + 共用 renderer／checker／runner + collection／reset pipeline，不建立平行 deployment source 或第二套 experiment runner。每個 slice 開始前再將此安排細化為該 slice 的實作計畫與 end-to-end baseline disposition；未經 review 不直接擴張下一個 slice。

## 2. Slice 1：新版 Component 與 Testbed 設定、證據接通

目標是讓目前 testbed 能以可重建的 component 版本產生新版 PyMTLF 可接受的設定，並保存後續四情境分析必需的原始證據。此時不宣稱 E2 故障恢復已在四 VM 上跑通。

- 將 Go NWDAF／PyMTLF 的 selected revisions 與 parent gitlinks、component lock 對齊；以相同版本建置 Guest binary、Host image，重新產生 selected config，核對實際 runtime identity。
- 擴充現有 scenario／renderer／checker：以新版 `on_branch_failure` 取代過時的 generated `admission`；將故障目標與 Root 修復策略分開；讓 Root 在 E2 可存取實際 direct Leaf 的 artifact，同時保留既有 Branch／ADRF 路徑。
- 沿用四台 VM，依 selected scenario 啟動 process：E0／E2a／E2b 不啟動 A* 的 Guest／Host process，E1 啟動；情境啟動完成時確認一次未運行狀態，不新增各階段反覆證明 A* 不存在的驗證。
- 更新共用 runner／evidence checker 對新版逐節點 JSON events、`loss`／`accuracy`、model artifact 及實際 direct children 的解讀；不固定三種 participant set 或 degraded rounds 數量，以文字 log 為診斷而非主要協定證據。
- 延伸現有 collection checkpoint，在停止 process 後、reset 前保存每個實際節點的原始 `observations.jsonl`、controller events、scenario／版本資訊、final model 與評估結果；collection 失敗可在不重訓下重試，保留不完整 run。
- 在同一 lifecycle 內減少逐步重建 VM SSH 連線與重複狀態查詢；可獨立的 VM 準備、Guest 啟停、停止後原始紀錄收集及 Host volume 操作採受控並行，保留訓練／故障時序、final model／評估／reset 的依賴、失敗先停機與補收語意，以及跨 VM config activation、實際 runtime identity 和 guarded reset 的失敗邊界。具體改動、保留的檢查與驗證方式見 Slice 1 計畫。

交付與驗證：生成的 E0／E1／E2 topology 可由 selected component native config 解析；聚焦驗證新版事件與逐節點收集路徑；在 approved Host context 以一個短程無故障 run 核對部署版本、逐輪 evaluation、final model、collection、stop 與 selected reset。完整 run lifecycle 優化後須再用同一短程路徑確認 transport、並行失敗處理與原有 acceptance；既有成功 run 不代表修改後的 lifecycle 已通過。若要操作 real provider，必須維持既有 Host-context guard、process inventory、capacity 與 exact reset 防護。短程 run 只驗證接線，不算五 seed 正式結果。

## 3. Slice 2：E0–E2b 情境與故障生命週期

目標是讓同一條 runner 路徑依 selected scenario 完成四種情境，並保存真實四 VM 的替換、直接 Leaf reparenting 與部分修復證據供事後判讀；不在此 slice 加入五 seed 批次排程。

- 讓 E0 不注入故障；E1 在指定 accepted-round barrier 後停止 A 並由 A* 接手；E2a 停止 A 後讓 A1／A2 直掛 Root；E2b 停止 A 且讓 A2 在 Root 新訂閱前不可用，只由 A1 嘗試直掛。
- 沿用現有 fail-stop／cleanup 機制擴充多個明確 target，分別記錄 effective stop time、Guest／container identity 與操作結果；失敗或中斷時也須恢復所有受影響的 process state／restart policy，並保留可審查的 run evidence。
- 保存足以事後核對 requested、realized、accepted topology、receiver-assigned subscription resource identity、`mlCorreId`、B／C 未受影響的 edges、逐輪參與者及首次故障後貢獻的原始紀錄；E2b 的 A2 停機與新訂閱時間線也須可追溯。Runner 不因修復效果而拒收已完成的 run。

交付與驗證：四個短程 MNIST 情境各在實驗室 testbed 執行一次，採 4 local epochs、8 accepted rounds，故障情境在完成 2 輪後注入；確認完整啟動、指定停機、訓練完成、逐節點收集、final model、停止與 exact reset。實際修復或 `not recovered` 僅在收集後判讀，不是 runner 的成功條件，也不假設固定 degraded rounds。短程測試不取代正式 MNIST／CIFAR-10 訓練。每個 run 之前仍須核對當日 Host／GPU／storage 容量與 selected runtime。

## 4. Slice 3：五 Seed 配對輸入與離線分析就緒

目標是讓正式矩陣可重建、可逐次執行，且所有訓練資料一經收集便能離線分析。此 slice 的實作完成不等於四十個正式 runs 已完成。

- 在既有 scenario／dataset／model preparation 流程中支援確定的五 seed 設定；同 workload／seed 的 E0、E1、E2a、E2b 共用資料 source indices、初始模型、訓練隨機性語意與超參數，並保持四個獨立 run identities。
- 為不同 seeds 提供不互相覆寫、可由同 seed 四情境共用的 dataset identity；建立 seed-specific initial model 的準備、選擇與 component-native import 路徑，不複製新的模型格式或平行 provenance 證明。
- 以現有 run 入口按 selected workload／condition／seed 序列執行；保存明確對應及 checkpoint，使 collection／analysis 可重試、失敗 run 可識別且不會被當成有效配對。單 GPU 預設不並行訓練。
- 從保存的原始資料離線計算逐輪五 seed mean／95% CI、同 seed E0 的 paired 差值、預定 window 的 accuracy AUC、連續兩個 accepted rounds 的 recovery、故障／修復時間線、成功與 `not recovered` 數、subscription operations、topology／participant 及 E2b coverage。

交付與驗證：用不同 seeds 的小型資料準備結果核對差異、同 seed 四情境核對共用輸入，再用 Slice 2 留下的原始 run 資料核對事件解讀、缺失資料處理及原檔唯讀性；單 seed 短程資料不能驗證五 seed CI 的數值。第一個正式 run 前須凍結原始資料 layout、必收事件、run metadata 與統計定義。完成 review 後另行確認正式矩陣的執行批次，才依序跑兩個 workloads × 四情境 × 五 paired seeds；任何失敗或不完整 run 保留並單列，不靜默補值或剔除。

## 5. 必須先確認的決策與停止點

1. **Slice 1／2 topology 決策已確認**：四情境複用四台 VM；A* 可預先安裝，但只在 E1 啟動，E0／E2a／E2b 不啟動。詳細計畫須讓現有 scenario、selected process inventory、registration、Compose、capacity 及 cleanup 對此一致；情境啟動完成時觀測一次未運行狀態，不增設到處重複的負面驗證。
2. **Slice 2 Root policy 與 fault 邊界已確認**：四情境沿用 `TESTBED` 目前與 `nwdaf-resources` 一致的 Root policy；PyMTLF 以當輪實際 selected direct children 計算 completion。Scenario 的 `fault` 只指定停機時機及有序節點，不設修復達標輪數；runner 只守執行與收集契約，selected／successful／failed identities、實際 topology 及 `not recovered` 事後分析。E2b 多目標停機及 60 秒 round timeout 見 Slice 2 計畫。
3. **Slice 3 seed／分析決策**：五個 seed 值、各 seed 控制的隨機來源、seed-specific model ID／生成方式、95% CI 方法與 post-failure AUC common window `K`，都須在實作相應來源或正式執行前確認。

上述事項若尚未決定，該部分保持 open；不以短程接線成功宣稱整個 slice 或正式實驗完成。若實作盤點發現需新增 config source、service、VM、external dependency，或改變 component contract、部署 ownership、destructive scope／驗收門檻，先更新計畫並請使用者決策。
