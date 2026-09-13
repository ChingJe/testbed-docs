# Slice 4 正式比較實驗準備詳細計畫

日期：2026-09-13

狀態：Ready for User Review（第一輪）；Slice 4 後續輪次仍開放，正式實驗條件待決策

上層計畫：[正式 Branch Replacement 比較實驗計畫](../formal-branch-replacement-comparison-plan.md)

本 Slice 收納正式實驗開始前需要的 testbed 改動，隨資料、訓練與報告條件逐輪決策、盤點和擴充。
第一輪只讓現有 experiment runner 支援無故障 baseline 與故障替換 treatment；後續 non-IID、完整 validation／test、
正式輪數及分析需求尚未批准，不在本輪順手實作。每輪都保持同一 Slice 的進度與 review 邊界，不能因名義上屬於
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
treatment 驗證。正式比較之前仍須以凍結後的設定執行 treatment pilot／正式 run。過去 100 samples／1 epoch
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

後續資料切分、non-IID 強度、完整 validation／test、正式 rounds／epochs、paired repetitions、per-class
需求與報告門檻，各自在決策後更新本計畫的下一輪範圍、驗證及 real evidence，再開始該輪實作。
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
不代表正式比較或模型品質；本次未重新執行 treatment real run，該邊界留給後續 pilot。
