# Slice 4 正式比較實驗準備詳細計畫

日期：2026-09-13

狀態：Planning（第二輪）；第一輪已通過 review 並提交，Slice 4 仍開放，正式實驗條件待凍結

上層計畫：[正式 Branch Replacement 比較實驗計畫](../formal-branch-replacement-comparison-plan.md)

本 Slice 收納正式實驗開始前需要的 testbed 改動，隨資料、訓練與報告條件逐輪決策、盤點和擴充。
第一輪已讓現有 experiment runner 支援無故障 baseline 與故障替換 treatment；後續 non-IID、完整 validation／test、
正式輪數及分析需求在本文件記錄下一輪方向；資料數量已盤點並提出切分候選，但條件尚未凍結。
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

資料切分、non-IID 強度與完整 validation／test 的盤點及候選見下節；正式 rounds／epochs 仍須盤點
可行性。條件凍結後更新本計畫的下一輪範圍、驗證及 real evidence，再開始該輪實作。第一版每條件
只跑一次；per-class 指標暫不做。
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
簡測保存在 `runs/protocol-hierarchical/mnist/mnist-normal-smoke-20260913/`：Root 到達 `COMPLETE`，
兩個 accepted outcomes 都為三個 Branch 正常貢獻並各有 validation；final model、held-out evaluation、
`events.jsonl`／`run.json` 及 reset verification 均成功。Host 上七個 GPU participants、十一個
PyMTLF containers 與十四個 Guest services 的 runtime evidence 已保存。簡測後四台 VM 的 provider
狀態均為 `poweroff`，Host process inventory 沒有 VirtualBox VM process。這些結果只證明短 baseline flow，
不代表正式比較或模型品質；本次未重新執行 treatment real run，該邊界留給後續正式 treatment run。

## 6. 第二輪方向與正式執行前盤點

目前選定 MNIST 與 CIFAR-10 各一組無故障 baseline／中段故障並替換 treatment，第一版共四個有效 runs。
同資料集的兩條件使用相同六份 Leaf shards、固定 seed、初始模型與訓練參數；Area A replacement 仍連回原本
兩個 Leaves。下列只盤點資料切分與 validation；尚未修改 generator 或執行正式訓練。

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

### 6.2 建議的 label-skew 與 validation 切分

建議先用同一個簡單配額候選處理兩資料集：每類從 official-train 固定抽取 200 筆 validation，
再抽取 4,800 筆分配給三個 Leaves，各取 1,600 筆且 source indices 互斥。如此每個 Leaf 只持有
五個類別、共 8,000 筆；六個 Leaves 共 48,000 筆，validation 共 2,000 筆，每輪完整評估這
2,000 筆。建議沿用目前 scenario 的 `partition.seed: 42`，固定每類先選 validation、再分配 training；
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

建議由既有 scenario 的 `partition` 記錄各 Leaf 的類別清單，沿用 `samplesPerLeaf: 8000`，
由五個類別推導每類 1,600 筆；不另設一份人工配額檔，也不把本次類別分組固定在通用 generator。
沒有類別清單的既有 balanced smoke scenario 應維持原本切分語意。本輪只記錄此表示方式，
欄位名稱與相容行為留待下一輪實作計畫確認，不預做通用配額引擎。

### 6.3 現有流程差距與凍結前核對

目前 `build_split()` 將每個 Leaf 做成十類等量，並分別輸出 validation 與 held-out artifacts；
兩份各 200 筆卻都從 official-test 抽出。`validate_output()` 又要求每個 artifact 類別等量。
完整 MNIST test 的各類樣本數不等，所以只增加 scenario 的計數既不能產生上述 non-IID shards，
也不能正確保留全部 test 樣本。下一輪
需在既有 dataset pipeline 內調整切分來源、配額表示／選取及相應語意檢查，不建立第二套人工資料來源。
候選凍結時明確指定 paired runs 共用固定來源索引、seed 與初始模型；實作後以
Leaf／Area 類別直方圖、training／validation 互斥 source indices、native loader、每輪 validation
完整覆蓋及 final test 完整 10,000 筆作直接核對。具體 scenario 表示與切分行為仍需在下一輪實作
計畫中確認；數量可行不等於目前程式已支援。

正式參數的第一版候選為 MNIST 24 accepted rounds／4 local epochs／第 12 個 accepted round 後故障，
CIFAR-10 40 accepted rounds／5 local epochs／第 20 個 accepted round 後故障。這些不是已凍結的設定；
須先確認 GPU／時間成本、fault barrier 可命中與 replacement 後仍有足夠輪次，且同資料集 baseline／
treatment 除 fault 外保持一致。第一版只有單一配對，結果只能做描述性比較。

正式執行前只對新增的 dataset／evaluation 邊界做針對性檢查：來源索引互斥、六份等量與類別分布、
native loader，以及固定 validation 的完整覆蓋；不另跑一組短輪數 GPU pilot 作流程彩排。首批正式 runs
同時核對新 validation 路徑、baseline 無故障、treatment 中段故障／priority replacement、collection
evidence 與 exact reset。失敗或不完整 run 保留並依凍結設定修正重跑，不計入有效配對；有效但效果
不明顯的 run 仍保留，若要改參數須另立實驗版本。待上述配額／validation 候選完成 user review，
seed、訓練參數及時間可行性確認後，才將下一輪實作及正式執行標為已批准。
