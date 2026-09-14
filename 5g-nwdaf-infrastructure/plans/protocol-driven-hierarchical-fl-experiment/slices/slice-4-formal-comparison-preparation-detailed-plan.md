# Slice 4 正式比較實驗準備詳細計畫

日期：2026-09-13

狀態：資料工具與正式 scenario 已提交；MNIST／CIFAR-10 四組正式執行完成、結果待 User Review；
離線分析工具與圖表已通過 User Review，Slice 4 仍開放

上層計畫：[正式 Branch Replacement 比較實驗計畫](../formal-branch-replacement-comparison-plan.md)

本 Slice 收納正式實驗開始前需要的 testbed 改動，隨資料、訓練與報告條件逐輪決策、盤點和擴充。
第一輪已讓現有 experiment runner 支援無故障 baseline 與故障替換 treatment；後續 non-IID、完整 validation／test、
正式輪數及分析需求在本文件記錄後續方向；資料數量已盤點，資料切分及評估來源已核准進入第二輪實作，
正式 rounds／local epochs／故障時點已確認，四份 scenario 與資料／config 已建立；首組 MNIST baseline
經使用者單獨批准並執行；MNIST treatment 首次執行失敗、修正後重跑成功，直接 evidence 見上層計畫
Sections 10–11；CIFAR-10 baseline／treatment 已依序執行，結果見上層計畫 Section 12。
每輪都保持同一 Slice 的進度與 review 邊界，不能因名義上屬於
Slice 4 就繞過新 architecture 或 acceptance 決策。

## 1. 第一輪目標與現況

Operator 可用同一個 run／collect 入口，依 selected scenario 執行無故障 baseline 或既有 Branch replacement
treatment。Baseline 的 primary Branch 全程健康，Root 完成 scenario 指定的 accepted rounds；treatment 的
hard fail-stop、priority selection、自然 degraded／restored 判定與既有驗收保持原樣。兩者使用相同的 run name、
training checkpoint、Root final-model collection、held-out evaluation、`events.jsonl`／`run.json`及 exact cleanup。
`fl-training-start`／`fl-training-status`仍是以 `RUN_ID` 操作的低階入口，不承擔 run-level lifecycle。
Baseline 的三個預期 active Branch identities 由 selected `TESTBED` 的各組 priority 與實際 Root outcomes 核對，
不從不存在的 `fault.branchGroup` 推導，也不在 scenario 額外指定 participant。

改動前，`fl-branch-replacement-run`／`fl-branch-replacement-collect` 使用同一支
`scripts/host/branch-replacement-run.py`，但 `BranchReplacementContract.build()` 必須取得 `fault`，
runner 無條件等待故障 barrier 並停止 primary，`PhaseTracker`與`check_evidence()`也必須看到 stop、
failure、replacement 與相應 protocol resources。因此現有 normal scenario 雖可訓練，改動前不能直接使用這套
run-level evidence／collection 流程。這是現有路徑的擴充，不新增第二支 runner、selector 或 config source。

## 2. 既有流程的最小調整

| 現有階段 | 第一輪處置 |
| --- | --- |
| `TESTBED`／scenario 選取、render、validate、dataset、stage／activate | 原樣複用；由 selected scenario 是否具有 `fault` 決定執行條件，不新增人工 mode 欄位或新資料集 |
| VM／Guest／Host startup、GPU admission、training request／status | 原樣複用既有 owner 與 guard；第一輪仍使用 `DEVICE=gpu` 的四 VM／十一組 runtime |
| Fault trigger 與 round 判讀 | Treatment 原樣保留；baseline 不執行 fault、全部 accepted rounds 為 normal，仍要求 Root outcome 與 evaluation 一對一對齊 |
| Evidence／checkpoint／collection | 沿用同一套輸出與 retry；baseline 不產生虛構的 stop、failure、ready、degraded、restored 或 replacement resource 事件，checker 依 selected scenario 核對實際 cohort、protocol resources 與既有 final artifact |
| Stop／reset／recovery | 原樣使用 selected-and-active exact scope；collection 失敗仍保留可重試 checkpoint，不清除 Root record source |

對外提供不綁定 replacement 的 `fl-experiment-run`／`fl-experiment-collect` 作為共用入口；既有
`fl-branch-replacement-*` target 只作同一實作的薄別名，仍要求含 `fault` 的 selected scenario，
保留既有 operator 語意，不複製 runner。
實作可依目前檔案責任調整名稱，但不得為 baseline 建立平行 state machine、第二套 evidence schema 或
獨立 cleanup path。`RUN_NAME` 到 UUIDv4 `requestId`／`planId` 的既有 binding 不變。

## 3. 第一輪驗證與簡測

先在既有 owning test suite 加入最少的 baseline 行為證據：無 `fault` 的 selected scenario 能建立 run contract，
完成 scenario 指定的 accepted rounds 時不注入故障，保留每輪 Root evaluation、final-model checkpoint 與
collection-only 語意；現有 treatment 行為測試須維持通過。只測會因這次擴充而可能失效的行為，不建立新的
Slice 專用測試檔、不固化任意訓練數值，也不為已由既有 suite 保護的邊界重複加測。必要的 repository
verification 依實際修改範圍執行，不把保留但不維護的 legacy experiment suite 納入本輪驗收。

接著以現有 MNIST normal scenario（2 accepted rounds、每 Leaf 100 samples／1 local epoch）做**一次**
short GPU real-flow smoke run：使用新共用入口、目前四 VM／十一組 runtime，確認 primary 全程健康，
Root 到達 `COMPLETE`，兩個 accepted outcomes 各有 validation，final model 可收集、held-out evaluation
可完成、核心 evidence 可讀，最後完成 exact stop／reset。這只驗證 runner 的 baseline 接線與收尾，不是正式
baseline 數據；不為了增加測試數量再跑 CIFAR-10 normal，也不故意製造 collection failure。

Treatment 的直接 regression 先由既有行為測試與已保存的 Slice 3 real-run evidence 對照；本輪不要求再跑
8-round／32-epoch real replacement。後者是原實作的歷史證據，不宣稱已對本次擴充後的程式做過 real
treatment 驗證。凍結設定後由正式 treatment run 核對此邊界，不另設獨立 pilot。過去 100 samples／1 epoch
的 accepted rounds 相隔約 0.16–0.17 秒，controller 曾錯過故障 barrier；因此不能把短 epoch treatment
real run 當成本輪可靠的驗收手段。

Real VM／Vagrant／VirtualBox 操作僅在 approved sandbox-outside host context 執行；不因「簡測」放寬
provider guard、runtime inventory、wrong-config、GPU capacity 或 exact reset。若 selected runtime 尚有其他
scenario state，先依既有 guard 確認並由 operator 明確 stop／reset，不自動 destroy／rebuild VM。

## 4. 第一輪交付與後續擴充

第一輪交付是單一 runner 的雙模式 production path、保留的 treatment 行為、一次短 baseline real smoke
evidence，以及明確列出的未驗證邊界。Focused verification 後執行 mandatory initial review；只有本輪
實際影響的 owner／failure path 有 direct evidence 才宣稱這一輪 ready。實作與驗證結果保持 unstaged／
uncommitted 供 user review，commit 與 push 另行批准。Slice 4 本身保持 open，不能把第一輪通過寫成整個
正式實驗已準備完成。

資料切分、non-IID 強度與完整 validation／test 的盤點及第二輪實作邊界見下節；rounds／epochs 已完成
靜態盤點並獲確認，實際時序留待正式 run 核對。正式 scenario 通過 review、資料與 config 產物核對及
執行條件確認後才開始正式執行。第一版每條件只跑一次；per-class 指標暫不做。
若需改變 component contract、新增 config source／service／state，或改變既有 safety／acceptance，先回到
上層計畫取得決策；不因本 Slice 可逐輪擴充而自動獲准。

## 5. 第一輪實作與驗證紀錄

2026-09-13 已在 `5G_NWDAF_Infrastructure` 擴充現有 runner：共用 `fl-experiment-run`／
`fl-experiment-collect` 接受無故障或故障替換 scenario；既有 `fl-branch-replacement-*` 保持替換專用。
共用實作與 owning suite 改用 `fl-experiment-run.py`、`fl_experiment.py`、`fl-experiment.py` 等通用名稱；
只有實際描述 fault scenario 或替換專用入口的項目保留 Branch replacement 名稱，不複製第二套 runner。
Baseline 從 selected `TESTBED` priority 解析三個 active Branch，不注入 fault，仍檢查每輪 Root evaluation、
final model、held-out evaluation、protocol resource 與 exact cleanup。原有 replacement 行為不改。

既有 owning suite 的聚焦測試通過；`config-validate`、`dataset-validate` 與 sandbox 外的
`experiment-validate` 通過，後者僅警告 Host 無可用 swap。以 MNIST normal scenario 執行的一次 GPU
簡測當時保存在 `runs/protocol-hierarchical/mnist/mnist-normal-smoke-20260913/`；該舊 run 目錄
已依使用者確認清理，以下保留已 review 的執行摘要：Root 到達 `COMPLETE`，
兩個 accepted outcomes 都為三個 Branch 正常貢獻並各有 validation；final model、held-out evaluation、
`events.jsonl`／`run.json` 及 reset verification 均成功。Host 上七個 GPU participants、十一個
PyMTLF containers 與十四個 Guest services 的 runtime evidence 當時已記錄。簡測後四台 VM 的 provider
狀態均為 `poweroff`，Host process inventory 沒有 VirtualBox VM process。這些結果只證明短 baseline flow，
不代表正式比較或模型品質；本次未重新執行 treatment real run，該邊界留給後續正式 treatment run。

## 6. 第二輪方向與正式執行前盤點

目前選定 MNIST 與 CIFAR-10 各一組無故障 baseline／中段故障並替換 treatment，第一版共四個有效 runs。
同資料集的兩條件使用相同六份 Leaf shards、固定 seed、初始模型與訓練參數；Area A replacement 仍連回原本
兩個 Leaves。下列先記錄實作前的資料切分盤點；第二輪工具結果見 Section 6.5，兩資料集的
正式訓練結果見上層計畫 Sections 9–12。

### 6.1 本地官方資料的逐類數量

從 testbed 已快取的 MNIST IDX labels 與 CIFAR-10 binary batches 直接計數；來源與讀取方式對應
`scripts/host/image_dataset.py` 的 `load_mnist()`／`load_cifar10()`，不以 scenario 中的預期值代替實際數量。

| 類別 | MNIST train | MNIST test | CIFAR-10 train | CIFAR-10 test |
| ---: | ---: | ---: | ---: | ---: |
| 0 | 5,923 | 980 | 5,000 | 1,000 |
| 1 | 6,742 | 1,135 | 5,000 | 1,000 |
| 2 | 5,958 | 1,032 | 5,000 | 1,000 |
| 3 | 6,131 | 1,010 | 5,000 | 1,000 |
| 4 | 5,842 | 982 | 5,000 | 1,000 |
| 5 | 5,421 | 892 | 5,000 | 1,000 |
| 6 | 5,918 | 958 | 5,000 | 1,000 |
| 7 | 6,265 | 1,028 | 5,000 | 1,000 |
| 8 | 5,851 | 974 | 5,000 | 1,000 |
| 9 | 5,949 | 1,009 | 5,000 | 1,000 |
| 合計 | 60,000 | 10,000 | 50,000 | 10,000 |

MNIST train 的最少類別仍有 5,421 筆，故每類取 4,800 筆給 Leaves、200 筆作 validation 的
5,000 筆需求，在兩資料集的逐類容量內。CIFAR-10 會用盡 train 的每類 5,000 筆；MNIST 則有
10,000 筆 train 樣本不使用。MNIST test 各類不等量，完整官方 test 不能用現有的等類別抽樣代替。

### 6.2 第二輪的 label-skew 與 validation 切分

第二輪以同一個簡單配額處理兩資料集：每類從 official-train 固定抽取 200 筆 validation，
再抽取 4,800 筆分配給三個 Leaves，各取 1,600 筆且 source indices 互斥。如此每個 Leaf 只持有
五個類別、共 8,000 筆；六個 Leaves 共 48,000 筆，validation 共 2,000 筆，每輪完整評估這
2,000 筆。沿用目前 scenario 的 `partition.seed: 42`，固定每類先選 validation、再分配 training；
baseline／treatment 必須重現相同 source indices。具體 Leaf 配額如下；表中每個列出的類別均為
1,600 筆，未列出的類別為零。

| Area | Leaf | 持有類別 | Leaf 樣本數 |
| --- | --- | --- | ---: |
| A | `nwdaf-leaf-a1` | 0–4 | 8,000 |
| A | `nwdaf-leaf-a2` | 0–4 | 8,000 |
| B | `nwdaf-leaf-b1` | 0–4 | 8,000 |
| B | `nwdaf-leaf-b2` | 5–9 | 8,000 |
| C | `nwdaf-leaf-c1` | 5–9 | 8,000 |
| C | `nwdaf-leaf-c2` | 5–9 | 8,000 |

因此每個類別恰好出現在三個 Leaves，總計 4,800 筆；正常時 A 偏向 0–4、B 包含全部類別、
C 偏向 5–9，失去 A 後 B／C 仍覆蓋十類，但 0–4 的樣本供給相對減少。這是刻意明顯的單一
non-IID 配置，與「故障加替換」配對比較共用，不能把此一次結果推廣成所有 label-skew 情境。
官方完整 test 各 10,000 筆只在模型完成後作 final held-out evaluation，不拿來逐輪驗證或挑選配置。
模型結果先觀測整體 loss／accuracy，per-class 指標和 component 新紀錄暫不納入。

由既有 scenario 的 `partition.leafLabels` 記錄各 Leaf 的類別清單，沿用 `samplesPerLeaf: 8000`，
由五個類別推導每類 1,600 筆；不另設一份人工配額檔，也不把本次類別分組固定在通用 generator。
沒有類別清單的既有 balanced smoke scenario 維持原本切分語意，不預做通用配額引擎。

### 6.3 改動前流程差距與正式執行前核對

第二輪改動前，`build_split()` 將每個 Leaf 做成十類等量，並分別輸出 validation 與 held-out artifacts；
兩份各 200 筆卻都從 official-test 抽出。`validate_output()` 又要求每個 artifact 類別等量。
完整 MNIST test 的各類樣本數不等，所以只增加 scenario 的計數既不能產生上述 non-IID shards，
也不能正確保留全部 test 樣本。本輪在既有 dataset pipeline 內調整切分來源、配額表示／選取及
相應語意檢查，不建立第二套人工資料來源。
Paired runs 共用固定來源索引、seed 與初始模型；實作後以
Leaf／Area 各類筆數、training／validation 互斥 source indices、native loader、每輪 validation
完整覆蓋及 final test 完整 10,000 筆作直接核對。具體 scenario 表示與切分行為見 Section 6.4；
數量可行不取代實作及直接驗證。

第一版已確認 MNIST 24 accepted rounds／4 local epochs／第 12 個 accepted round 後故障，
CIFAR-10 40 accepted rounds／5 local epochs／第 20 個 accepted round 後故障。執行當日仍須核對
GPU／時間成本、fault barrier 可命中與 replacement 後仍有足夠輪次，且同資料集 baseline／
treatment 除故障及其必要觀測欄位外保持一致。第一版只有單一配對，結果只能做描述性比較。

正式執行前只對新增的 dataset／evaluation 邊界做針對性檢查：來源索引互斥、六份等量與類別分布、
native loader，以及固定 validation 的完整覆蓋；不另跑一組短輪數 GPU pilot 作流程彩排。首批正式 runs
同時核對新 validation 路徑、baseline 無故障、treatment 中段故障／priority replacement、collection
evidence 與 exact reset。失敗或不完整 run 保留並依凍結設定修正重跑，不計入有效配對；有效但效果
不明顯的 run 仍保留，若要改參數須另立實驗版本。資料工具可先依已核准配額實作；
正式 scenario 的初始模型與訓練參數通過 review、執行當日容量核對後，才將正式執行標為已批准。

### 6.4 第二輪資料工具實作邊界

沿用既有 scenario、`image_scenario_contract()`、`build_split()`、`validate_output()`、dataset
generate／check、Root validation mount 與 final held-out evaluation 入口；不建立另一套配置或 runner。
`partition.leafLabels` 是選填的 Leaf 名稱至類別清單 mapping；正式 scenario 依 Section 6.2 配置六個 Leaf，
但資料工具依 mapping 產生 shards，不在共用驗證內另列固定 Leaf 名單。沒有 mapping 的既有 balanced smoke
模式從 selected TESTBED 取得 Leaf 名稱。類別須合法且不重複，每個 Leaf 的 `samplesPerLeaf` 可平均分給所列類別。
`partition.validationSource` 選填，缺省為 `official-test`，保留既有 smoke 的 validation／held-out
抽樣語意；正式切分指定 `official-train`，
先從 train 每類選 200 筆 validation，再分配互斥的 Leaf shards。產生器以實際載入的 official-test 筆數
核對 scenario 的 `heldOutSamples`，並將整份 test 寫入 held-out；共用 scenario 驗證不另寫固定筆數。
`validation.npz`、`held-out.npz`、Leaf shard 與 manifest 路徑不變。

第二輪只調整現有 schema、generator、validator 與 owning dataset tests；以真實本地官方來源直接
產生、檢查兩個資料集的切分，確認來源索引互斥、Leaf／Area 各類筆數、native loader、固定 validation
的 2,000 筆及完整 test 的 10,000 筆。現有 smoke 行為需保持。評估入口先核對按所載樣本數完整迭代；
若路徑和 count contract 已足夠，不改 renderer、runner 或 component。此輪不建立正式四條件 scenario、
不啟動 VM／container 或長時間訓練；正式 rounds／epochs／fault 時點及其執行另行決策。

### 6.5 第二輪實作與驗證紀錄

初版在既有 `configlib.py`、`image_dataset.py` 與 `tests/image-dataset.py` 實作並測試上述切分。
初版曾要求 `leafLabels` 精確匹配固定六個 Leaf；review 修正見 Section 6.6。未指定新欄位的 smoke scenario
仍使用原本 balanced train shards 與 official-test validation／held-out 抽樣。新模式先從 official-train
留出 validation，再依 scenario 類別清單分配互斥的 Leaf shards；held-out 原樣保留完整 official-test，
不要求 MNIST test 各類等量。未新增圖表或直方圖產生工具；檢查只使用既有 manifest 的各類筆數。

聚焦 dataset 測試及 scenario／runtime inventory 測試通過。使用本地已快取的兩份官方資料，各自在
暫存目錄直接產生並由 native loader 檢查：MNIST 與 CIFAR-10 均為六份共 48,000 筆訓練資料、
2,000 筆 official-train validation、10,000 筆完整 official-test held-out；來源索引不重疊，
Leaf 類別配額符合 Section 6.2。這是資料工具證據，不是正式 scenario 的 render／stage 或訓練 evidence。
現有 Root recorder 會載入完整 validation 並以樣本數迭代評估；final evaluator 亦遍歷所載資料，
但仍須在正式 run 核對其記錄的實際 sample count。本輪沒有啟動 provider、VM、container 或 GPU。

當時另發現 PyMTLF final evaluator 內部計算 loss，但 CLI JSON 尚未輸出它；後續已決定本次
final full-test 只報 accuracy／樣本數。現有 runner 將 evaluator 輸出保存到 `events.jsonl` 與
`run.json`，因此不需為本比較修改 component contract。

### 6.6 第二輪 review 發現與修正

使用者 review 確認的發現及本輪 targeted follow-up 如下；owner 均為 Slice 4，implementation closing commit 為 `e626fa0`：

- D1（已修正，review 已確認）：初版將資料產生器原有的固定六個 Leaf 名稱移到共用設定，要求
  `leafLabels` 精確匹配，重複維護部署身分。新模式已改為依 scenario mapping 產生 shards，不檢查其
  是否屬於 selected TESTBED；沒有 mapping 的 balanced smoke 從 selected TESTBED 取得 Leaf 名稱，
  保留原切分語意。只保留防止檔案路徑或 manifest key 衝突的輸入條件。
- D2（已修正，review 已確認）：初版在共用設定和輸出檢查加入固定 `10_000` 要求，並以拒絕較小數字
  的測試固化它；這重複了產生器和實際 official-test 筆數的核對。已移除固定常數、重複檢查與該測試；
  直接複製後重複檢查完整索引的條件也已移除。產生器仍比對實際來源筆數並寫入完整 test，
  不另建完整複製的永久測試。
- D3（已修正，review 已確認）：修正 D1 後，若 `leafLabels` mapping 的 YAML 書寫順序不同，同一
  配額與 seed 仍可能分到不同 source indices，破壞配對條件。新模式以排序後的 Leaf 名稱決定分配順序；
  現有 owning test 以相同 mapping 的不同書寫順序確認相同切分。

修正後既有 balanced 與新 label-skew 資料測試、runtime inventory 測試、dataset／test 檔的 `ruff` 聚焦檢查及
`git diff --check` 通過；使用本地 MNIST、CIFAR-10 官方資料的暫存切分均為六份共 48,000 筆訓練、
2,000 筆 train-derived validation 與完整 10,000 筆 test held-out。沒有啟動 VM／container／GPU，也沒有
執行正式 scenario。正式實驗仍按 Section 6.2 使用六個指定 Leaf；這是 selected scenario 與實際資料的
核對項目，不是通用資料工具的固定名單或固定筆數限制。`configlib.py` 的 `ruff` 仍報既存的
E402／F401，對照 `HEAD` 同樣存在，未納入本輪清理。Slice 4 與正式訓練條件仍保持 open。

### 6.7 正式訓練條件的靜態盤點與建議

本節當時只讀取 `runs/protocol-hierarchical/` evidence、現行 scenario、controller 與
`testbed.protocol-hierarchical.yaml`；未改實作、未啟動 provider／VM／container／GPU。2026-09-12 UTC（本地 09-13）的
`mnist-replacement-20260913-b` 與 `cifar10-replacement-20260913-a` 均使用每 Leaf 8,000 筆、
32 local epochs、8 accepted rounds、200 筆 validation 與 250 ms controller poll，且已完成
2 normal、2 degraded、4 restored rounds。兩個 runner 從 `startedAt` 到 `finishedAt` 分別約 23 與
21 分鐘，不含預先的 VM startup／config preparation；
正常／恢復輪的 accepted outcome 間隔約 69 秒。故障後首輪約 256 秒才偵測到失效，
replacement 首次貢獻約在 stop 後 374 秒。另一次 100 samples／1 epoch normal smoke 的
兩輪 accepted outcome 只相隔約 0.14 秒，不能拿它推估正式工作量或證明 fault barrier 可命中。
上述舊流程 run 目錄已依使用者確認清理，本節保留當時盤點與其限制；本次正式比較以新 run evidence 為準。

| 靜態核對 | 結論與限制 |
| --- | --- |
| 時間 | 以舊 32-epoch 正常輪的約 69 秒粗略按 epoch 數換算，4／5 epochs 約為 9／11 秒一輪；再計入每次 runner 約數分鐘的準備／收尾與 treatment 的一次約 4 分鐘故障等待，四個 runner executions 應按約 1–2 小時量級預留，VM startup 與其他操作另計。這不是測得的正式執行時間；2,000 筆 validation、non-IID 分布、重試及當日負載均可能改變結果。 |
| Fault barrier | Controller 在 `normalAcceptedRounds` 個 accepted outcomes 後，還要求 status 的 `completedRounds` 相同，且下一輪已進入 `ROUND_DISPATCH`／`ROUND_WAITING`；250 ms poll 才能觸發 stop。舊 32-epoch runs 都命中，4／5 epochs 的較短下一輪只支持「可能命中」的估計，不能宣稱已驗證。 |
| Deadline | 選定 TESTBED 的 `roundTimeoutSeconds`、`preparationTimeoutSeconds` 均為 300；runner closure budget 會依 selected accepted rounds 計算。已確認輪數沒有顯示固定 runner budget 不足，但正式 run 仍要觀測 timeout、rejected rounds 與 replacement。 |
| 容量 | 舊 run 的 RTX 3080 為 10,240 MiB，GPU admission 時約 9,988 MiB free，高於 8,192 MiB floor，七個 GPU participants 成功啟動；正式拓樸不增加 participants。這只證明當時容量，不能代替執行當日的 GPU／Host admission。 |

靜態盤點後，使用者確認 MNIST `acceptedRounds: 24`、`localEpochs: 4`、第 12 個 accepted round 後故障，
CIFAR-10 則為 40／5／第 20 個 accepted round 後故障；兩者均保留至少一次 restored accepted round
的驗收。故障點約在訓練中段，後半仍有足夠目標輪次觀測恢復，但對新 epoch 值的 barrier
可命中性與 non-IID learning curve 不能由舊 run 證明。不另加獨立 GPU pilot，首個正式 treatment
run 直接核對實際時序；若直接證據顯示設定不可行，須先修訂配對設計並重新確認。

建議執行順序為 MNIST baseline、MNIST treatment、CIFAR-10 baseline、CIFAR-10 treatment，
每次使用不同 `RUN_NAME` 並完成既有 stop／exact reset 才切換 scenario。同資料集的一組配對
沿用相同 `partition.seed: 42`、`leafLabels`、Leaf／validation 配額、batch size、learning rate、
seed model ID／artifact key、component revisions、image 與 GPU placement；只讓 treatment 設定
`fault`。正式 scenario 各自產生資料後，直接核對兩份 manifest 的 source indices 一致；
初始模型依選定 seed source 與 reset 後狀態核對，不新增另一套 identity proof。
正式 run 名稱與建議順序可在執行前選定。

### 6.8 正式 scenario 設定

四份 scenario 建立時放在 `experiments/protocol-hierarchical/formal-comparison/`，後續依 Section 8
整理到資料集目錄，以檔名區分 baseline／replacement；沿用現有 `TESTBED`、資料產生器、renderer、checker
與 `fl-experiment-run`。同資料集配對固定
`partition.seed: 42`、`leafLabels`、8,000 筆／Leaf、2,000 筆 official-train validation、完整
official-test held-out、batch size 16、learning rate 0.001 與既有 component-native seed model。
MNIST 設為 24 accepted rounds／4 local epochs，replacement 在第 12 輪後故障；CIFAR-10 設為
40／5，第 20 輪後故障。Baseline 不設定 `fault` 或 `observation`；replacement 使用現有
`fault` 加上其 contract 要求的 `observation`，沿用 250 ms poll／30 s heartbeat。除各自的
scenario identity 與 treatment 專屬控制欄位外，同資料集的 workload、partition、training 相同。

`kind: formal-comparison` 僅描述實驗用途，不選擇拓樸或 runner。Protocol image path 的
`image_scenario_contract()` 未限制 `kind` 值；`config-render.py` 將它複製到 generated config manifest，
現有 `config-check.py` 在此 path 不比較 `kind`，也不把它寫入 image dataset manifest。
先前把舊 UE／Flat path 的 `kind` enum 誤套到此 path，並把它列為 blocker，現已更正；不需改 validator。
本節記錄 scenario 建立時的靜態核對；後續已完成四組 dataset generate／config render，並經使用者
單獨批准執行首組 MNIST baseline。其 source indices、初始模型、runtime revision 與當日容量已在
執行前核對；scenario 建立本身當時不授權其餘三組訓練。

### 6.9 Treatment runner 的故障時序 review 與修正邊界

R1（程式修正及實際 treatment 驗證完成，待 User Review）：首次正式 MNIST treatment 的失敗 evidence 見上層計畫 Section 10。
第 12 個 accepted round 後 runner 啟動 fail-stop，但在它完成前，Root 又接受一輪原 primary 的貢獻；
runner 隨後把較晚的 `BRANCH_PROCESS_STOPPED` 寫入 `events.jsonl`，再讀到較早的 Root event，
觸發 chronological guard 而中止。這是當時 supported treatment path 的 failure，不是模型品質結果；
原訂故障時點、accepted-round 目標及至少一次 restored contribution 的驗收不放寬。

修正沿用現有 runner 與 lifecycle：runtime startup 仍完成全部署、active config、Guest service、
container 及 provider guard 核對。故障 barrier 命中後，先在 approved Host context 解析選定的
Area A primary，對其 Guest 使用一次 SSH 呼叫核對 active config、service 與 PID，立即送
`SIGSTOP` 並核對同一 PID；不在此關鍵路徑重新巡查其他 Guests，也不先查 Docker container。
Go 已暫停後，再核對並暫停對應 PyMTLF、為 Guest 設 runtime mask、hard-kill 兩邊並驗證防重啟；
任何目標不符或部分失敗仍 fail closed，沿用既有 recovery／stop responsibility。這只縮短
fault-effective critical path，不變更 VM、component、scenario、訓練超參數或資料來源。

既有 `BRANCH_PROCESS_STOPPED` event 仍代表已確認兩邊 hard-stop，但其 `recordedAt` 作為 Host 收到
Guest `SIGSTOP` 成功回覆的 effective fault time；這是可觀測的確認時點，不宣稱是 Guest kernel 送出訊號的
精確時刻。payload 另保存兩邊 hard-stop 完成後的 Host 時間。
runner 在同步停機期間先緩存 Root observation，與 controller stop event 依原始時間排序後再寫入
既有 `events.jsonl`，不得倒填、放寬 chronological guard，或把多出的一輪 normal outcome 認成有效。
PhaseTracker 以 effective time 分類；若 primary 在該時間後仍成功貢獻，或第 13 輪在該時間前
已正常 accepted，仍判該 run 失敗。舊 run 的 stop event 缺少 hard-stop 完成時間時維持既有讀取語意。

只在現有 owning runner／evidence tests 覆蓋「先核對 exact Guest target 並快速 freeze、後續
hard-stop」、「停機期間 Root event 與 stop event 按時間順序處理」以及多出的 normal round 必須拒絕；
provider 相關測試只使用 mock，不在 sandbox 啟動 real Vagrant／VirtualBox。聚焦測試及
mandatory review 後，依計畫在 approved Host context 以新 run identity 重新執行正式 MNIST
treatment，核對 `SIGSTOP` 是否趕在下一輪原 primary 貢獻前，以及故障／恢復 evidence、collection 與
exact reset；下段記錄此 real-run gap 的驗證結果。

本次 targeted review 已檢查 runner／evidence／owning test diff：故障前的完整 runtime preflight 未變，
故障時只對 selected Area A Guest 做一次 SSH 核對與 freeze；Docker 查詢移到其後，兩邊 hard-stop 與
既有 cleanup 責任不變。停機期間的 Root events 與 stop event 會按來源時間寫入，PhaseTracker 仍拒絕
多出的 normal accepted round。既有 owning suite `python3 tests/fl-experiment.py` 通過；兩段修改過的
provider shell 與 Guest remote body 通過 `bash -n` 語法檢查，均未啟動 real provider。這些證據
當時尚不能證明實際 SSH 耗時與第 13 輪前能否成功暫停；後續直接 evidence 如下。

在 approved Host context 用未提交的 testbed runner 修正及新 run name
`mnist-formal-replacement-20260913-b` 重跑。先依 selected reset plan 清除上次失敗 run 的實驗狀態並通過
verify；執行當時保留舊 run 目錄，該失敗目錄其後已依使用者確認清理。重跑的 Host `SIGSTOP`
確認時間比第 12 個 accepted outcome 晚約 2.23 秒，
兩邊 hard-stop 約在其後 8.28 秒完成；第 13 個 accepted round 沒有舊 primary 貢獻，
Root 自然形成 12 normal、2 degraded、10 restored，replacement 從第 15 個 accepted round 開始貢獻。
runner `status=successful`、24 outcomes 與 25 次各 2,000 筆 Root validation 對齊、完整 10,000 筆
官方 test evaluation、final model collection 及 exact reset verify 均通過，無 failures。詳細執行結果與
可宣稱限制見上層計畫 Section 11；這次實跑關閉 R1 的時序與 end-to-end verification gap，
但 user review 前不移入 verified records，Slice 4 與正式比較仍保持 open。

### 6.10 CIFAR-10 正式配對執行

使用者批准連續執行剩餘兩組後，先對 baseline 與 treatment 各自完成 config／dataset 與 approved
Host-context preflight，再依序跑 `cifar10-formal-baseline-20260914-a`、
`cifar10-formal-replacement-20260914-a`。第一組完整結束、final model 與官方 test 結果保存、exact
reset verify 通過後才切換第二組。兩者皆完成 40 accepted rounds、41 次固定 2,000 筆 Root
validation、完整 10,000 筆官方 test、final model collection 及 exact reset；沒有 rejected round
或 runner failure。Treatment 在第 20 輪後成功停止 Area A primary，形成 20 normal、2 degraded、
18 restored；第 23 輪起 replacement 成功貢獻。配對的來源切分、訓練、component／image revision
及 GPU identity 一致。詳細結果與描述性比較見上層計畫 Section 12；四組原始資料待 User Review，
本 Slice 與正式比較仍保持 open，不移入 verified records。

## 7. 第三輪：離線 CSV 整理與繪圖（已通過 User Review）

正式配對與新增 CIFAR-10 全類別不等量配對均已保存 `run.json`、`events.jsonl` 與 final model；
上層計畫 Section 5 已要求在訓練完成後離線對齊逐輪曲線。本輪只擴充 testbed 的 Host-side
分析工具，不改 runner、evidence schema、component、scenario、VM 或訓練結果。現有
`run.json.phases.rounds` 已記錄 accepted round 的階段與成功 Branch identities，
`events.jsonl` 的 Root `MODEL_EVALUATION` 已記錄 initial／每個 accepted round 的 validation
accuracy、loss 與樣本數；`run.json.heldOutEvaluation` 與 `phases.latencies` 已保存 final test
及故障恢復摘要。直接使用這些已保存欄位，不另從一般 log 重建事件或重新判定 phase。

Operator 透過離線腳本或薄層 `make fl-analysis BASELINE_RUN=... TREATMENT_RUN=... OUTPUT_DIR=...`
明確提供 baseline run 目錄、treatment run 目錄及另一個輸出目錄；此 Make 入口不要求 `TESTBED` 或
`CONFIG_DIR`，也不呼叫 provider、runner 或訓練流程。工具不掃描 run catalog
自動挑選配對，也不寫回原始 run 目錄。第一版只處理已成功完成並保存必要 evidence 的一對 runs；
缺少可對齊的 accepted round 或 Root validation 時明確報錯，不補值、不插值。圖與 CSV 使用
1-based accepted-round 序號，初始 validation 單列為 round 0；百分比差值定義為 treatment
減 baseline 的百分點。原始 `roundInd` 為 0-based，不直接當成圖上的 accepted-round 序號。

每組配對輸出下列最小產物，名稱不包含特定資料集或 work-item identity：

- `rounds.csv`：initial 與逐輪兩組 validation accuracy／loss、accuracy 差值、treatment phase，
  以及各輪成功 Branch identities；不把 final test 當成下一個 training round。
- `summary.csv`：每組配對一列，包含兩個 run 名稱、資料集、兩組 final 官方 test
  accuracy／正確數／樣本數，以及 treatment 的 normal／degraded／restored 輪數和
  fault-to-detection／ready／first-contribution 時間。
- `comparison.svg`：同一張圖的 accuracy、loss 兩個面板，疊合 baseline／treatment，標示
  primary Branch 停止與 replacement 首次貢獻的 round 邊界，兩條事件線之間是 degraded 區段，
  不使用底色區塊；圖上使用 validation 指標，final test 僅列於摘要，不畫成逐輪曲線。

工具是資料整理與展示入口，不取代既有 run evidence checker 或正式報告的配對條件審查。
驗證只用現有保存的 MNIST 正式配對、CIFAR-10 原正式配對與新增全類別不等量配對，
核對各自輸出的輪數、已知故障／恢復區間、CSV 數值與 final test 摘要；用一個聚焦的
資料對齊測試保護欄位映射與 0-based／1-based 序號，不建立實驗名稱或固定輪數的永久測試。
本輪不產生 per-class 圖、直方圖、統計推論或跨資料集排名，也不執行 provider、VM、container
或 GPU 操作。實作與產圖已獲使用者確認；本輪完成 review 前保持 open state。

目前以獨立 Host-side 腳本及薄層 `make fl-analysis` 入口產生上述三檔，未修改 runner、component
或既有 run evidence。聚焦資料對齊測試通過；三組已保存的 MNIST 正式、CIFAR-10 原正式及
CIFAR-10 全類別不等量配對均可從同一入口離線產出，分別得到 24、40、40 個 accepted rounds，
treatment 階段數與原 `run.json` 相符，final test 摘要亦與保存資料相同。這僅是分析工具的
直接驗證，不重審三組訓練的實驗條件或宣稱整體比較已完成；輸出仍待 User Review。

使用者檢視初版圖後決定改由 Matplotlib 繪圖。Host 目前使用 Python 3.8，分析工具在 testbed
專案的選用依賴中固定相容的 Matplotlib 3.7.5；薄層 Make 入口透過專案本地 `uv` 環境執行。
CSV 欄位、兩條 validation 曲線、故障階段標記及 `comparison.svg` 檔名維持不變，僅替換繪圖實作與
樣式，不影響 runner 或原始 run evidence。已確認 `runs/protocol-hierarchical/analysis/` 的三張
舊圖為 exact targets，並以新圖覆蓋；各組 `rounds.csv`、`summary.csv` 均保持原檔且與重算結果
逐檔相同。三組新圖可正常解析並完成視覺檢查；本輪仍保持 User Review open state。

後續依使用者檢視意見移除 degraded 區塊底色；曲線圖例改為 `No-failure baseline` 與
`Branch failure + replacement`，事件線標為 `Primary branch stopped` 與
`Replacement first contributes`，避免 `Healthy`、`Replacement`、`Fault` 等簡寫造成條件或事件歧義。
CSV 和原始 run evidence 不變，三組圖在同一輸出路徑重新產生後仍待 User Review。

再依使用者要求調整成適合論文正文的圖面：採約雙欄寬度、兩個有 (a)／(b) 標記的面板、較小的
字級與圖例，不在圖內放大標題或背景格線；兩組曲線以顏色、實線／虛線和不同資料點符號區分，
灰階檢視時仍能辨認。事件線語意、原始數值與輸出檔名不變；資料集與完整實驗條件由後續圖說
交代，不將展示樣式當成實驗結果。更新後三組圖已通過 User Review。

## 8. Scenario 路徑整理（已通過 User Review）

使用者確認將 `experiments/protocol-hierarchical/` 下的 scenario 統一按資料集分類，並以檔名表示
條件。`mnist/` 保存 `smoke.yaml`、`replacement-smoke.yaml`、`formal-baseline.yaml`、
`formal-replacement.yaml`；`cifar10/` 除同名四份外，另保存 `all-class-skew-baseline.yaml`、
`all-class-skew-replacement.yaml`、`proximal-mu-0.1.yaml`。這次只搬移原有十一份定義，不改其
`name`、`kind`、資料切分、訓練參數或故障條件；現行 README、Make 提示與 repository tests 改用新路徑。

既有 `runs/` 內的 `run.json` 是執行當時的 evidence，不回寫舊 `scenario.definition`。本機八份
`config/local/` manifest 原先指向舊路徑；不直接手改 generated artifact。現有 reset 會先做 config
check，因此不能搬移後才依賴缺少舊 source 的 config 執行 reset。使用者已確認接續處理這八份
本機設定，不重建 VM、不執行訓練，也不改資料集或歷史 run evidence。

遷移前先讓目前 selected 的舊定義暫時可解析，透過既有 guard 核對 selected／active identity、
process 狀態及 exact reset verify；若狀態不一致或仍有實驗程序執行，停止，不覆蓋任何設定。
再用原 renderer 依每份設定原有 name、GPU policy 與 WebConsole 選項產生候選 config，和原本
generated tree 比對；預期只變動 manifest 的 scenario definition，若有其他差異則先回報，不直接
替換。通過後才更新 Host 本機設定，並透過既有 stage／activate 與 rollback boundary，把目前
selected config 在四台現有 VM 上切換成新 identity，核對全部 Guest 與 Host 一致；服務維持停止。
不自動執行會清除實驗資料的 reset。最後移除暫時恢復的舊 source，只保留資料集優先的十一份
scenario。既有資料／config checks 是靜態及 Host 證據；Guest activation 另需 approved Host context
的直接證據。不因先前訓練完成而自動關閉本次遷移的 review。

2026-09-14 在 approved Host context 核對四台 VM 均 running、十一個 ML containers 均 exited，
Guest 實驗服務無 active；Area A primary 保留前次故障後的 `failed` 狀態但沒有執行中程序。
目前 selected 的 `cifar10-all-class-skew-replacement` 舊 Host／Guest identity 均為
`2a28a8f141e6…`，暫時恢復其原 scenario 路徑後，舊 config check 與 exact reset verify 通過；
NRF／ADRF 實驗資料、ADRF models 與十一個 ML volumes 均為空，未執行 reset apply。

原 renderer 在暫存目錄產生八份候選 config；逐檔比較證實每份僅有 manifest 的
`scenario.definition` 路徑變動，八份候選與安裝後的 config checks 均通過。切換 selected
Host config 後，透過既有 stage／activate 與 rollback boundary 同步四台 VM，新的 Host／Guest
identity 為 `8dfdf6f6a86c…`；後續 Guest service status、八份 dataset checks 與新 identity 下的
exact reset verify 通過，服務仍未啟動，四台 VM 未重建。其餘七份本機 generated config 已換成
相同方式產生的新路徑版本；十一份 scenario 原文逐檔與搬移前相同。既有 run evidence、資料集與
實驗數值未改，本次只取得設定遷移的直接證據，不宣稱新的訓練 run 已通過。
完成核對後已清除本次在 `/tmp` 建立的候選與舊 generated config 暫存備份；原始 run 資料未刪除。
