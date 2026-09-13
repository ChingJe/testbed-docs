# Protocol-driven Hierarchical FL 正式 Branch Replacement 比較實驗計畫

日期：2026-09-13

狀態：Draft／待確認；資料工具已提交，正式訓練條件已有靜態盤點但尚未凍結，不授權正式執行

前置工作：[Multi-host Branch Replacement Experiment Plan](./multi-host-branch-replacement-experiment-plan.md)
已完成雙資料集、四 VM、GPU、八個 accepted rounds 的流程驗證；其結果只證明流程可運作，
不是本計畫的 paired baseline 或模型品質比較。本計畫是後續獨立工作，不改寫該計畫的驗收與紀錄。

正式實驗前的 testbed 改動歸入 [Slice 4 詳細計畫](./slices/slice-4-formal-comparison-preparation-detailed-plan.md)，
依已決定的實驗條件逐輪擴充。共用 runner 的雙模式支援與短 epoch 簡測已完成；第二輪只實作已確認的
資料切分與評估來源，正式訓練條件仍須另行確認，不因資料工具可用而自動獲准執行正式實驗。

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

兩個資料集都採固定、可重建的六個 Leaf training shards。預定採用 label-skew non-IID：每個 Leaf
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
已沒有太多空間同時增加 Leaf 獨立樣本與保留 train-derived validation。第一版建議沿用每 Leaf
8,000 筆，兩資料集均為六個等量 shards，不為了用滿來源資料而提高數量。盤點證實上述五類／Leaf
配額在數量上可行；第二輪已用暫存切分直接驗證 generator，正式 scenario 仍待建立。不能把增加 local epochs、重複抽樣或
資料增強描述為增加了獨立樣本。

Per-round validation 應完整遍歷一份固定、與 Leaf training shards 互斥的 validation set，而非沿用
200 筆 smoke set，也不是每輪遍歷官方 test split。CIFAR-10 在六個 Leaves 共使用 48,000 筆
official-train 樣本時，剩餘 2,000 筆作為固定 validation。MNIST 建議採同一配額：48,000 筆
training、2,000 筆 train-derived validation，其餘 10,000 筆 official-train 不使用。兩資料集的
validation 均為每類 200 筆；此配額已核准供資料工具實作，正式 scenario 資料尚未持久產生。最終模型另用
**完整的官方 10,000 筆 test split**作一次主要 held-out 評估；正式訓練設定凍結前
不得用這份 test 結果挑選配置。
切分沿用目前 scenario 的 `partition.seed: 42`：每類先固定選出 200 筆 validation，再把互斥的
4,800 筆分配給三個 Leaves；同資料集的 baseline／treatment 須得到相同的 source indices。
第二輪改動前的 generator 雖分別輸出 validation 與 held-out artifacts，卻把兩者都取自官方 test split、各只用
200 筆，且要求所有輸出每類別等量；正式切分必須調整來源與檢查規則，不能只把 scenario 的
計數改成 10,000。MNIST 官方完整 test split 各類樣本數並不相等，final evaluation 不得為了等量
而丟棄 test 樣本。

## 4. 時間軸與訓練工作量

目前八輪流程只在第二個 accepted round 後故障，不足以觀察中段替換的前後曲線。第一版正式設定候選
如下；靜態盤點見 Slice 4 Section 6.7，仍須經使用者確認才凍結。同資料集的 baseline／treatment 必須一致，只有
treatment 注入故障。現行每 Leaf 32 local epochs 是先前流程驗證設定，不直接沿用為正式實驗值。

| 資料集 | accepted rounds | 每 Leaf local epochs | treatment 故障時點 |
| --- | ---: | ---: | --- |
| MNIST | 24 | 4 | 第 12 個 accepted round 完成後 |
| CIFAR-10 | 40 | 5 | 第 20 個 accepted round 完成後 |

既有 GPU 紀錄支持保留上述候選作第一版設定；它只能給執行時間量級，不能證明 4／5 epochs 的實際
fault barrier 時序、新 validation 成本或執行當日容量。靜態盤點建議先按此候選凍結，正式 treatment
run 直接核對 barrier 與 replacement；不另加 GPU pilot。不把較弱或較強的 learning-curve 效果
當成修改同一次有效 run 條件的理由。

故障後的 degraded accepted rounds 數量由 production timeout／recovery timing 決定，不預先強制固定；
accepted、rejected、failure-detection、replacement-ready 與首次 replacement contribution 都分別記錄。
Treatment 至少要觀測一次 replacement 成功參與後續 accepted round；若目標輪數結束仍未恢復，
保留不完整結果，不能事後延長單一 run 來冒充原訂方案。若事前盤點顯示候選設定不可行，先修訂並
凍結同資料集的配對設定；若正式結果的效果不明顯，有效 run 仍須保留並報告，後續新設定另立版本。

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

Baseline 與 treatment 使用同一 run-level evidence／checkpoint／collection contract；baseline 不產生虛構的
fault、degraded 或 replacement events，treatment 才要求相應的故障與恢復證據。兩者都須保留完整的
accepted-round 與 validation 對齊資料，才能進行配對比較。

## 6. 實作邊界與驗證順序

此版仍是正式實驗主計畫草案；Slice 4 第一輪 runner 與第二輪資料工具變更已由詳細計畫限定，
正式 scenario、訓練與報告條件仍待後續確認。
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
   在同一 Slice 4 第二輪完成必要的 dataset／evaluation 調整；正式訓練設定另行凍結。執行前以來源索引、
   Leaf／Area 各類筆數、互斥性、native loader 與 validation 全量覆蓋做針對性檢查，不另跑獨立
   流程彩排 pilot，也不預先產生另一份 Slice 文件。
3. 確認容量與時間可行、凍結設定後，各條件先跑一次；第一批正式 runs 同時核對新的 validation
   路徑、baseline 無故障、treatment 中段故障與 priority replacement、collection／evidence 完整性及
   exact reset。任何失敗或不完整 run 保留紀錄，修正後依原凍結設定重跑，不計入有效配對；有效但
   效果不明顯的 run 不因研究結果而剔除。
4. 對等檢查各配對的初始條件、accepted-round 對齊、缺失資料、evaluation 覆蓋與故障時序，
   然後才產生圖表和 report claim。任何未完成或不符合預定條件的 run 另列，不補值或靜默剔除。

## 7. 驗收與 verification matrix

正式比較整體尚未 implementation-ready；下表是正式執行前必須細化的驗收邊界，
不表示目前已有正式 runs 的對應 evidence。Slice 4 第一輪 runner 擴充已有獨立的 review 與簡測紀錄。

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

## 8. 待決策與非目標

在正式比較後續輪次進入 implementation-ready 前，尚須確認：

- 已核准的五類／Leaf 配額與兩資料集各 2,000 筆 train-derived validation 已用暫存切分核對；
  正式 scenario 的 source indices、Leaf／Area 分布及 evaluation 覆蓋仍須在執行時直接核對；
- 使用者確認是否按靜態盤點建議凍結 MNIST 24／12／4、CIFAR-10 40／20／5，以及兩組配對的固定 seed；
- 執行當日 GPU／Host 容量 admission、正式 scenario 的 fault barrier 可命中性與新 validation 實際耗時；
- 第一版四次執行的 run identity；建議順序與資料／初始模型一致性核對方式見 Slice 4 Section 6.7。

本輪不比較 Flat FL、FedAvg／FedProx、Branch 聚合頻率、GPU 效能成本或「故障但不替換」；
不改 5g-viz、5GC user plane、component recovery semantics，也不把流程驗證紀錄當成正式比較資料。
在上述正式實驗條件完成決策與 user review 前，本計畫保持 Draft，不進行正式 runs。Slice 4 第一輪
runner 擴充及短 epoch 簡測已完成；第二輪資料工具實作也不使未決的正式訓練條件自動獲准。
