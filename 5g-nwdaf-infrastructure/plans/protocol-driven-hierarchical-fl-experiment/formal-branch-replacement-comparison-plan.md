# Protocol-driven Hierarchical FL 正式 Branch Replacement 比較實驗計畫

日期：2026-09-13

狀態：Draft／待確認；正式實驗條件尚未凍結，不授權正式執行；Slice 4 第一輪另依詳細計畫確認

前置工作：[Multi-host Branch Replacement Experiment Plan](./multi-host-branch-replacement-experiment-plan.md)
已完成雙資料集、四 VM、GPU、八個 accepted rounds 的流程驗證；其結果只證明流程可運作，
不是本計畫的 paired baseline 或模型品質比較。本計畫是後續獨立工作，不改寫該計畫的驗收與紀錄。

正式實驗前的 testbed 改動歸入 [Slice 4 詳細計畫](./slices/slice-4-formal-comparison-preparation-detailed-plan.md)，
依已決定的實驗條件逐輪擴充；目前先規劃共用 runner 的雙模式支援與短 epoch 簡測，不預先把後續資料切分、
訓練設定及分析需求寫成已批准的實作範圍。

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
實際 revision、dirty flag、config、dataset seed、裝置及容量資訊。四種條件不等於只有四次 run；需要多少組
paired repetitions 才能支持報告中的統計用語，待 pilot 時間與資源盤點後決定。只有單一配對時，報告僅作
描述性比較，不宣稱統計顯著。

## 3. 資料與 non-IID 設計

兩個資料集都採固定、可重建的六個 Leaf training shards。正式比較傾向採用 **Area／region 層級的
label skew**：讓 Area A 的類別分布與 Area B／C 有可觀測差異，同時保留必要的類別覆蓋與可訓練性；
baseline 與 treatment 必須使用完全相同的切分。切分前先確認每個 Leaf／Area 的樣本數、類別直方圖、
互斥 source indices，以及故障後只剩 B／C 時的類別覆蓋。具體 skew 強度與 seed 在短 pilot 後凍結，
不可看完正式結果才反向調整。Non-IID 是兩種條件共同的實驗背景，不在本輪增加 IID／non-IID 兩套條件。

目前流程每個 Leaf 使用 8,000 筆、共 48,000 筆訓練樣本；MNIST 官方 train／test 為
60,000／10,000，CIFAR-10 為 50,000／10,000。CIFAR-10 在維持六個互斥 Leaf shards 的前提下，
已沒有太多空間同時增加 Leaf 獨立樣本與保留 train-derived validation。不能把增加 local epochs、
重複抽樣或資料增強描述為增加了獨立樣本。正式每 Leaf 數量與 validation 留存量須一起盤點後決定；
本版不承諾提高每 Leaf 樣本數。

Per-round validation 應完整遍歷一份固定、與 Leaf training shards 互斥的 validation set，而非沿用
200 筆 smoke set；候選做法是從官方 train split 預留 2,000 筆供兩資料集使用，實際數量與類別分布
須經資料供給及評估成本驗證。最終模型另用**完整的官方 10,000 筆 test split**作一次主要 held-out
評估，並保留各類別樣本數；正式參數與 non-IID 切分凍結前不得用這份 test 結果挑選配置。
目前 generator 把 validation 與 held-out 都取自官方 test split、各只用 200 筆，且要求兩者每類別
等量；正式切分必須明確調整這個來源與檢查規則，不能只把 scenario 的計數改成 10,000。

## 4. 時間軸與訓練工作量

目前八輪流程只在第二個 accepted round 後故障，不足以觀察中段替換的前後曲線。暫定以每次
**24 個 accepted rounds**、treatment 在 **第 12 個 accepted round 完成後**觸發故障作為 pilot 候選；
baseline 跑相同 24 輪但不注入故障。兩個數值尚未凍結，須先以 GPU pilot 的每輪耗時、總執行預算、
fault barrier 可命中性及 replacement 可恢復輪數檢查。現行每 Leaf 32 local epochs 是流程驗證設定，
正式實驗是否沿用也須在 pilot 前決定；baseline／treatment 不得採不同設定。

故障後的 degraded accepted rounds 數量由 production timeout／recovery timing 決定，不預先強制固定；
accepted、rejected、failure-detection、replacement-ready 與首次 replacement contribution 都分別記錄。
Treatment 至少要觀測一次 replacement 成功參與後續 accepted round；若目標輪數結束仍未恢復，
保留失敗結果，不能事後延長單一 run 來冒充原訂方案。若 pilot 證明 24／12 不合適，先修訂計畫與
四條件的共同設定，再開始正式 run。

## 5. 觀測、分析與資料保存

沿用目前 node-local structured JSONL、`events.jsonl`、`run.json`、Root 持久 final model 與可獨立重試的
collection。每個 accepted round 的 Root validation loss／accuracy 與 `ROOT_ROUND_OUTCOME` 一對一對齊；
比較圖以相同 accepted-round index 疊合 baseline／treatment，並在 treatment 標註故障、degraded 與
restored 區間。主要另報 final full-test loss／accuracy、故障至偵測／ready／首次貢獻的時間、各階段
accepted rounds 數量及每輪成功 Branch identities。

Non-IID 效果至少需要保存各 Area／Leaf 的類別分布，並評估 final model 的 per-class 指標是否可從
持久模型與完整 test set 離線計算。現有 `MODEL_EVALUATION` 與 final evaluator 只直接提供整體
loss／accuracy；**本版不宣稱已有 per-round per-class 曲線**。若正式研究問題要求觀察 Area A 類別
在 degraded 區間的逐輪變化，須先盤點可用的逐輪模型與 evaluator 邊界；需要 component 新增輸出時，
先取得跨 repository contract 決策，不能在報告階段從整體 accuracy 推論出類別效果。

正常監控只使用增量 structured events 與精簡 milestone；一般 journald／Docker logs 僅在精確故障時間窗
按需存入 `diagnostics/`，不持續串流。Training 與 collection 保持獨立 checkpoint；後處理失敗可用同一
run identity 重試，不因作圖或評估 bug 重新訓練。正式分析輸出可在全部 runs 完成後由保存資料離線產生，
不綁在訓練 runner 的成功路徑上。

Baseline 與 treatment 使用同一 run-level evidence／checkpoint／collection contract；baseline 不產生虛構的
fault、degraded 或 replacement events，treatment 才要求相應的故障與恢復證據。兩者都須保留完整的
accepted-round 與 validation 對齊資料，才能進行配對比較。

## 6. 實作邊界與驗證順序

此版仍是正式實驗主計畫草案；第一輪 runner 變更已在 Slice 4 詳細計畫限定，其他程式變更待後續盤點。
預期 owner 是 `5G_NWDAF_Infrastructure/` 的 scenario validation、dataset preparation、現有 renderer／runner、
evidence checker 與 tests；`testbed-docs/` 保存本計畫、後續實驗定義和 reviewed records。
NWDAF／PyMTLF／NRF／ADRF 原則上沿用現有 revisions，不預設修改 component contract。

Slice 3 後續 review 已將共用 validator 中屬於當次驗收的固定輪數、樣本數、local epochs 與觀測頻率
改為由 selected scenario 決定，追補已完成 review。目前
`fl-training-start` 是以 `RUN_ID` 送出 request／查詢 status 的低階控制入口，不負責實驗 lifecycle 或
evidence；現有 Branch replacement runner 直接使用相同的底層 `fl-control` 功能，並另外負責 runtime、
fault、observation、collection 與 cleanup。正式比較應擴充**這一個既有 runner**，讓它依 scenario
執行無故障 baseline 或故障替換 treatment，而不把 `fl-training-start` 改成另一個 run-level workflow。
對外 run／collect 命令採不綁定 replacement 的名稱；既有 target 的過渡方式見 Slice 4 詳細計畫，
只能有一套 runner implementation 與 checkpoint 語意。

正式 baseline 與更長 treatment 必須由同一現有 `TESTBED` + scenario +
`CONFIG_DIR` pipeline 支援，不能只增一組 YAML 或複製第二個 runner。規劃細化時要逐段標明現有
dataset → render／validate → stage／activate → start → trigger → observation → collection → stop／reset
哪些原樣複用、哪些因正式比較而調整；provider、wrong-config、容量與 exact reset 防護維持不變。

建議驗證順序：

1. 先完成 Slice 4 的共用 runner 雙模式擴充，沿用既有 short normal scenario 做一次短 epoch GPU 簡測；
   treatment 保留原有流程與直接回歸證據。這只是 runner 接線驗證，不作正式比較，也不藉此凍結資料與輪數。
2. 再凍結資料切分、skew、round、故障時點、local epochs、paired seed 與報告指標；完成相應的
   deterministic split、schema 與 evidence 驗證。後續變更繼續記入同一 Slice 4 計畫，不預先產生另一份 Slice 文件。
3. 以縮短輪數的 GPU pilot 驗證完整 validation 路徑、baseline 不注入故障、treatment 中段故障、
   priority replacement 與 collection-only retry；final test evaluator 以獨立 fixture 驗證讀取／輸出契約，
   不使用官方 test 結果挑選正式配置。Pilot 不併入正式比較。
4. 確認容量、執行時間與資料切分可行後，以凍結設定跑四種條件及核准的 paired repetitions；
   逐次保存 source／runtime identity、完整 structured evidence 與 exact cleanup 結果。
5. 對等檢查各配對的初始條件、accepted-round 對齊、缺失資料、evaluation 覆蓋與故障時序，
   然後才產生圖表和 report claim。任何未完成或不符合預定條件的 run 另列，不補值或靜默剔除。

## 7. 驗收與 verification matrix

正式比較整體尚未 implementation-ready；下表是正式執行前必須細化的驗收邊界，
不表示目前已有對應 evidence，也不阻止先 review Slice 4 的第一輪 runner 擴充。

| 驗收項目 | 靜態／受控驗證 | 正式執行 evidence |
| --- | --- | --- |
| 四條件可由同一 pipeline 選取 | scenario／render／同一 runner tests 覆蓋 baseline 與 treatment；低階 `fl-training-start` 語意不變，無第二套 selector／runner | 同一四 VM／十一 process topology，baseline primary 全程健康且無虛構故障事件，treatment 由 Root 依 priority 替換 |
| 資料互斥且配對一致 | 來源索引、類別直方圖、seed、完整 validation／test count 與 native loader checks | 配對 runs 保存一致的 split 與初始模型資訊，完整官方 test 僅作凍結後的 final evaluation |
| 中段故障與 learning curve 可重建 | fault barrier、accepted／rejected、phase classifier 與 metric alignment tests | Treatment 有故障、degraded 與至少一次 restored contribution；兩條曲線按 accepted round 對齊 |
| Evidence 與結果可審查 | collection retry、final artifact、run completeness 與缺失資料 checks | `events.jsonl`、`run.json`、final model、完整 test 結果及清楚標示的未完成 run |
| Runtime safety 保持 | 現有 provider guard、wrong-config、容量、inventory、exact reset tests | Approved host context 的 actual inventory、GPU、stop／reset 與 selected state 核對 |

正式比較的最低可交付物是每資料集至少一組有效 baseline／treatment 配對、對應逐輪曲線、
完整官方 test 的 final 結果，以及 treatment 的故障與恢復 timeline；這只允許描述性結論。
重複次數、統計方法與任何模型品質門檻尚未核准，不能從最低交付物推導顯著性或改善保證。
未通過矩陣或缺少 real-environment evidence 時保持 open state，不將 run 收入 verified record。

## 8. 待決策與非目標

在正式比較後續輪次進入 implementation-ready 前，尚須確認：

- region-level label skew 的精確規則、強度、Leaf 樣本數及 validation 數量；
- 正式總 accepted rounds、故障 barrier、local epochs 與 pilot 的時間／GPU 預算；
- 每資料集 paired repetitions 數量，以及報告只作描述或要求統計推論；
- 是否必須有 per-round per-class 指標；若是，先確認逐輪模型／component 記錄的實際可行性；
- 最終 success criterion 與失敗／不完整 run 的報告方式。

本輪不比較 Flat FL、FedAvg／FedProx、Branch 聚合頻率、GPU 效能成本或「故障但不替換」；
不改 5g-viz、5GC user plane、component recovery semantics，也不把流程驗證紀錄當成正式比較資料。
在上述正式實驗條件完成決策與 user review 前，本計畫保持 Draft，不進行正式 runs。Slice 4 第一輪
runner 擴充及短 epoch 簡測可在其詳細計畫通過 review 後先行；它們不使未決的資料與訓練條件自動獲准。
