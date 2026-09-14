# Protocol-driven Hierarchical FL 正式 Branch Replacement 比較實驗計畫

日期：2026-09-13

狀態：原四組正式實驗與新增 CIFAR-10 全類別不等量配對均已執行，結果待 User Review；正式比較分析與報告仍為 Draft

前置工作：[Multi-host Branch Replacement Experiment Plan](./multi-host-branch-replacement-experiment-plan.md)
已完成雙資料集、四 VM、GPU、八個 accepted rounds 的流程驗證；其結果只證明流程可運作，
不是本計畫的 paired baseline 或模型品質比較。本計畫是後續獨立工作，不改寫該計畫的驗收與紀錄。

正式實驗前的 testbed 改動歸入 [Slice 4 詳細計畫](./slices/slice-4-formal-comparison-preparation-detailed-plan.md)，
依已決定的實驗條件逐輪擴充。共用 runner 的雙模式支援與短 epoch 簡測已完成；第二輪只實作已確認的
資料切分與評估來源。正式 rounds／local epochs／故障時點已確認，四份 scenario 已依此建立並產生資料與
config；使用者在完成產物及當日容量核對後，批准先執行 MNIST baseline。其餘條件不因此自動獲准執行。

## 1. 研究問題與可宣稱範圍

在相同的 Root–Branch–Leaf 拓樸、資料切分、初始模型與訓練設定下，比較：

- Area A primary Branch 全程正常時的學習曲線；
- Area A primary Branch 在訓練中段 fail-stop、Root 暫時只聚合 Area B／C，之後透過既有 priority 機制讓
  replacement Branch 接手時的學習曲線與恢復過程。

本比較能描述「正常運作」相對於「故障加替換」的整體影響，不能單獨證明 replacement 機制相對於
「故障但不替換」的因果效益；若要回答後者，須另設第三種故障條件。MNIST 與 CIFAR-10 各自比較，
不把兩個資料集的絕對 accuracy 直接視為同尺度優劣。

## 2. 四種實驗條件

| 資料集 | 無故障 baseline | 故障與替換 treatment |
| --- | --- | --- |
| MNIST | Area A primary Branch 全程健康 | 中段停止 Area A primary；Root 自然選擇次順位 replacement |
| CIFAR-10 | Area A primary Branch 全程健康 | 中段停止 Area A primary；Root 自然選擇次順位 replacement |

四種條件使用相同的四 VM 與十一組 NWDAF↔PyMTLF 配置；Area A replacement candidate 在 baseline 中
同樣存在、註冊，但不取代健康的 primary。Baseline 不是「故障但不替換」，也不是舊的兩輪 normal smoke run。
Treatment 沿用已驗證的 hard fail-stop、Root priority selection、自然 preparation 與後續 round rejoin；
controller 不指定 replacement、不加入人為 preparation delay 或 component 內部捷徑。

每個資料集的一組 baseline／treatment 必須使用相同資料切分、初始模型、訓練超參數、accepted-round
目標與 runtime placement。不同條件各自使用獨立 run identity 和乾淨的 selected runtime state；每次執行記錄
實際 revision、dirty flag、config、dataset seed、裝置及容量資訊。第一版每個條件各跑一次，共四個
有效 runs、每資料集一組配對；報告僅作描述性比較，不宣稱統計顯著。若日後要做統計推論，另行決定並
執行配對重複實驗，不把這四次結果冒充足夠的重複樣本。

## 3. 資料與 non-IID 設計

兩個資料集都採固定、可重建的六個 Leaf training shards。這版採用 label-skew non-IID：每個 Leaf
仍取得相同數量的獨立樣本，但只持有部分類別；Area A 與 Area B／C 的類別分布應有可觀測差異，同時
保留必要的類別覆蓋與可訓練性。baseline 與 treatment 必須使用完全相同的六份切分、seed 與初始模型；
replacement Branch 重新接回 Area A 原本的兩個 Leaves，不換成另一份資料。本地官方資料快取的逐類
盤點與已核准的資料配額見 Slice 4 詳細計畫：每類 4,800 筆進入 Leaves、200 筆留給 validation；
每個 Leaf 僅取五個類別、每類 1,600 筆。此配置中 Area A 偏向類別 0–4，Area C 偏向 5–9，
B／C 在 Area A 故障時仍覆蓋全部十類。正式執行前須以實際 source indices、Leaf／Area 各類筆數與
native loader 核對切分；此配置已核准供資料工具實作，不可看完結果才反向調整。
Non-IID 是兩種條件共同的實驗背景，不增加 IID／non-IID 兩套條件。

第二輪工具調整在既有 scenario 的 `partition.leafLabels` 表達各 Leaf 持有的類別，沿用
`samplesPerLeaf: 8000`，由每 Leaf 的五個類別推導每類 1,600 筆，不把本次 0–4／5–9 分組寫死在
通用 generator，也不另建人工切分來源。沒有該配置的既有 balanced smoke scenario 保持原有語意；
第二輪的具體欄位與相容方式見 Slice 4 詳細計畫。

目前流程每個 Leaf 使用 8,000 筆、共 48,000 筆訓練樣本；MNIST 官方 train／test 為
60,000／10,000，CIFAR-10 為 50,000／10,000。CIFAR-10 在維持六個互斥 Leaf shards 的前提下，
已沒有太多空間同時增加 Leaf 獨立樣本與保留 train-derived validation。第一版沿用每 Leaf
8,000 筆，兩資料集均為六個等量 shards，不為了用滿來源資料而提高數量。盤點證實上述五類／Leaf
配額在數量上可行；第二輪先用暫存切分直接驗證 generator，後續已為四份正式 scenario 產生持久資料。
不能把增加 local epochs、重複抽樣或資料增強描述為增加了獨立樣本。

Per-round validation 應完整遍歷一份固定、與 Leaf training shards 互斥的 validation set，而非沿用
200 筆 smoke set，也不是每輪遍歷官方 test split。CIFAR-10 在六個 Leaves 共使用 48,000 筆
official-train 樣本時，剩餘 2,000 筆作為固定 validation。MNIST 採同一配額：48,000 筆
training、2,000 筆 train-derived validation，其餘 10,000 筆 official-train 不使用。兩資料集的
validation 均為每類 200 筆；此配額已寫入正式 scenario 並產生資料。最終模型另用
**完整的官方 10,000 筆 test split**作一次主要 held-out 評估；本次訓練參數決策未使用 test 結果，
後續也不得依這份 test 結果回頭調整同一版配置。
切分沿用目前 scenario 的 `partition.seed: 42`：每類先固定選出 200 筆 validation，再把互斥的
4,800 筆分配給三個 Leaves；同資料集的 baseline／treatment 須得到相同的 source indices。
第二輪改動前的 generator 雖分別輸出 validation 與 held-out artifacts，卻把兩者都取自官方 test split、各只用
200 筆，且要求所有輸出每類別等量；正式切分必須調整來源與檢查規則，不能只把 scenario 的
計數改成 10,000。MNIST 官方完整 test split 各類樣本數並不相等，final evaluation 不得為了等量
而丟棄 test 樣本。

## 4. 時間軸與訓練工作量

先前八輪流程只在第二個 accepted round 後故障，不足以觀察中段替換的前後曲線。第一版正式設定中，
下列 rounds／local epochs／故障時點已由使用者確認；靜態盤點見 Slice 4 Section 6.7。同資料集的 baseline／treatment 必須一致，只有
treatment 注入故障。先前流程驗證的每 Leaf 32 local epochs 不沿用為正式實驗值。

| 資料集 | accepted rounds | 每 Leaf local epochs | treatment 故障時點 |
| --- | ---: | ---: | --- |
| MNIST | 24 | 4 | 第 12 個 accepted round 完成後 |
| CIFAR-10 | 40 | 5 | 第 20 個 accepted round 完成後 |

既有 GPU 紀錄只支持上述已確認設定的執行時間量級，不能證明 4／5 epochs 的實際
fault barrier 時序、新 validation 成本或執行當日容量。正式 treatment
run 直接核對 barrier 與 replacement；不另加 GPU pilot。不把較弱或較強的 learning-curve 效果
當成修改同一次有效 run 條件的理由。

故障後的 degraded accepted rounds 數量由 production timeout／recovery timing 決定，不預先強制固定；
accepted、rejected、failure-detection、replacement-ready 與首次 replacement contribution 都分別記錄。
Treatment 至少要觀測一次 replacement 成功參與後續 accepted round；若目標輪數結束仍未恢復，
保留不完整結果，不能事後延長單一 run 來冒充原訂方案。若後續直接證據顯示已確認設定不可行，
須先修訂同資料集的配對設定並重新取得確認；若正式結果的效果不明顯，有效 run 仍須保留並報告，
後續新設定另立版本。

## 5. 觀測、分析與資料保存

沿用目前 node-local structured JSONL、`events.jsonl`、`run.json`、Root 持久 final model 與可獨立重試的
collection。每個 accepted round 的 Root validation loss／accuracy 與 `ROOT_ROUND_OUTCOME` 一對一對齊；
比較圖以相同 accepted-round index 疊合 baseline／treatment，並在 treatment 標註故障、degraded 與
restored 區間。主要另報完整官方 test 的 final accuracy／樣本數、故障至偵測／ready／首次貢獻的時間、
各階段 accepted rounds 數量及每輪成功 Branch identities。Final test 不要求 loss；目前 evaluator 輸出的
accuracy、正確筆數與樣本數會保存到 `events.jsonl`／`run.json`，不需為本比較修改 PyMTLF。

Non-IID 配置至少保存各 Area／Leaf 的類別分布，但本版模型表現只報整體 loss／accuracy，不要求
per-class 評估或曲線。現有 `MODEL_EVALUATION` 提供逐輪整體 loss／accuracy，final evaluator CLI
目前只提供整體 accuracy。若整體指標不足以回答後續問題，再另行決定是否新增 component 記錄或離線評估，不能從整體 accuracy
推論出類別效果。

正常監控只使用增量 structured events 與精簡 milestone；一般 journald／Docker logs 僅在精確故障時間窗
按需存入 `diagnostics/`，不持續串流。Training 與 collection 保持獨立 checkpoint；後處理失敗可用同一
run identity 重試，不因作圖或評估 bug 重新訓練。正式分析輸出可在全部 runs 完成後由保存資料離線產生，
不綁在訓練 runner 的成功路徑上。

下一輪先製作獨立的離線 CSV 整理與繪圖工具，細節見 Slice 4 Section 7。Operator 明確指定一組
baseline／treatment run 目錄與輸出目錄；工具讀取原有 `run.json`／`events.jsonl`，以 accepted-round
順序對齊 Root validation，輸出逐輪比較 CSV、最終結果與恢復時間摘要 CSV，以及同圖呈現兩組
accuracy／loss 並標記 treatment 故障、degraded、restored 區間的曲線圖。原始 run evidence 保持唯讀；
不為了作圖重新訓練、蒐集 log、計算 per-class 指標，或把原本五類／Leaf 與新增全類別不等量
CIFAR-10 配對自動混成同一條曲線。正式報告仍須先人工確認配對條件與可宣稱範圍，圖表本身
不是統計顯著性或 replacement 單獨效益的證明。

Baseline 與 treatment 使用同一 run-level evidence／checkpoint／collection contract；baseline 不產生虛構的
fault、degraded 或 replacement events，treatment 才要求相應的故障與恢復證據。兩者都須保留完整的
accepted-round 與 validation 對齊資料，才能進行配對比較。

## 6. 實作邊界與驗證順序

此版仍是正式實驗主計畫草案；Slice 4 第一輪 runner 與第二輪資料工具變更已由詳細計畫限定，
正式 scenario 及產物已建立，兩資料集的四組正式執行已完成；比較分析與報告仍待 review，
已確認的三項訓練數值不因此自動變更。
預期 owner 是 `5G_NWDAF_Infrastructure/` 的 scenario validation、dataset preparation、現有 renderer／runner、
evidence checker 與 tests；`testbed-docs/` 保存本計畫、後續實驗定義和 reviewed records。
NWDAF／PyMTLF／NRF／ADRF 原則上沿用現有 revisions，不預設修改 component contract。

Slice 3 後續 review 已將共用 validator 中屬於當次驗收的固定輪數、樣本數、local epochs 與觀測頻率
改為由 selected scenario 決定，追補已完成 review。目前
`fl-training-start` 是以 `RUN_ID` 送出 request／查詢 status 的低階控制入口，不負責實驗 lifecycle 或
evidence；共用 `fl-experiment-run` 已使用相同的底層 `fl-control` 功能，並另外負責 runtime、fault、
observation、collection 與 cleanup。它已能依 scenario 執行無故障 baseline 或故障替換 treatment，
不把 `fl-training-start` 改成另一個 run-level workflow。對外 run／collect 命令不綁定 replacement；
既有 target 的過渡方式見 Slice 4 詳細計畫，維持一套 runner implementation 與 checkpoint 語意。

正式 baseline 與更長 treatment 必須由同一現有 `TESTBED` + scenario +
`CONFIG_DIR` pipeline 支援，不能只增一組 YAML 或複製第二個 runner。下一輪規劃細化時要逐段標明現有
dataset → render／validate → stage／activate → start → trigger → observation → collection → stop／reset
哪些原樣複用、哪些因正式比較而調整；provider、wrong-config、容量與 exact reset 防護維持不變。

後續驗證與執行順序：

1. Slice 4 第一輪已完成共用 runner 雙模式擴充及 short normal scenario 的短 epoch GPU 簡測；
   treatment 保留原有流程與直接回歸證據。這只是 runner 接線驗證，不作正式比較，也不藉此凍結資料與輪數。
2. 已盤點來源類別數量並核准六份等量 label-skew shards、固定 validation 配額及 seed，
   在同一 Slice 4 第二輪完成必要的 dataset／evaluation 調整；rounds／local epochs／故障時點已於後續確認。
   執行前以來源索引、
   Leaf／Area 各類筆數、互斥性、native loader 與 validation 全量覆蓋做針對性檢查，不另跑獨立
   流程彩排 pilot，也不預先產生另一份 Slice 文件。
3. 完成正式 scenario review、資料／config 產物核對並確認當日容量後，各條件先跑一次；第一批正式 runs
   同時核對新的 validation 路徑、baseline 無故障、treatment 中段故障與 priority replacement、collection／evidence 完整性及
   exact reset。任何失敗或不完整 run 保留紀錄，修正後依原凍結設定重跑，不計入有效配對；有效但
   效果不明顯的 run 不因研究結果而剔除。
4. 對等檢查各配對的初始條件、accepted-round 對齊、缺失資料、evaluation 覆蓋與故障時序，
   然後才產生圖表和 report claim。任何未完成或不符合預定條件的 run 另列，不補值或靜默剔除。

## 7. 驗收與 verification matrix

正式比較整體尚未完成；下表是各條件的驗收邊界。MNIST 的直接 evidence 見 Sections 9、11，
CIFAR-10 見 Section 12；四組結果仍待 User Review。Slice 4 第一輪 runner 擴充已有獨立的 review 與簡測紀錄。

| 驗收項目 | 靜態／受控驗證 | 正式執行 evidence |
| --- | --- | --- |
| 四條件可由同一 pipeline 選取 | scenario／render／同一 runner tests 覆蓋 baseline 與 treatment；低階 `fl-training-start` 語意不變，無第二套 selector／runner | 同一四 VM／十一 process topology，baseline primary 全程健康且無虛構故障事件，treatment 由 Root 依 priority 替換 |
| 資料互斥且配對一致 | 來源索引、Leaf／Area 各類筆數、seed、完整 validation／test count 與 native loader checks | 配對 runs 保存一致的 split 與初始模型資訊；每輪走完整固定 validation，完整官方 test 僅作凍結後的 final evaluation |
| 中段故障與 learning curve 可重建 | fault barrier、accepted／rejected、phase classifier 與 metric alignment tests | Treatment 有故障、degraded 與至少一次 restored contribution；兩條曲線按 accepted round 對齊 |
| Evidence 與結果可審查 | collection retry、final artifact、run completeness 與缺失資料 checks | `events.jsonl`、`run.json`、final model、完整 test 結果及清楚標示的未完成 run |
| Runtime safety 保持 | 現有 provider guard、wrong-config、容量、inventory、exact reset tests | Approved host context 的 actual inventory、GPU、stop／reset 與 selected state 核對 |

正式比較的最低可交付物是每資料集至少一組有效 baseline／treatment 配對、對應逐輪曲線、
完整官方 test 的 final 結果，以及 treatment 的故障與恢復 timeline；這只允許描述性結論。
後續重複次數、統計方法與任何模型品質門檻尚未核准，不能從第一版單一配對推導顯著性或改善保證。
未通過矩陣或缺少 real-environment evidence 時保持 open state，不將 run 收入 verified record。

## 8. 執行條件與非目標

四組正式執行已完成；下列條件及證據仍待 User Review：

- 已核准的五類／Leaf 配額與兩資料集各 2,000 筆 train-derived validation 已用持久產物核對；
  四組正式 run 的 evaluation 覆蓋已核對，配對來源索引與分布依各資料集的相同 split manifest 核對；
- MNIST 24 rounds／4 local epochs／第 12 輪後故障，以及 CIFAR-10 40／5／第 20 輪後故障已確認並執行；
  四份正式 scenario 沿用現有 batch size、learning rate 與各資料集 seed model，兩資料集的配對
  partition、training、workload、component revisions、image 與 GPU identity 已核對；
- 四次執行的 Host admission、fault barrier、逐輪 validation、final evaluation 與 exact reset evidence
  均需在 User Review 中確認，不能只憑 runner 成功狀態宣告整體比較完成。

本輪不比較 Flat FL、FedAvg／FedProx、Branch 聚合頻率、GPU 效能成本或「故障但不替換」；
不改 5g-viz、5GC user plane、component recovery semantics，也不把流程驗證紀錄當成正式比較資料。
正式比較仍保持 Draft。使用者已批准並完成四組正式執行；Slice 4 第一輪 runner 擴充及短 epoch
簡測、第二輪資料工具實作與參數確認均不取代本次結果的 User Review 與後續分析。

## 9. 首組執行結果（待 User Review）

2026-09-13 使用 `feat/hierarchical-fl-protocol-extension` 的 `d2e8ba3`，選定
`config/local/formal-mnist-baseline` 與 `RUN_NAME=mnist-formal-baseline-20260913-a` 執行第一組。
四份正式 config／dataset 均通過既有 validator；MNIST 配對的 source indices、類別分布、seed model 與訓練設定一致。
執行當日 Host preflight 通過，GPU 啟動前有 9,988 MiB 空閒，高於 8,192 MiB floor；可用 swap 為 0 MiB，
屬監測警告。四台既有 VM 由 approved Host context 啟動，沒有重建或重新 provision。

`runs/protocol-hierarchical/mnist/mnist-formal-baseline-20260913-a/` 的 `run.json`／`events.jsonl` 記錄
Root `COMPLETE`，24 個 accepted rounds 均為 normal、Area A primary 與 B／C Branch 成功貢獻，沒有 degraded、
restored 或失敗輪。24 個逐輪 Root validation 與 outcomes 一對一對齊，每次及初始評估均使用固定的
2,000 筆 validation；初始 accuracy 10.0%，最後一輪 accuracy 86.9%、loss 0.4124。最終模型已保存，
完整官方 test 的 held-out accuracy 為 8,884／10,000（88.84%）。runner 回報無 failures，
Guest／容器服務已停止且 exact reset 驗證通過；四台 VM 保持 running。

這是 baseline run 的直接結果；後續 treatment 重跑見 Section 11。單一配對只能做描述性比較。
本結果在 user review 前不移入 verified records。

## 10. MNIST treatment 首次執行失敗（原始 run 已清理）

2026-09-13 使用相同 testbed revision、配對資料與訓練設定，選定
`config/local/formal-mnist-replacement`，以 `RUN_NAME=mnist-formal-replacement-20260913-a` 執行。
執行前確認四台 VM 的 Host process 與 provider 狀態一致、前次 Guest／容器服務已停止，GPU 有
9,988 MiB 空閒；baseline／treatment scenario 只有名稱、故障及其觀測設定不同。

`runs/protocol-hierarchical/mnist/mnist-formal-replacement-20260913-a/` 當時保存失敗的 `run.json`、
`events.jsonl` 與 `diagnostics/`；該目錄已依使用者確認清理，以下保留當時的診斷摘要。runner 記錄前 12 個 accepted rounds 均為 normal，並在預定 barrier
開始停止 Area A primary；Guest 與 Host PyMTLF 均被 hard-stop。Root 診斷 log 顯示，故障操作完成前，
`roundInd=12` 仍由原 primary、B、C 成功貢獻並成為第 13 個 accepted round。runner 於
14:31:50 UTC 寫入 `BRANCH_PROCESS_STOPPED` 後，再讀取時間更早的 Root 事件時被
`events.jsonl would not be chronological` 防護中止。因此目前不能以此 run 判定 degraded／restored
行為或模型最終表現；它不計入有效配對，也不以 `collect-only` 補成完整訓練。

失敗路徑已停止所有實驗容器與 Guest process，四台 VM 仍 running；primary Guest service 為
`failed`（無執行中 process），其餘 Guest services 為 `inactive`。runner 未完成成功路徑的 exact reset，
下一次啟動前需按 selected runtime 執行 guarded reset 與核對。使用者確認依 Slice 4 Section 6.9
修正現有 runner 的 critical path 與事件順序，維持原訂 24 rounds／4 local epochs／第 12 輪後故障；
修正通過聚焦測試與 targeted review 後，才執行 Section 11 的重跑。此失敗 run 仍不能作為有效 treatment。

## 11. MNIST treatment 重跑結果（待 User Review）

使用者確認 runner 修正後，以原訂 24 accepted rounds／4 local epochs／第 12 輪後故障設定，
使用 `config/local/formal-mnist-replacement` 與新的 `RUN_NAME=mnist-formal-replacement-20260913-b`
重跑。執行前 selected config、dataset、Host/provider preflight 通過；GPU 空閒 9,988 MiB，
仍有 0 MiB free swap 警告。先依 selected reset plan 清除上次失敗 run 留下的 NRF／ADRF
實驗狀態及十一個 ML volumes，並通過 verify；執行當時未刪除既有 run 目錄、容器與映像，
失敗 run 目錄其後已依使用者確認清理。
本次 testbed revision 為 `d2e8ba3`、working tree 含尚未提交的 runner 修正；PyMTLF、NWDAF、
NRF、ADRF 使用與 baseline 相同的 component revisions，配對的 scenario partition、training 與 seed
model 設定相同。

`runs/protocol-hierarchical/mnist/mnist-formal-replacement-20260913-b/` 的 `run.json` 記錄
`status=successful`、`finalized=true`、無 failures，Root 完成 24 個 accepted rounds：12 normal、
2 degraded、10 restored。第 12 個 outcome 在 15:36:10.179892 UTC；Host 於
15:36:12.405374 UTC 收到 Guest `SIGSTOP` 成功回覆，15:36:20.689112 UTC 完成兩邊 hard-stop。
第 13 個 accepted round 只由 Area B／C 成功貢獻，沒有上次多出的舊 primary normal round；
replacement 從第 15 個 accepted round 開始貢獻。以 effective fault time 計，Root 在約
297.9 秒後記錄 failure-detected、298.7 秒後 replacement-ready、313.3 秒後首次接受 replacement
貢獻。這些是本次執行的觀測時間，不是固定的 recovery 保證。

`events.jsonl` 有 24 個 Root outcomes、25 個對應的 Root validation（含初始一次），每次均評估
2,000 筆，事件時間順序正確；最終模型已保存，完整官方 test 為 8,899／10,000（88.99%）。
runner 完成 selected process stop、Guest restart policy restoration 與 exact reset verify，四台 VM
未被 destroy。相對於 Section 9 baseline 的 88.84%，final test accuracy 高 0.15 個百分點；
單一配對只能描述這次結果，不能據此宣稱 replacement 提升模型品質或有統計顯著性。
這次重跑提供 Slice 4 R1 的 real-environment evidence，但結果在 user review 前不移入 verified records；
其後已執行的 CIFAR-10 配對見 Section 12，正式比較仍保持 Draft。

## 12. CIFAR-10 配對執行結果（待 User Review）

2026-09-14 使用相同的 `feat/hierarchical-fl-protocol-extension` revision `d2e8ba3`，依序執行
`cifar10-formal-baseline-20260914-a` 與 `cifar10-formal-replacement-20260914-a`。兩組各自的
config／dataset 與 approved Host-context preflight 通過；GPU free 9,988 MiB 高於 8,192 MiB floor，
仍有 free swap 0 MiB 的監測警告。兩次使用相同的 partition、training、workload、component revisions、
PyMTLF image 與 GPU identity；split manifest、validation 與 held-out artifacts 相同。baseline 成功完成
並通過 exact reset verify 後，才切換到 replacement；兩次都沿用未提交的 runner 修正，沒有改動
40 accepted rounds／5 local epochs／第 20 輪後故障的凍結設定。
原始資料分別保存在 `runs/protocol-hierarchical/cifar10/` 下的同名 run 目錄。

兩個 run 的 `run.json` 均為 `status=successful`、`finalized=true`、無 failures，各有 40 個 accepted
outcomes、41 次 Root validation（含初始一次），每次完整評估固定的 2,000 筆，沒有 rejected round；
final model、完整官方 10,000 筆 test evaluation 與 process stop／restart-policy restoration／exact reset
verify 均成功。Baseline 40 輪全為 normal；treatment 為 20 normal、2 degraded、18 restored。
兩組初始及前 20 輪的 Root validation accuracy／loss 完全相同。treatment 第 20 個 outcome 於
16:28:53.012353 UTC；Host 在 16:28:55.146406 UTC 確認 primary Guest `SIGSTOP`，
16:29:03.398877 UTC 完成兩邊 hard-stop。第 21、22 個 accepted rounds 只由 Area B／C 成功貢獻，
replacement 從第 23 個 accepted round 開始貢獻。以 effective fault time 計，failure-detected、
replacement-ready、首次 accepted contribution 分別約為 298.0、298.8、317.5 秒；這些只是本次觀測值。

故障前第 20 輪兩組 validation accuracy 均為 37.60%；degraded 第 21／22 輪，baseline 為
37.50%／37.85%，treatment 為 36.00%／36.50%，且 treatment loss 較高。replacement 首次貢獻的
第 23 輪，treatment validation accuracy 為 42.55%、baseline 為 38.25%；第 40 輪分別為
40.35%／40.15%。完整官方 test 的 final accuracy 分別為 baseline 3,862／10,000（38.62%）、
treatment 3,874／10,000（38.74%），相差 0.12 個百分點。單一配對只能描述這次故障與恢復軌跡，
不能宣稱 replacement 提升了 CIFAR-10 模型品質或有統計顯著性。四組正式執行的原始 run 保留供
User Review；後續圖表與報告仍未完成，因此正式比較保持 open。

## 13. CIFAR-10 μ=0.1 診斷實驗（結果待 User Review）

四組正式 run 完成後，以保存的 CIFAR-10 final models 在同一份每類 200 筆的固定 validation 上作
離線逐類評估。無故障 baseline 的 0–4 類平均 accuracy 為 66.4%、5–9 類為 13.9%；替換組
分別為 66.3%、14.4%。兩組整體 accuracy 為 40.15%、40.35%，與第 40 輪紀錄相同。
這確認了類別偏向，但不能單憑它判定 non-IID、聚合或模型能力何者為因。此診斷不改寫前述
四組 run，也不把逐類結果加入第一版正式比較的原訂驗收。

使用者已確認下一步先試一次 CIFAR-10 無故障、`proximal_mu: 0.1` 的獨立診斷 run，
其餘資料切分、初始模型、訓練設定與 40 accepted rounds 維持現有 baseline；主要以固定
validation 的逐輪整體與最終逐類結果，和原本 `proximal_mu: 0.01` baseline 作探索性對照。
這不是原四條件中的第五個正式配對，也不因單次結果宣稱 μ 是類別偏向的原因。
既有 runner 會在完成時產生官方 test 結果，但此次調參判斷不以該結果選擇 μ。

目前 `proximal_mu` 位於共用 `TESTBED` 的 Root 與三個 Branch group strategy；
`protocol_topology()` 將它複製到 generated topology，scenario 尚無覆寫欄位。
直接修改共用 `TESTBED` 會同時改變所有 scenario 的選定來源；複製一整份部署設定則會
重複維護 topology、identity 與 placement。使用者已確認在現有 scenario 的 `training` 增加選填
`proximalMu`，由現有 renderer／checker 對本次 selected scenario 的 Root 與 Branch group
strategy 使用該值；未指定者仍沿用 `TESTBED` 既有值。這只擴充既有資料來源與 pipeline，
不改 component contract、VM、dataset 或 runner。原四組正式比較仍保持待 User Review。

本次在 `5G_NWDAF_Infrastructure` 的既有 `configlib.py`、renderer、checker 與 runner contract
加入選填覆寫，另建 CIFAR-10 診斷 scenario；未指定 μ 的舊 scenario 保持 0.01。Owning
runtime-inventory／experiment tests 通過，診斷 config／dataset checks 通過。生成拓樸的 Root 與
三個 Branch group 均為 0.1，原 TESTBED 與正式 baseline 仍為 0.01；新舊 dataset 目錄逐檔相同。
Host-context preflight 通過，當時 GPU free 9,988 MiB，高於 8,192 MiB floor；free swap 0 MiB
仍是監測警告。四台既有 VM 的 Host process／provider 狀態一致且均 running，沒有重建 VM。

`cifar10-mu01-baseline-20260914-a` 以新的 config／run identity 完成 40 個 normal accepted
rounds，沒有 rejected round 或 runner failure；final model 已保存，Guest／container services 已停止，
selected exact reset verify 通過。兩次初始 validation 均為 9.95%，第 10／20／30／40 輪的
固定 2,000 筆 validation accuracy，原 μ=0.01 baseline 為 32.90%／37.60%／39.70%／40.15%，
μ=0.1 為 25.05%／29.90%／32.55%／33.10%。用保存的 final models 在同一 GPU 路徑上
離線逐類核對，0–4 類平均從 66.4% 降為 38.1%，5–9 類平均從 13.9% 升為 28.1%；
類別落差縮小，但整體 accuracy 下降 7.05 個百分點。官方 test 結果雖由既有 runner 產生，
本次 μ 診斷不以它調參或作效果主張。

本次與原 baseline 的 PyMTLF revision、GPU identity、來源資料與初始 validation 一致，
但 Docker image ID 不同；兩個 image 的部分 filesystem layers 也不同。故這只是探索性
對照，不能將差異嚴格歸因於 μ；若要作控制變因的模型品質結論，須另行決定如何固定
image artifact 並建立新配對，不把本次 run 混入原四條件的正式結果。

## 14. CIFAR-10 全類別不等量切分（資料準備待 User Review）

Section 13 的逐類結果顯示原本只有五類／Leaf 的配置有明顯類別偏向；單次 μ=0.1
診斷雖縮小類別落差，整體 validation accuracy 卻下降，且 image artifact 不同，
不能據此選定新的 μ。使用者決定先試每個 Leaf 均持有十類、但各類數量不同的
soft label-skew 配置。這是原四組正式 run 以外的新實驗版本，不回寫或替換既有結果。

沿用 CIFAR-10 的 `partition.seed: 42`、每 Leaf 8,000 筆、每類 200 筆 train-derived
validation、每類 4,800 筆 Leaf training，以及完整官方 test。下表數字是單一 Leaf
對該組每個類別的筆數；同 Area 的兩個 Leaves 使用相同配額，但 source indices 互斥。

| Leaves | 類別 0–4 各類筆數 | 類別 5–9 各類筆數 |
| --- | ---: | ---: |
| A1、A2 | 1,300 | 300 |
| B1、B2 | 800 | 800 |
| C1、C2 | 300 | 1,300 |

每類六個 Leaves 的配額合計為 4,800，故在目前 generator 的「逐類以同一 seed
打亂、先留 validation、再按 Leaf 名稱排序分配」規則下，新舊切分可使用相同的
48,000 筆 training 來源、2,000 筆 validation 與完整官方 test；只有 training 樣本
分配到哪個 Leaf 改變。原有 `leafLabels` scenario 必須維持相同 seed 下的逐 Leaf
source indices 與既有輸出，不能因新模式而重新排列舊資料。

在既有 scenario 的 `partition` 增加選填 `leafClassCounts`，表示 Leaf 名稱至逐類整數
配額的 mapping，與 `leafLabels` 互斥；兩者都未指定時保留既有十類等量模式。
每個 Leaf 的配額總和必須等於 `samplesPerLeaf`，實際官方來源在保留 validation 後
必須足以供應每類總配額。新模式只調整現有 `image_scenario_contract()`、
`build_split()`、`validate_output()` 與同一 dataset generate／check 入口；維持
Leaf NPZ、validation、held-out、split manifest、config render／check、stage 與
runner 的既有 contract。不在共用程式或永久測試固定本次 Leaf 名稱和配額。

先建立共用此切分的 CIFAR-10 無故障與 Branch replacement 兩份診斷 scenario：
40 accepted rounds、5 local epochs、既有 batch size／learning rate、TESTBED 預設
`proximal_mu: 0.01`，treatment 仍在第 20 個 accepted round 後停止 Area A primary。
兩份 scenario 除名稱、fault 及必要 observation 外保持一致。此階段先批准計畫、
資料工具、scenario、資料／config 產物及聚焦驗證；長時間訓練其後另獲批准，結果見 Section 16。
原四組正式 run 與 μ=0.1 診斷資料保持不變。

驗證先用既有 owning dataset test 的小型來源，直接證明逐類配額、來源互斥與
相同 seed 的確定性；再用實際 CIFAR-10 cache 產生兩份 scenario 的資料並核對
各 Leaf／類別筆數、全體 training 來源集合、validation／held-out 來源索引、
native loader 與 config checker。這些只證明資料準備；後續若執行配對訓練，
仍須另行核對相同 component／image／GPU identity、capacity、逐輪 evidence 與
exact cleanup，不能把本節靜態結果當作訓練通過。

已在既有 `configlib.py`、`image_dataset.py` 與 owning dataset test 擴充逐類配額，
建立兩份診斷 scenario 及其 generated config／dataset。小型來源測試、runtime
inventory test、兩份資料與 config checker、資料工具與測試檔的聚焦 lint 均通過。
使用同一 seed 重新產生舊 `leafLabels` scenario，得到與原持久 split manifest
完全相同的結果；新舊切分的 training 來源索引集合均為同一 48,000 筆，
validation／held-out 的來源索引與 NPZ 檔案相同。新配對的兩份 dataset 逐檔相同，
scenario 的 workload、partition 與 training 相同；新設定的各 Leaf 配額經
native loader／manifest checker 核對。未啟動 VM、container 或訓練，也未改動
原四組及 μ=0.1 診斷產物；本段仍待使用者 review，不移入 verified records。

## 15. 配對資料共用與既有產物整理（待 User Review）

已完成的 MNIST、CIFAR-10 正式 baseline／replacement 配對，各自的兩份 dataset
目錄逐檔相同，但原 `scenario`、generated config 與 `run.json` 均記錄各自的舊路徑。
這些是既有實驗證據，不追溯加入新欄位或改寫執行設定。先再次確認逐檔相同與
實際路徑後，只讓重複檔案共用實體儲存，保留原本兩個邏輯目錄及所有 run 紀錄；
整理後兩個舊 config 仍須能通過資料與 config 檢查。若無法在不改路徑的情況下
安全共用，停止整理並回報，不以刪除舊目錄換取空間。

尚未訓練的 CIFAR-10 全類別不等量 baseline／replacement 配對則在既有 scenario
的 `partition` 增加相同、選填的 `datasetId`，例如 `cifar10-all-class-skew`。
它是兩個 scenario 共用一份切分的儲存名稱，不是 run identity，也不改變 Leaf
配額或隨機種子；未指定時沿用 `scenario.name` 作為資料目錄，以維持舊配置相容。
沿同一 dataset generate／check、config render／check、Compose mount pipeline
解析該目錄。第一次產生資料；共同目錄已存在時檢查其是否符合選定 scenario，
符合則直接複用，不符合則拒絕覆蓋。兩份新 generated config 均改指同一目錄；
確認共用資料及兩份 config 可用後，清除這組尚未執行的舊重複資料產物。

本項只涉及 `5G_NWDAF_Infrastructure` 的現有資料／config 路徑與兩份新 scenario，
以及本計畫的紀錄；不新增來源資料、runner、hash、VM 或 component 行為，
也不啟動訓練。聚焦驗證須覆蓋共用資料的生成／複用、兩份新 config 的資料與
Compose 路徑、舊 config 的相容，以及歷史配對整理前後的逐檔一致性與路徑保留。

本輪已讓兩份新 scenario 指向 `cifar10-all-class-skew`，並重產兩份 generated
config。第一次 `dataset-generate` 建立共用目錄，第二次回報 `REUSED`；兩份新
config 的 dataset／config／Compose checks 通過。舊 MNIST 與 CIFAR-10 配對在
整理前後均逐檔相同、dataset／config checks 通過；兩個 replacement 目錄的檔案
已改與對應 baseline 共用 hardlink，但原路徑、config 與 run 紀錄未改。
已清除新配對先前按 scenario 名稱產生、且與共用目錄逐檔相同的兩份未執行資料；
不再佔三份實體空間。本節記錄資料儲存及設定準備；其後的新配對訓練見 Section 16，
結果仍待 User Review。

## 16. CIFAR-10 全類別不等量配對執行結果（待 User Review）

2026-09-14 依序使用 `config/local/cifar10-all-class-skew-baseline`、
`config/local/cifar10-all-class-skew-replacement`，執行
`cifar10-skew-baseline-20260914-a` 與 `cifar10-skew-replacement-20260914-a`。
兩組均使用同一 `datasetId: cifar10-all-class-skew`、seed 42、每 Leaf 8,000 筆、
2,000 筆固定 train-derived validation、40 accepted rounds 與 5 local epochs；
seed model、component revisions、PyMTLF image ID 及 GPU identity 相同。
執行前資料與 config checks、approved Host-context preflight 均通過；GPU free 9,988 MiB
高於 8,192 MiB floor，free swap 0 MiB 為監測警告。兩組使用 testbed revision `740e9c7`，
執行時相關 repositories 均為 clean。

兩次 runner 均正常退出；各自的 `run.json` 為 `status=successful`、`finalized=true`、
無 failures。每組有 40 個 Root accepted outcomes、41 次 Root validation（含初始一次），
每次均評估相同的 2,000 筆；final model 與完整官方 10,000 筆 test evaluation 均已保存。
Baseline 40 輪全為 normal，沒有 fault／replacement 事件；treatment 在第 20 輪後
hard-stop Area A primary，得到 20 normal、2 degraded、18 restored。第 21、22 輪只有
Area B／C 貢獻，第 23 輪 replacement 首次貢獻。以 effective fault time 計，
failure-detected、replacement-ready、首次 accepted contribution 分別約為 298.0、
298.9、317.3 秒；這是本次觀測值，不是固定恢復時間保證。

兩組初始至第 20 輪的 validation accuracy／loss 逐筆相同；第 20 輪 accuracy 均為
54.10%。第 21／22 輪 baseline 為 54.30%／55.10%，treatment 為 51.30%／51.75%；
第 23 輪分別為 55.50%／55.55%，第 40 輪為 59.70%／59.85%。完整官方 test 的 final
accuracy 為 baseline 5,806／10,000（58.06%）、treatment 5,812／10,000（58.12%），
相差 0.06 個百分點。這組配對描述故障期間的短暫下降與後續恢復；單次結果不能
證明 replacement 改善模型品質或具有統計顯著性，也不取代原四組正式實驗。

兩次執行均完成 process stop、Guest restart-policy restoration 與 selected exact reset
verify；原始 `events.jsonl`、`run.json` 與 final model 保留在
`runs/protocol-hierarchical/cifar10/` 下各自的 run 目錄。結果待 User Review，不移入
verified records；正式分析圖表與報告尚未完成。
