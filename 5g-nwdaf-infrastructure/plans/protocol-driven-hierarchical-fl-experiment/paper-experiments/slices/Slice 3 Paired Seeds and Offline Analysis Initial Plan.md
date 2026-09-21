# Slice 3：配對 Seed 輸入與離線分析初始計畫

日期：2026-09-22

狀態：Review Confirmed；兩個不同 seed 的隔離短跑與最終檢查已完成，使用者已確認 review；正式矩陣待另行授權

上層依據：[實作順序與 Slice 安排](../Testbed%20Implementation%20Sequence%20and%20Slices.md)與
[Testbed 實驗就緒盤點](../Testbed%20Experiment%20Readiness%20Inventory.md)。
[Slice 2 計畫](./Slice%202%20Scenario%20Fault%20Lifecycle%20Initial%20Plan.md)已記錄 E0、E1、E2a、E2b 在四台 VM 上各一次的
MNIST 短跑；這些是故障生命週期接線證據，不計入正式五 seed 結果。

## 1. 目標與範圍

沿用目前 `TESTBED`、scenario、generated config、dataset／model preparation、共用 runner、原始資料收集與
guarded reset 流程，準備兩個 workload × 四個 condition × 五個 paired seeds 的正式實驗輸入、逐次執行方式及
離線分析。Slice 3 的實作交付是「可重建、可執行、可分析」的能力；四十個正式 runs 須在本 slice review 後，
另行確認執行批次，不因建立本計畫而自動開始。

同一 workload／seed 的 E0、E1、E2a、E2b 應共用資料來源索引、初始模型、訓練隨機性語意與超參數，
但保有各自的 run identity 與原始結果。不同 seed 的資料與模型輸入不得互相覆寫。單 GPU 預設循序訓練；
不新增平行 deployment source、第二套 runner、模型格式或只為 provenance 而建立的衍生證明。
本批五個 seed 使用 `1–5`；兩個 workload 因此各需五份生成後的資料切分，同 seed 的四情境共用一份，
合計十份資料切分與四十次訓練。每個 workload／seed 也須有一份以該 seed 真正產生的初始模型權重，
合計十份初始模型；只修改 seed metadata 或沿用另一 seed 的權重不符合配對輸入要求。

## 2. 現有流程盤點與 baseline disposition

本輪只讀取目前 `5G_NWDAF_Infrastructure` source、PyMTLF native source，以及 Slice 2 四個已保存的 MNIST 短跑
`run.json`／部分 `events.jsonl`；沒有執行 provider、重新產生資料、啟動訓練或統計五 seed 結果。下表記錄現況與
所需適配；已確認的實作方向見第 3 節，尚未核對的技術契約仍保持 open。

| 邊界 | 直接證據與目前行為 | disposition／Slice 3 所需 |
| --- | --- | --- |
| Scenario 選取與 config 生成 | `config-render.py` 讀單一 `--scenario` YAML，將選中內容寫入 `manifest.yaml`，並生成 PyMTLF／Compose 設定；`resolve_config_scenario()` 會再次從 `manifest.scenario.definition` 讀來源。現有基準 YAML 只寫 seed 42、固定 `datasetId`、`seedModelId` 及 `seedArtifactKey`。論文選用的 CIFAR-10 是 `all-class-skew-baseline/replacement.yaml`，不是歷史的五類／Leaf `formal-baseline/replacement.yaml`；目前只有 MNIST 的 E2a／E2b 短跑 YAML，沒有 CIFAR-10 E2a／E2b 定義。 | **適配**：維持既有 `TESTBED`、scenario、`config-create` 與 renderer／checker 單一管線，讓一個 workload／condition 定義可選 seed `1–5`，並在生成設定後保留可再次讀取的實際選中 scenario；不複製四十份手寫完整 YAML。MNIST E2a／E2b 從現有短跑延伸；CIFAR-10 E2a／E2b 必須沿用論文選中 all-class-skew 的資料配額與正式訓練參數，不能誤用歷史 `formal-*`。 |
| Dataset 生成與掛載 | `image_dataset.py` 用 `partition.seed` 選 official-train 的 validation 與各 Leaf source indices，manifest 保存 `sourceIndices`；official-test 作 held-out。`partition.datasetId` 決定 `.generated/image-datasets/<datasetId>`，現有目錄若存在會按 selected scenario 核對再複用。`dataset.py` 的 image 路徑已轉給此工具；`experiment-start.sh` 會在 process start 前呼叫 generate。Compose 從 selected dataset 根目錄掛 Root validation 與各 Leaf shard。 | **複用＋小幅適配**：同 workload／seed 四情境指向同一 `datasetId`，不同 seed 有不同目錄；來源下載 cache 可複用。不同 seed 的 official-test 內容相同，是否各自保存 held-out 副本沿用現有生成方式即可，不另設驗證或新 cache。 |
| Leaf 訓練隨機性 | `config-render.py` 將 `partition.seed` 寫入每個 Leaf 的 `federated_learning.client.training.random_seed`；PyMTLF `FederatedTrainer` 以該值設定 NumPy／PyTorch 與 DataLoader shuffle generator。 | **複用**：同 seed 四情境的配置種子相同；不同 seed 配置值不同。這是隨機來源的控制契約，不把 GPU 執行結果承諾為逐 byte 相同。 |
| 初始模型來源與匯入 | 現有 `ML/PyMTLF/seed_models/image_classification/<workload>/` 各只有一份 `config.json`、`model.py`、`model.npy`；`initialization_seed` 是 metadata，匯入直接讀 `model.npy`，沒有五 seed 生成器。Dockerfile 將來源打進 image；renderer、`config-check.py`、`ml-compose-check.py` 與 `manifest.seedRestoration` 都以固定 workload 路徑為準；Root entrypoint 使用 PyMTLF native import 並核對 native `artifact_key`。`configlib.image_scenario_contract()` 目前固定 MNIST／CIFAR-10 的 seed model ID 為 1001／1002。 | **適配**：用 PyMTLF 現有 `Model` 架構，按選中 seed 真正生成 `model.npy`，再沿用 native bundle／import 及其既有 artifact identity；將 seed-specific source、生成設定、checker、Root container 可讀路徑和 reset 後重新匯入一起接通。只改 metadata、只換 dataset 或只改 seed ID 都不足。是否維持每 workload 的既有 model ID、以不同權重／artifact key 區分 runs，須在實作方案中確認；目前 native catalog 只要求同一配置內的 model IDs 唯一。 |
| Build、deployment 與啟停 | `experiment-start.sh` 先 generate／reuse dataset，再呼叫 `services-start.sh` stage selected config 到 Guests、啟動 Guest services 及 Host Compose；`fl-experiment-run.py` 執行既有 preflight、runtime start、ready observation、訓練及 final collection。現有 image 內建固定 seed source，現有 reset 清理 runtime state，未清除 Host 的生成資料。 | **複用＋模型來源適配**：四台 VM、process placement、provider guard、image revision／capacity admission、跨 Guest config activation、stop／guarded reset 不改語意。每 seed 的模型必須在 Root import 前可見，且 selected config、checker、entrypoint 與實際來源一致；若用 Host-generated source 掛載，無須為每個 seed 重建 image，但須確認 Root-only 唯讀 mount、來源可用性與 reset 後匯入。 |
| 單次 run 與故障生命週期 | `fl-experiment-run.py` 每次執行一個 selected config，在 `runs/protocol-hierarchical/<workload>/<runName>/` 建新目錄及 UUID request；`run.json.scenario` 已保存 selected workload、partition seed、dataset ID、model key、訓練及 fault snapshot。檔案鎖避免同時跑兩個實驗；成功路徑收集 final model、逐節點 JSONL、held-out，停止 process 後 guarded reset。失敗資料保留；`--collect-only` 可以針對同一 selected config／image 補收。 | **複用＋執行層適配**：四十次仍走同一 runner，逐次執行且每次有獨立 `runName`。增加只負責逐個呼叫現有設定與單次 runner 入口的薄工具；失敗即停，補收及處理失敗須在切到下一 selected config／model source 前完成，不能把補收當成重訓或把失敗 run 視為有效配對。不建立第二套訓練 runner。 |
| 原始紀錄與離線分析 | Slice 2 四個 MNIST 短跑均為 `successful`／`finalized`，保留 `run.json`、`events.jsonl`、各節點 `observations/*.jsonl`、final model 與 held-out 結果。Root `MODEL_EVALUATION` 使用 `accuracy`／`loss`、`ROOT_INITIAL` 與 `ROOT_GLOBAL`；底層 `roundInd` 從 0 起算，論文 accepted round 須按 `run.json.phases.rounds` 的接受順序從 1 起算，保留原 `roundInd` 以供對照。`run.json.phases` 只有 `beforeFault`／`afterFault`，不判修復成敗。現有 `fl-analysis.py` 只吃 baseline／treatment 一對，仍讀舊 `validationAccuracy`／`validationLoss` 與 normal／degraded／restored 分類。 | **適配既有分析入口**：由 saved raw evidence 按 workload／seed／condition 配對，讀新欄位與拓樸／operation events，計算多 seed 統計；不重寫原始 `run.json` 或讓 runner 以 recovery 判斷成功。四個短跑只用來檢查解析路徑與缺失處理，不驗證五 seed 統計數值。 |

這個 end-to-end 路徑仍需在實作設計中補上 seed source 的 exact lifetime／cleanup、selected scenario snapshot 的讀取方式、
以及批次失敗後的操作界面。若這些要求導致新人工 config source、service、VM、external dependency、component contract 或
destructive scope／驗收門檻變更，先回填方案並請使用者決策。

## 3. 已確認的最小實作方向

1. **選取一個實際情境**：保留每個 workload 的 E0／E1／E2a／E2b 基本情境，在既有 `config-create` 管線加入
   seed 選取，將選中的有效 scenario 保存在 generated config 中供 renderer、checker、dataset generator、runner
   一致讀取。`partition.seed` 同時控制資料與 Leaf 訓練種子；`datasetId` 由 workload／seed 對應到一份穩定輸出。
   這比四十份手寫情境更不易 drift；實作時須確認 generated scenario snapshot 如何保留原始定義與選中值，
   不能讓 manifest 指向會被改寫的來源。MNIST E2a／E2b 的正式情境可由現有短跑故障語意延伸；
   CIFAR-10 E2a／E2b 應從論文選中的 all-class-skew 資料配置與正式訓練參數出發，接上 Slice 2 已驗證的
   故障語意，而非使用歷史五類／Leaf 的 `formal-*`。
2. **準備十份真正不同的初始模型**：以目前 PyMTLF `model.py` 與 `config.json` 的模型參數為唯一架構來源，
   在 testbed 既有 Host preparation 流程按 seed 初始化 PyTorch model、輸出 component 已接受的 `model.npy`，
   並以 PyMTLF native bundle／repository 取得相應 `artifact_key`。把可重建的來源留在 testbed
   `.generated` 下，讓 Root container 唯讀掛載選中來源；同 workload／seed 四情境使用同一來源，不為
   每 seed 重建 image。這是已確認方向、不是已存在的功能；必須同步調整 renderer、Compose、checker、
   `seedRestoration` 與 entrypoint source 選擇，且不得另建 testbed-owned hash 證明。生成來源由 Host
   持有並跨同 seed 四情境保留；每 run 的既有 guarded reset 只清除所選 runtime state，不順手刪除這十份
   輸入。Native catalog 在單一 runtime 配置內要求 model ID 唯一；既有 reset 會清空模型相關 runtime state，
   因此建議先沿用每 workload 的既有 model ID、由不同 source／native artifact key 識別 seed，
   不為十個 seeds 額外分配 model IDs；實作前再核對這個跨 run 重用是否符合整條 model lifecycle。
3. **以薄工具逐次執行**：增加一個簡單工具，依 workload → seed → E0／E1／E2a／E2b 選取組合，
   對每個組合逐個呼叫既有 `config-create` 與 `fl-experiment-run` 入口；每次都使用獨立 `runName`。
   工具不複製設定生成、啟停、訓練、故障、收集或 guarded reset 邏輯，也不自行判定修復達標。
   前一 run 完成既有 collection／reset 才進下一個；任何入口失敗即停止序列並保留已產生的 evidence，
   由操作者使用同 config 的既有 `collect-only` 與安全 cleanup 路徑處理，再明確決定重試或從未完成組合接續。
   不增加自動補值、自動略過失敗 run 或另一套 queue／state database。run 對應以 saved scenario 與
   `runName`／request ID 為準；condition label 如何明確保存，須在選取介面設計時一併確定，
   不靠自由文字檔名猜測。
4. **離線彙整，不碰訓練判斷**：適配既有 `fl-analysis.py` 讀新 `MODEL_EVALUATION` 與多 run input，
   只從已保存的 `run.json`／原始 JSONL 產生表與圖。對完整、可配對的五 seed 計算逐輪 accuracy／loss、
   endpoint 及其同 seed E0 配對差值；五筆 endpoint 差值另計平均與 Student-t CI。由 `NODE_PROCESS_STOPPED`、Root／各節點 operation、edge、topology 及
   aggregation events 建立時間線與修復判讀。Subscription API-call 計數須限定發起端 `SENT`，避免
   將對端 `RECEIVED` 再算一次。完整但未修復的 run 可報 `not recovered`；中斷、未完成或缺失資料的 run
   另列，不補值、不默默剔除，也不在 runner 增設修復門檻。現有兩 run 的歷史分析入口不得在沒有
   明確 replacement 決策下被悄悄破壞；新增多 run 分析時先確認能否在同一工具保留舊模式，
   不為舊模式增加與本實驗無關的維護或驗證。

上述第 1–3 點的方向已由使用者確認；generated scenario snapshot、模型 ID／native artifact lifecycle、
condition label 與工具接續介面的具體契約仍須在實作前核對，不把尚未驗證的細節當成既有能力。
第 4 點的統計公式可晚於資料收集程式完成，但原始欄位、condition／seed 對應與必收事件必須在首個正式 run 前固定。

## 4. 預計交付與驗證方向

- **配對輸入**：採用 seed `1–5` 並確認其控制範圍；同 workload／seed 的四情境選用相同資料與初始模型，
  不同 seed 使用互不覆寫的 identity。以小型資料準備核對「同 seed 共用、不同 seed 有差異」，不把特定值寫成
  所有 scenario 的通用合法性限制。模型掛載／匯入路徑更動後，建議只用兩個不同 seed 的短程 MNIST E0
  在 approved Host context 直接核對切換、native import、訓練、收集與 reset；Slice 2 已跑過的 E1／E2
  故障路徑不因 seed plumbing 重跑四遍。若 focused evidence 顯示故障路徑受影響，再擴大驗證。
- **執行交接**：沿用 `fl-experiment-run` 及既有 collection／reset；明確保存 workload、condition、seed 與
  run identity 的對應。失敗或不完整 run 保留並單列；收集及分析可重試，不因此重訓或靜默補值。
- **離線分析**：從原始紀錄重算逐輪五 seed mean／雙側 95% Student-t CI、同 seed 相對 E0 的 endpoint paired 差值及其平均／CI、故障後 accuracy AUC、
  recovery、故障／修復時間線、subscription operations、topology／participant 與 E2b coverage。
  這些結果不寫回 runtime acceptance；先用 Slice 2 原始資料確認事件解讀、缺失資料處理與原檔唯讀性。

單 seed 短跑只能驗證資料讀取與分析流程，不能證明五 seed CI 的數值正確。正式執行前還須固定原始資料 layout、
必收事件、run metadata 與統計定義，並重新核對當時的 selected runtime、component revision 與 Host／GPU／storage 容量。

## 5. 目前討論結論與建議

- **已確認**：十份資料切分分屬兩個 workload × 五個 seed；四情境共用同 workload／seed 的資料與真正由該 seed
  產生的模型權重。官方原始資料可沿用各 workload 的既有 cache；十份是生成後的切分，不是十次下載。
- **資料 identity 與模型來源已確認方向**：沿用 `partition.datasetId`，以 workload／seed 區分生成目錄，
  例如 `mnist-formal-s1`；由現有 `config-create` 選 seed 並保存實際 scenario snapshot。Host 按 seed 準備初始
  權重，Root 唯讀掛載後使用 PyMTLF native import；示例名稱不構成通用 schema 規則，model ID 跨 run
  重用仍須核對完整 lifecycle。
- **逐次執行已確認方向**：依 workload → seed → E0／E1／E2a／E2b，由薄工具逐個呼叫現有設定與單次
  runner 入口；前一 run 完成收集與 guarded reset 後才切換。失敗即停，保留 run 並單列；操作者處理後
  再從未完成組合接續，不新增另一套 runner。
- **結果呈現方向**：逐輪呈現五 seed 平均 accuracy／loss 與變異範圍，並列最終表現、相對同 seed E0 的
  paired 差值和修復情形。上層實驗定義要求 E0 對應輪次的 95% CI 作為 recovery 參照，故曲線建議標示
  95% CI 並清楚註明樣本數為五；CI 方法已確認為雙側 Student-t。
- **AUC 視窗已確認**：此處指故障後 accuracy 對 accepted-round 曲線的面積，不是分類器的 ROC AUC。
  依老師草稿先採「故障後預先固定的 common window `K`」作為方向，E0 以相同名義故障邊界對齊；
  不把 round 1 起算的全程 AUC 或整段剩餘 rounds 擅自替代此定義。先前提出的 `K=8` 缺乏實驗理由，
  已撤回。目前選用的 MNIST E0／E1 source 是 24 輪、第 12 輪後故障，故障後有 12 輪；
  CIFAR-10 all-class-skew E0／E1 source 是 40 輪、第 20 輪後故障，故障後有 20 輪。
  E2a／E2b 目前只有 8 輪短跑情境，正式情境尚須沿用各 workload 的輪數與故障點建置。
  **已確認**：兩個 workload 共用 `K=12`，分別計算 MNIST 第 13–24 輪與 CIFAR-10 第 21–32 輪；
  E0 以對應的名義故障邊界對齊。CIFAR-10 第 33–40 輪仍呈現在完整曲線與 endpoint，但不進入主要 AUC。
  分析程式不得按每次 run 的實際長度臨時改變視窗。
- **CI 方法已確認**：依論文草稿與本次決策，逐輪 accuracy／loss、endpoint 與故障後 AUC 的五 seed 彙整
  採雙側 95% Student-t CI；五筆觀測值使用樣本標準差與自由度 4，保留各 seed 原始數值。
  Recovery 依上層定義，參照 E0 對應 accepted round 的同一方法所得 CI，且連續維持兩輪。
  Endpoint paired effect 先在每個 seed 計算情境與 E0 的差值，再對五筆差值計算平均與同一方法的 CI；
  不把跨情境未配對的平均值差當成 paired effect。
- **Run 紀錄分類已確認**：目前 runner 固定寫入
  `runs/protocol-hierarchical/<workload>/<runName>/`，同一 workload 的短跑、失敗紀錄與正式實驗會平鋪混雜。
  正式矩陣的原始 run 應能按實驗系列、workload、seed、condition 定位，且仍以 `runName` 為單次 run 目錄；
  新 run 使用 `runs/protocol-hierarchical/e0-e2b/<workload>/seed-<n>/<condition>/<runName>/`。
  此目錄只供分類，不取代 `run.json` 的實際 scenario／seed 證據；既有 run 保持原位，
  不為整理新實驗而搬移或刪除。實作時須讓單次 runner、`collect-only` 與離線分析使用同一選定路徑。

## 6. 實作前核對與停止點

1. **方向已確認，實作前核對**：既有 `config-create` 選 seed，generated scenario snapshot 保存實際選中值，
   同 seed 四情境共用輸入；第 7 節已有接線建議，實作時須核對所有 consumer 都讀到有效值，以及 E2a／E2b 正式 scenario 的配置。
   不直接以四十份手工完整 YAML 取代。
2. **方向已確認，實作前核對**：模型生成由 testbed Host preparation 擁有，Root-only 唯讀掛載選中來源；
   尚須核對既有 model ID 跨已 reset runs 的重用、native artifact key 的選取方式及 reset 後重新匯入；
   若需改 PyMTLF component contract，須另經 component policy 與使用者決策。
3. **方向已確認，實作前核對**：第一版增加只逐個呼叫現有 `config-create`／`fl-experiment-run` 的薄工具，
   失敗即停並由操作者處理後接續；第 7 節建議 condition 從 selected scenario 進入 run evidence，實作時須核對
   組合選取、失敗後補收與同 config 接續。不新增第二套 runner 或自動恢復機制。
4. **正式執行前**：CI 算法及 endpoint paired 差值的 CI 已固定為雙側 95% Student-t，
   故障後 AUC 共用視窗已固定為 `K=12`；第 7 節提出離散面積、recovery 與缺失 run 的計算口徑。
   正式 E2a／E2b 仍須沿用各 workload 的輪數與故障點。統計計算屬後處理，不進入訓練期間的接納判斷。
5. **目錄布局已確認，實作前核對接線**：正式 run 使用上述 `e0-e2b` 分層，與短跑、既有歷史紀錄分開；
   單次 runner、`collect-only` 與離線分析須指向同一 run 目錄。失敗 run 留在原目錄，不按狀態搬移；
   不另建一份 run 索引或複製原始資料。

使用者已授權依第 7 節方向進入實作；實作時須核對各接線點與 end-to-end baseline disposition，若既定邊界不成立則停止並回到決策。
本階段不執行正式矩陣，也不以 Slice 2 的四次短跑代替正式比較結果。

## 7. 第二輪接線盤點與建議

本節是依實作前的 source 與已保存短跑資料提出的 **Slice 3 實作細節**，不是已完成能力。盤點時未啟動 provider、container、資料產生或訓練。

| 接線點 | 已核對的現況 | 建議的最小接法 |
| --- | --- | --- |
| 有效 scenario | `config-render.py` 先載入 YAML 才 render，`manifest.scenario` 已複製 image scenario 的主要欄位；但 `resolve_config_scenario()` 目前只用 `definition` 重讀原始 YAML，因此後續 dataset、checker、runner 不會自然取得選中 seed。 | 在現有 `config-create` 加 seed 選擇，只對選中 image scenario 改 `partition.seed`、穩定的 per-seed `datasetId` 與模型來源／native key；把完整有效欄位存入 `manifest.scenario`。對 protocol-hierarchical 的後續 consumer，以該快照作為 selected scenario；`definition` 僅保留原始檔案的 provenance。其他 deployment kind 保持原解析方式；不另建四十份 YAML 或平行 config source。Seed `1–5` 是本批選擇，不寫成通用 scenario 合法值限制。 |
| Condition 與正式 scenario | MNIST E2a／E2b 現有檔案是 8 輪、200 筆 validation／test 的短跑；CIFAR-10 沒有 E2a／E2b。MNIST 正式 E0／E1 為 24 輪、故障界線 12；選中的 CIFAR-10 all-class-skew E0／E1 為 40 輪、故障界線 20，現有 `kind` 為 `diagnostic`。 | 在各 workload 的四份基本情境明確保存實驗 series／condition，例如 `experiment.series: e0-e2b` 與 `experiment.condition: E2a`，由同一 scenario 快照傳到 `run.json`；不從檔名、`kind` 或 fault 型態猜 condition。正式 E2a／E2b 沿用所屬 workload 的正式 partition、training、故障界線，僅接上已驗證的直掛與停機語意；既有短跑保留原位。 |
| 初始模型 | 現有 Root image 內只有各 workload 一份固定 seed source；entrypoint 原生匯入後核對 `artifact_key`。`config-check.py`、Compose checker、`seedRestoration` 與 Root 環境變數皆假定固定路徑。既有 reset 清空 PyMTLF runtime volume，不刪 Host 生成輸入。 | 由 `config-create` 所屬 Host preparation 以選中 seed 和現有 PyMTLF `Model` 架構真正生成權重，保留 per-workload／seed source 於 `.generated`；用 PyMTLF native bundle 取得 key。只在 Root container 唯讀掛載選中 source，讓 renderer、兩個 checker、`seedRestoration` 與 entrypoint 指向同一位置；每次 reset 後由 Root 再匯入。不新增 testbed-owned hash。先沿用既有 workload model ID，因每 run 的 runtime catalog 在 guarded reset 後清空；focused verification 必須實測兩個不同 seed 順序切換與重新匯入，不能只憑單一配置的 ID uniqueness 宣稱跨 run 成立。 |
| 執行與補收 | 單次 runner 和 `collect-only` 都由 selected config 加 `runName` 算同一路徑；目前路徑固定平鋪在 workload 下，且 `check_evidence()` 要求末層名稱等於 `runName`。 | 若 selected scenario 屬於正式 `e0-e2b` 系列，兩個入口都使用 `runs/protocol-hierarchical/e0-e2b/<workload>/seed-<n>/<condition>/<runName>/`；其他 scenario 仍用舊路徑。薄工具只依操作者選定的 workload／seed／condition 順序呼叫現有 `config-create` 和單次 runner，為每次嘗試給獨立 `runName`。失敗立即停止；同一 selected config 下先補收或安全清理，後續由操作者明確選擇未完成組合重呼叫，不做自動跳過或 queue state。 |
| 分析輸入 | `events.jsonl` 的 Root `MODEL_EVALUATION` 真正欄位為 `payload.accuracy`／`loss`；`run.json.phases.rounds` 保存 accepted 順序，`ROUND_AGGREGATION` 保存直接參與者。現有 `fl-analysis.py` 還讀舊欄位與不存在於新 run 的 `normal`／`degraded`／`restored` phase，不能直接分析本批資料。E2b 的 per-leaf `classHistogram` 在生成資料的 `split-manifest.yaml`，不在已保存的 run 原始檔內。 | 在既有分析工具加入正式系列的多 run 模式，從 `run.json.scenario.experiment` 配對，依 accepted 順序對上 Root validation，不修改原始事件；舊兩 run 入口保持原狀。為讓 E2b class coverage 隨 run 可重算，run 建立時只保存選中 split manifest 的 per-leaf 樣本數／class histogram 摘要，不複製完整 source indices 或資料檔。正式系列的分析輸出放在同系列獨立 `analysis/`，不寫回任何 run 原始檔。 |

離線統計的最小明確口徑：五個 E0 seeds 皆具備對應 round 時才建立該 workload 的 E0 逐輪 Student-t CI；缺失者列出，不能以少於五筆的 CI 冒充預定結果。Post-failure accuracy AUC 採 `K=12` 個單位寬 accepted rounds 的離散面積，即指定視窗內十二筆 validation accuracy 的和；同時保留逐 seed 原值，必要時可用面積除以 12 表示視窗平均，不混入故障前邊界點。Recovery 從首次修復後 accepted contribution 起，找首次連續兩個 accepted rounds 的 accuracy 均落在 E0 對應輪次 CI 內；找不到則記 `not recovered`，若 E0 CI 或必要事件缺失則記無法判定，不能補值。完整但未修復的 run 仍保留訓練與事件資料；中斷 run 另列且不得默默併入五 seed 統計。時間線以實際 UTC 事件與各目標 stop 時刻對齊，缺少的階段保持未觀測，不把 subscription ready 當作首次 accepted contribution。

上述建議維持既有 source／runner／reset owner，不要求 PyMTLF component contract、額外 VM、service 或新的資料庫。若實作核對發現需改這些邊界，先停止並回到計畫決策；正式矩陣仍須在本 slice review 後另行授權執行。

## 8. 實作進度與尚待證據

目前已在既有管線接上 `config-create SEED=<值>`：生成後的有效 scenario 快照保存選中 seed、dataset ID、
原生模型 artifact identity 及 condition；原始 scenario 路徑只作來源紀錄。Host 依選中 seed 與目前 PyMTLF 模型架構
產生權重，Root 唯讀掛載對應來源；舊 scenario 與舊 run 位置不改。正式情境已補齊兩個 workload 的 E0、E1、
E2a、E2b，單次 runner／`collect-only` 共用新分層 run 路徑，並保存節點 ID 與 Leaf 類別摘要。
`fl-series-run` 只逐個呼叫既有設定生成與單次 runner，需明確選 workload、seed、condition 及 run 前綴；失敗即停。
`fl-analysis` 新增正式系列模式，由已保存原始紀錄計算五 seed Student-t CI、同 seed E0 配對差值、固定 K=12 AUC、
修復判讀與事件細節；舊兩 run 模式保留。

短跑前的 Host-only 核對沒有啟動 VM、container 或正式訓練。MNIST seed 1／2 的資料切分與模型 key 彼此不同，
同 seed E0／E1 共用 dataset ID 與模型 key；MNIST E2b、CIFAR-10 E2a 的設定與原生 checker 通過，
Root 的 Compose 唯讀掛載也通過靜態解析。離線分析以小型合成五 seed 原始紀錄驗證 CI、配對差值與 AUC，並以
Slice 2 的已保存 replacement events 核對首次修復貢獻解析；後者沒有五 seed E0 CI，因此不宣稱達成 recovery。
初次 diff review 已完成。Review 中修正了三處同範圍資料解讀：同組合一筆失敗 run 後成功重試時只選成功結果，
但仍單列失敗紀錄；subscription resource operation 只計發起端 `SENT`，並與 `NOTIFY` 分開；Branch 的
Leaf coverage 由各節點原始 JSONL 依實際事件時間對齊 Root accepted round，不假定 replacement Branch 的
`roundInd` 與 Root 相同。對應的既有分析測試已通過，尚無實機完整正式 run 驗證這些輸出。

兩個不同 seed 的短程 MNIST E0 已依第 9 節在 approved Host context 依序完成匯入、訓練、收集與 reset。
正式 E0 定義為 24 輪，這兩次短跑不放入正式系列或論文統計；初次及針對性 review 已完成，
使用者已確認 review。四十次正式實驗仍待另行授權，不將本 slice 標為 Completed。

## 9. Seed 切換實機短跑

使用者已確認先測兩個不同 seed 的實際接線，不以完整 24 輪 E0 作短測。沿用現有 MNIST `smoke.yaml` 的
無故障拓樸、2 accepted rounds、1 local epoch、資料配額與共用 runner；新增一份獨立的 seeded-smoke scenario，
只標明其短跑 identity，不修改既有 smoke 或正式 E0 的預設行為。`config-create SEED=<值>` 的模型生成、
Root 唯讀掛載及 native key 核對擴充到這個明確的短跑情境；各 seed 的 dataset ID 另分目錄，
不與正式 `mnist-formal-s<n>` 混用。短跑 run 保存在既有非正式 MNIST run 路徑，正式系列分析只讀
`e0-e2b/`，因此不會把短跑當成有效配對結果。

依序以 seed 1、2 各建立不同名稱的 generated config，使用原本的 `fl-experiment-run` 執行；每次確認
所選模型的 native artifact key 與 Root 匯入結果一致、2 輪完成、final model／held-out／原始紀錄已收集，
以及 guarded reset 完成後才切到下一 seed。第二次須核對選中的模型 key 與首次不同，且沒有沿用首次的
runtime catalog。這是接線驗證，不以 accuracy 或 recovery 作門檻；不新增第二套 runner、通用 scenario
合法值限制、額外 VM／service 或專用驗證框架。若第一 run 失敗，保留其紀錄與現場，按既有補收／安全清理
流程處理，不自動切換到第二 seed。

實際執行先使用既有 Host/provider guard 確認四台 VM 均為 `running`，再核對選中 component revision、
Host 容量、GPU、有效設定、資料與 Compose；preflight 為 0 failures，另有 free swap 為 0 MiB 的警告。
`mnist-seeded-smoke-s1` 與 `mnist-seeded-smoke-s2` 各自完成 2 個 accepted rounds，run 均為
`successful`／`finalized`，保存 final model、逐節點 JSONL、held-out 結果，並回報
`processesStopped=true`、`resetVerified=true`。兩次 generated dataset ID 不同，選中的 PyMTLF native
artifact key 也不同；第二次 Root container log 顯示匯入的 key 與 seed 2 selected config 一致。
兩次 raw runs 分別留在 `runs/protocol-hierarchical/mnist/` 下，不在 `e0-e2b/`；
這些只證明 seed 切換與共同生命週期，不證明正式 24／40 輪訓練或 E1／E2 的新五 seed 結果。
