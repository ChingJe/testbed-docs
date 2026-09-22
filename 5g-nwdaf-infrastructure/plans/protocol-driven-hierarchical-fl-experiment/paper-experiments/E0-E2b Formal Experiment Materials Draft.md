# E0–E2b 正式實驗資料與環境整理（草稿）

日期：2026-09-22。狀態：Review Confirmed；供論文實驗章節撰稿，尚未作為 `records/` 歸檔。除非另註，時間均為 UTC。

本文只整理本次 **MNIST／CIFAR-10 × E0／E1／E2a／E2b × seed 1–5** 的正式實驗。舊版[第四章材料草稿](../chapter-4-experiment-materials-draft.md)僅作環境欄位與呈現方式參考；其中 2026-09-13 至 14 日的單次實驗數值、300 秒 round timeout 與舊 CIFAR-10 五類／Leaf 切分，均不混入本文。實驗定義與統計口徑以 `nwdaf-docs` 的 E0–E2b 實驗文件及本分類的[Slice 3 計畫](./slices/Slice%203%20Paired%20Seeds%20and%20Offline%20Analysis%20Initial%20Plan.md)為準；實際執行值以每個 `run.json`、逐節點 JSONL 和離線輸出為準。

## 1. 實驗範圍與證據層級

本批在 2026-09-21 19:41 至 2026-09-22 02:00 UTC 間完成 40 個 `successful`、`finalized` 的正式 run：每個資料集、條件各有五個 paired seeds，MNIST 每次 24 個 accepted Root rounds，CIFAR-10 每次 40 個。40 個 run 均保存 Root 最終模型、官方 test 評估、Root 及各節點事件，並完成 process stop 與 guarded reset。MNIST seed 3／E0 曾有兩筆啟動失敗嘗試，原始檔仍供追溯，但**不屬於 40 個正式有效 run，亦未納入統計**。

下文區分三種證據：`run.json` 是單次執行的選中情境與 runtime snapshot；逐節點 `observations/*.jsonl`／`events.jsonl` 是當時的訓練、訂閱與拓樸事件；Host OS 及 testbed source 的數值是 2026-09-22 再查的機器 inventory／部署宣告，**不是每輪資源用量或逐 run 的 Host 硬體快照**。

## 2. 實驗環境

### 2.1 實體 Host 與軟體基礎

| 項目 | 本批使用或核對的資訊 | 證據界線 |
| --- | --- | --- |
| Host OS／CPU | Ubuntu 20.04.6 LTS；Intel Core i9-12900K；24 個 Linux logical CPUs | 2026-09-22 Host inventory；非 `run.json` 逐次快照 |
| Host 記憶體 | `lsmem` 為 64 GiB online；`/proc/meminfo` 的 `MemTotal` 為 65,679,944 kB，約 62.6 GiB | 前者是配置規格，後者是 Linux 可見總量；都不是當時 `available`，差額也不能歸因於 swap |
| Host swap | 總量約 2 GiB；本次事後檢查為 0 free | 本批 preflight 曾提示 free swap 為 0；不能當作每輪記憶體使用量 |
| GPU | 單張 NVIDIA GeForce RTX 3080，10,240 MiB VRAM，driver `535.183.01` | 40 個 `run.json.gpuAdmission` 均記錄同一 GPU；每次 admission 記錄 9,988 MiB free、8,192 MiB floor，非訓練期間量測 |
| FL runtime | PyMTLF Docker image；`uv.lock` 選定 Linux x86_64 的 PyTorch `2.5.1+cu121` | 套件版本依當時 source lock；實際 image／component revision 見下表與各 run metadata |
| VM provider／Guest | Vagrant + VirtualBox；`ubuntu/jammy64` box `20241002.0.0`，Guest 宣告 Ubuntu 22.04 | `testbed.protocol-hierarchical.yaml`／`Vagrantfile`；本文不以 provider command 重查已執行的 VM |

40 個有效 run 的 repository revision 及 dirty flag 一致，均在各自 `run.json.repositories` 保存：

| Repository | Revision | Dirty |
| --- | --- | --- |
| `5G_NWDAF_Infrastructure` | `a5f10f4ef6cba27be524fdc052721056e0c6e58d` | false |
| `ML/PyMTLF` | `c31b8b6129d355f4ba65d927a1f97d7b04f95739` | false |
| `NFs/nwdaf` | `de8b385977eeed3c641564de08ff1fc6390d62d3` | false |
| `NFs/nrf` | `0dd4024d4ab75b6630e04901968228b9b9718cf5` | false |
| `NFs/adrf` | `905f0599f68fe389bba14ed56db0ef9abeab5ccd` | false |

### 2.2 VM、網路與 process placement

| VM | vCPU／RAM／宣告磁碟 | Management／SBI IP | Guest 主要服務 |
| --- | --- | --- | --- |
| `core` | 2／3,072 MiB／24 GiB | `192.168.56.10`／`192.168.57.2` | MongoDB、NRF、ADRF、Root NWDAF |
| `path-a` | 2／2,048 MiB／20 GiB | `192.168.56.11`／`192.168.57.3` | Area A primary Branch、預部署 replacement Branch A*、A1／A2 Leaves |
| `path-b` | 2／2,048 MiB／20 GiB | `192.168.56.12`／`192.168.57.4` | Area B Branch、B1／B2 Leaves |
| `path-c` | 2／2,048 MiB／20 GiB | `192.168.56.13`／`192.168.57.5` | Area C Branch、C1／C2 Leaves |

四台 VM 宣告合計 8 vCPU、9,216 MiB RAM、84 GiB 虛擬磁碟容量；vCPU 與磁碟容量不是獨占 Host 核心或實際磁碟佔用。Management `192.168.56.0/24` 與 SBI `192.168.57.0/24` 均為 private network。這批只用 MongoDB、NRF、ADRF、NWDAF 與 PyMTLF 支持 FL 流程，沒有啟動 UPF／UE／完整 5GC user-plane 實驗；optional `webconsole` 不屬於必要訓練鏈。

部署宣告有 1 Root、4 個 Branch 身分（含 A*）、6 Leaves 的 NWDAF／PyMTLF 配對。PyMTLF 容器在 Host 上執行，Go NWDAF 服務在 Guest；A* 是預部署候選，但只有 E1 會啟動並接手，E0／E2a／E2b 不把它當成訓練參與者。各容器的 CPU／RAM **上限**：Root 1 vCPU／1,024 MiB；四個 Branch 各 0.75 vCPU／768 MiB；六個 Leaves 各 1 vCPU／2,048 MiB。Root 與六個 Leaves 使用同一張 GPU 的 `cuda:0`，四個 Branch 使用 CPU；上限不是記憶體保留或實際消耗。完整 NF 身分、SBI endpoint、container port 與 volume 宣告以 `5G_NWDAF_Infrastructure/testbed.protocol-hierarchical.yaml` 為準，各 run 的 `runtimeInventory`／`resolvedDevices` 保存選中 runtime 內容。

### 2.3 聚合、policy 與 timeout

正常拓樸是 `Root→A/B/C→各 Area 兩個 Leaves`：Branch 先依兩個 Leaf 的樣本數聚合，Root 再依直接參與者所代表的樣本數聚合。Root／Branch 使用 `fedProx`、`proximal_mu=0.01` 與 `sampleWeighted`；Branch 每輪向 Root 回報一次，不設定多個 Branch-local 聚合輪次後才上傳。Area A primary／A* 的 priority 分別為 100／50，Root 在 primary 不可用後依 priority 選取候選。

Root policy 宣告 `minAvailableNodes=2`、`minTrainNodes=2`、`fractionTrain=1`、`acceptFailures=true`、`minCompletionRate=0.66`；各 Branch 對自己的兩個 Leaves 要求兩者參與（`minAvailableNodes=2`、`minTrainNodes=2`、`minCompletionRate=1`、`acceptFailures=false`）。E2a／E2b 的 Root 直接參與者可同時含 Leaf 與 Branch，不能將其解讀成固定「三個 Branch」的比例。實際 `roundTimeoutSeconds=60`、`preparationTimeoutSeconds=300`；另有 MTLF backend request timeout 120 秒與單次最多 300 秒的 delay extension policy。這些是不同等待邊界，不能把 300 秒 preparation timeout 寫成 round timeout。

## 3. 資料、模型與實驗條件

### 3.1 配對輸入與資料切分

兩個 workload 各用 seeds `1–5`。同 workload／seed 的四條件使用同一份資料切分與同一份真正由該 seed 初始化的模型權重，並固定 Leaf shuffle 種子、訓練設定和 accepted-round 預算；不同 seed 的資料與模型各自生成。共計十份生成後的切分與十份初始模型。每個 run 的 `scenario.partition.datasetId`、`scenario.workload.seedArtifactKey` 和設定快照是實際配對依據，不以 run name 猜測。

| 資料集 | 官方 train | 六 Leaves 訓練 | Root validation | 官方 test／final held-out | 未使用的 train |
| --- | ---: | ---: | ---: | ---: | ---: |
| MNIST | 60,000 | 48,000；各 Leaf 8,000 | 2,000；每類 200 | 10,000；完整官方 test | 10,000 |
| CIFAR-10 | 50,000 | 48,000；各 Leaf 8,000 | 2,000；每類 200 | 10,000；完整官方 test | 0 |

Validation 從官方 train 留出，與六個 Leaf 的訓練來源索引互斥；Root 在初始模型及每個 accepted round 後評估完整 2,000 筆。官方 test 不參與訓練／逐輪 validation，只對完成模型做一次 final held-out 評估。實際來源索引與各 Leaf histogram 存在 `.generated/image-datasets/<datasetId>/split-manifest.yaml`；每個 `run.json.leafSplitSummary` 另保存該 run 的分割摘要。

| Workload／Leaves | 每個 Leaf 的類別配額 | 每 Leaf 總數 |
| --- | --- | ---: |
| MNIST：A1、A2、B1 | 類別 0–4 各 1,600；5–9 為 0 | 8,000 |
| MNIST：B2、C1、C2 | 類別 5–9 各 1,600；0–4 為 0 | 8,000 |
| CIFAR-10：A1、A2 | 類別 0–4 各 1,300；5–9 各 300 | 8,000 |
| CIFAR-10：B1、B2 | 十類各 800 | 8,000 |
| CIFAR-10：C1、C2 | 類別 0–4 各 300；5–9 各 1,300 | 8,000 |

這些是每個 Leaf 對「每一類」的筆數，不是整個 Area 的合計。CIFAR-10 本批使用全十類、但各類配額偏斜的切分；不是舊草稿的五類／Leaf 正式版。E2b 在 A2 永久不可用、A1 接回後，Root 每輪實際涵蓋 40,000 筆而非 48,000：MNIST 類別 0–4 各 3,200、5–9 各 4,800；CIFAR-10 類別 0–4 各 3,500、5–9 各 4,500。此為資料**覆蓋量**，不是逐類分類準確率。

### 3.2 模型、訓練與故障矩陣

兩個 workload 均用 PyMTLF `SmallCNN`；影像以 `uint8→float32÷255` 輸入，最後輸出十類 logits，不在模型內加 softmax。

| 順序 | 模型操作 | MNIST 輸出（不含 batch） | CIFAR-10 輸出（不含 batch） |
| ---: | --- | --- | --- |
| 輸入 | 影像 | `1×28×28` | `3×32×32` |
| 1 | `Conv2d(input_channels→32, kernel=3, padding=1)`、ReLU | `32×28×28` | `32×32×32` |
| 2 | `MaxPool2d(2)` | `32×14×14` | `32×16×16` |
| 3 | `Conv2d(32→64, kernel=3, padding=1)`、ReLU | `64×14×14` | `64×16×16` |
| 4 | `AdaptiveAvgPool2d(1)`、Flatten | `64` | `64` |
| 5 | `Linear(64→10)` | `10` | `10` |

Leaf 本地訓練使用 Adam、cross-entropy loss 與 shuffle；FedProx 項相對本輪收到的參考全域模型。Root validation batch size 為 128，與 Leaf training batch size 16 不同。MNIST seed model ID 為 `1001`，CIFAR-10 為 `1002`；同 workload 的不同 seed 在獨立 run／reset 後共用 model ID，但使用不同的初始權重與 native artifact identity。模型架構依 `ML/PyMTLF/seed_models/image_classification/{mnist,cifar10}/model.py`；有效訓練值依 `run.json.scenario`。

| Workload | Accepted Root rounds | Local epochs／Leaf／round | Leaf batch size | Learning rate | Root μ | 故障邊界 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| MNIST | 24 | 4 | 16 | 0.001 | 0.01 | 第 12 個 accepted round 後 |
| CIFAR-10 | 40 | 5 | 16 | 0.001 | 0.01 | 第 20 個 accepted round 後 |

| 條件 | 故障與修復設定 | 修復後 Root 直接參與者／資料覆蓋 |
| --- | --- | --- |
| E0 | 不注入故障；A* 不接手正常的 A | A、B、C；48,000 筆 |
| E1 | 停止 A；A* 接回 A1、A2 | A*、B、C；48,000 筆 |
| E2a | 停止 A，不使用 A*；A1、A2 改掛 Root | A1、A2、B、C；48,000 筆 |
| E2b | 停止 A 與 A2；僅 A1 改掛 Root | A1、B、C；40,000 筆 |

Fault controller 以 `SIGSTOP` 先使目標不可回報，再執行 hard stop；Host 紀錄的 `effectiveAt` 是操作確認時間，不宣稱是 Guest kernel 發出訊號的精確時刻。Root 依 round failure／timeout 發現 A 不可用，故障後先可能有只靠 B／C 的 accepted rounds；replacement／reparenting 的 readiness 與首次出現在 Root accepted outcome 中是兩個不同時點。Fault 條件的 observation poll 為 250 ms、heartbeat 為 30 s。

## 4. 正式結果摘要

下表為五 seed 的平均值。Validation endpoint 是最後 accepted round 在固定 2,000 筆上的結果；final test accuracy 是另一份官方 10,000 筆資料的評估。`AUC₁₂` 是名義故障邊界後連續 12 個 accepted-round validation accuracy（以 0–1 比例）的離散和，非 ROC AUC；E0 用相同輪次對齊。`paired Δ` 先計同 seed 條件值減 E0，再對五筆差值取平均，單位為百分點。完整的逐輪數值、雙側 95% Student-t CI、配對 CI 與 loss CI 在 `summary.csv`／`rounds.csv`。

| Workload | 條件 | Final validation acc | Final validation loss | Final test acc | AUC₁₂ | Paired endpoint Δ vs E0 | Accuracy recovery |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| MNIST | E0 | 87.07% | 0.408 | 88.59% | 10.1660 | — | 不適用 |
| MNIST | E1 | 87.68% | 0.397 | 89.15% | 9.7263 | +0.61 pp | 4/5 |
| MNIST | E2a | 87.67% | 0.397 | 89.16% | 9.7249 | +0.60 pp | 4/5 |
| MNIST | E2b | 80.13% | 0.565 | 81.00% | 8.8897 | −6.94 pp | 0/5 |
| CIFAR-10 | E0 | 59.28% | 1.150 | 58.47% | 6.5809 | — | 不適用 |
| CIFAR-10 | E1 | 59.36% | 1.150 | 58.44% | 6.5211 | +0.08 pp | 5/5 |
| CIFAR-10 | E2a | 59.38% | 1.150 | 58.44% | 6.5211 | +0.10 pp | 5/5 |
| CIFAR-10 | E2b | 58.88% | 1.159 | 57.84% | 6.4895 | −0.40 pp | 5/5 |

「Accuracy recovery」不是 topology repair success：它按老師定義，從修復後首次 Root accepted contribution 起，尋找**連續兩個 accepted rounds** 的 accuracy 均落在 E0 五 seed 對應輪次的 95% CI 內；未達標的完整 run 照樣保留。因而 CIFAR-10 E2b 雖顯示 5/5 accuracy recovery，仍永久缺少 A2 的 8,000 筆資料；MNIST E2b 的 0/5 也不表示 Root 拒絕了部分修復。表中少量 endpoint 差異不應單獨解讀為某種修復方法提升模型品質，須一併看 post-failure AUC、95% CI、資料覆蓋與協定證據。

在全部 30 個有故障的有效 run 中，分析可從 Root 接受的新直接參與者及 round event 找到首次修復後貢獻：MNIST 五 seed 各條件均在第 15 輪，CIFAR-10 均在第 23 輪。`failure→first accepted repaired contribution` 的五 seed 平均值如下；它是從保存的 UTC event timestamp 相減，尚未將完整時間分段和跨程序時鐘誤差整理成論文表格。

| Workload | E1 | E2a | E2b |
| --- | ---: | ---: | ---: |
| MNIST | 75.03 s | 74.92 s | 73.57 s |
| CIFAR-10 | 79.02 s | 78.97 s | 77.41 s |

本批的 protocol evidence 另保存在原始事件：`flTopology` instruction、`flTopologyReport`、Root `TOPOLOGY_ACCEPTANCE`、舊／新 subscription ID、`mlCorreId`、Root／Branch `ROUND_AGGREGATION` 的 selected／successful／failed IDs，以及可離線計數的 PyMTLF 訂閱操作。Slice 3 初步核對了 40/40 有效 run 的**最後一輪** Root 直接貢獻者符合所屬條件；本文不將此延伸為「已逐筆審定全部輪次的所有訂閱不變條件」。PyMTLF 可觀察操作數也不能稱為精確跨 NF wire-level HTTP 次數；候選 schema 沒有 `topologyVersion` 欄位。

## 5. 原始資料、後處理與仍待補的論文材料

- 原始 run：`5G_NWDAF_Infrastructure/runs/protocol-hierarchical/e0-e2b/<workload>/seed-<n>/<condition>/<runName>/`；有效 run 各有 `run.json`、`events.jsonl`、`observations/*.jsonl`、`final-root-model.tar.gz`。同一系列目錄另保留兩筆失敗啟動資料，但不在有效統計分母內。
- 本批完整分析：`.../e0-e2b/analysis/full-20260922/` 的 `summary.csv`（八個群組與 CI）、`rounds.csv`（逐 seed／五 seed 曲線）、`details.json`（時間線、操作與 Leaf 覆蓋）、`curves.svg`（四面板平均與 CI）。`analysis/mnist-20260922/` 是執行中途的 MNIST-only 快照，不是另一批資料。
- 可供交換的有效 run ZIP：workspace 根的 `e0-e2b-formal-runs-20260922.zip`，包含 40 個成功 run、40 個最終模型及完整分析；不包含兩筆失敗原始目錄或 MNIST-only 快照。這是匯出副本，不取代原始 run 目錄；生成資料切分與初始模型仍在本地 `.generated/`，不在該 ZIP 內。
- 論文後處理仍須把已保存的事件彙成五 seed 的修復分段時間、拓樸／訂閱轉換及操作數表；E2b 若要呈現逐類**模型表現**，可用保存的 final model 與官方 test 資料另做離線評估，不把 Leaf class coverage 誤稱為逐類 accuracy。現有 `curves.svg` 畫 accepted rounds 1 起；初始模型 round 0 evaluation 在原始 Root 事件中，若論文圖需要須另外加入。

本文是可追溯的撰稿材料，並非論文結論或完整 protocol conformance report。對外聲稱「拓樸局部修復成功」、「學習表現恢復」或精確延遲時，須分別引用相應的原始事件、離線判定規則及時鐘界線，不可只依最後一輪曲線或單一布林值推論。
