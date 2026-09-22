# Hierarchical FL 論文實驗

本分類保存論文 E0、E1、E2a、E2b 實驗在新版 testbed 端的部署、執行與證據計畫。詳細計畫隨工作進度逐份建立，
不預先建立所有 implementation slice 文件。

目前文件：

- [Testbed 實驗就緒盤點](./Testbed%20Experiment%20Readiness%20Inventory.md)：對照目前 component 能力與 testbed source，整理正式實驗前仍需完成的整合、runner、證據與五 seed 支援。
- [Testbed 實作順序與 Slice 安排](./Testbed%20Implementation%20Sequence%20and%20Slices.md)：將盤點的七項缺口整理為三個依序 review 的實作段落與決策關卡。
- [Slice 計畫](./slices/README.md)：隨進度逐份建立各 slice 的討論與詳細計畫。
- [E0–E2b 正式實驗資料與環境整理](./E0-E2b%20Formal%20Experiment%20Materials%20Draft.md)：彙整本批 40 組有效 run 的實驗條件、環境、主要結果與原始證據位置，供論文撰稿與 review。

## 責任邊界

本分類負責：

- 將已確認的 component 能力映射到 testbed topology、VM／Host process placement 與 runtime inventory；
- 定義同一 pipeline 下的 E0–E2b scenario、paired seeds、資料與初始模型輸入；
- 規劃故障注入、啟動、觀測、收集、停止、reset、失敗保留與重跑流程；
- 保存正式實驗的 run metadata、逐節點原始紀錄、模型結果及離線分析交接要求；
- 定義 testbed real-environment acceptance，以及無法由 component-local tests 證明的整合證據。

Go NWDAF、PyMTLF 的 protocol contract、訂閱生命週期、逐節點紀錄語意、direct Leaf reparenting 與 mixed-depth
aggregation 由 `nwdaf-docs` 的對應論文實驗文件及 component source 擁有。本分類引用其已確認結論，不在
testbed 文件另行定義一套 component schema 或行為。論文草稿與老師提供的調整建議可作需求背景，但若與
`nwdaf-docs` 已記錄的後續決策不同，以後者為準。

## 與既有實驗的關係

同層既有 Branch Replacement 比較計畫保存已完成的單次 MNIST／CIFAR-10 E0／E1 配對、testbed pipeline 與
歷史原始證據。這些內容可作後續部署與 runner 的 baseline，但不自動成為 E0–E2b 五 paired seeds 的正式結果，
也不證明 E2a／E2b 已通過實驗室 testbed 驗收。

後續文件應只記錄 testbed 擁有的差異、決策與驗證；component 實作狀態需在使用前重新由 owning repository 與
`nwdaf-docs` 確認。正式實驗結果在 required evidence 完整並通過 review 前留在 plan；通過後再整理至 records。
