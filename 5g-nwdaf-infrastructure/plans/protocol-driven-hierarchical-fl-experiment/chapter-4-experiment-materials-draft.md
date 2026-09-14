# 論文第四章實驗資料整理（草稿）

狀態：Draft。本文整理 2026-09-13 至 2026-09-14 已保存的單次實驗資料，供第四章撰稿；尚未作為正式結果紀錄歸檔，也不取代[正式比較主計畫](./formal-branch-replacement-comparison-plan.md)。所有時間如未另註，均為 UTC。

## 1. 實驗目的與可比較條件

在相同的 Root–Branch–Leaf 拓樸下，對每個資料集比較：

| 條件 | Area A primary Branch | Area A replacement Branch | 研究意義 |
| --- | --- | --- | --- |
| No-failure baseline | 全程正常 | 已部署並註冊，但不取代健康 primary | 正常學習曲線 |
| Branch failure + replacement | 訓練中段停止 | Root 透過既有 priority 機制選用，接回 Area A 原本的兩個 Leaves | 故障、暫時只由 B／C 貢獻、恢復的曲線 |

原正式版包含 MNIST 與 CIFAR-10 各一組配對。另有一組 CIFAR-10「每個 Leaf 都有十類、但各類數量不同」的診斷配對，與原五類／Leaf 正式版是不同資料切分版本。每個條件只執行一次，因此只能描述觀察到的差異，不能宣稱統計顯著；本設計也沒有「故障但不替換」組，不能單獨估計 replacement 相對於不替換的因果效益。

## 2. 實驗環境與拓樸

### 2.1 實體 Host 與虛擬化

| 項目 | 配置／觀測值 | 證據性質 |
| --- | --- | --- |
| Host 作業系統 | Ubuntu 20.04.6 LTS | 2026-09-14 本機唯讀 inventory；非 run 內快照 |
| CPU | Intel Core i9-12900K，24 logical CPUs | 2026-09-14 本機唯讀 inventory；非 run 內快照 |
| Host 系統記憶體 | 64 GiB | 2026-09-14 `lshw` 回報 64 GiB、`lsmem` 回報 64 G online；非 run 內快照 |
| Linux `MemTotal` | 65,679,944 kB，約 62.6 GiB（`free -h` 顯示 `62Gi`） | 2026-09-14 `/proc/meminfo`／`free`；不是當下的 `available` |
| GPU | NVIDIA GeForce RTX 3080，10,240 MiB VRAM；driver 535.183.01 | 各 run 的 `run.json.gpu`；GPU UUID 相同 |
| GPU runtime | PyTorch 2.5.1+cu121、CUDA 12.1；七個 GPU participants | run evidence 與 selected PyMTLF runtime；七個角色共用一張實體 GPU，非七張 GPU |
| VM provider／Guest | Vagrant + VirtualBox；`ubuntu/jammy64` box `20241002.0.0`，Guest Ubuntu 22.04 | `testbed.protocol-hierarchical.yaml` 與 Vagrantfile 的部署宣告 |

64 GiB 是 Host 的系統記憶體規格；`MemTotal` 是 Linux 回報的另一個數字，不是當下剩餘可用量。兩者差額不能歸因於另列的 2 GiB swap；目前沒有逐項查明差額的保留用途。GPU 的 10,240 MiB 也是裝置總容量，不是訓練期間的空閒量。已保存的 run preflight 曾記錄約 9,988 MiB free GPU memory，超過 8,192 MiB admission floor；free swap 為 0 MiB，屬 warning，表示當時沒有剩餘 swap，不表示沒有配置 swap。本文不把這些值寫成每輪的資源消耗或效能量測。

### 2.2 VM 與 process 配置

| VM | vCPU | RAM | 宣告磁碟容量 | Guest 主要服務 |
| --- | ---: | ---: | ---: | --- |
| `core` | 2 | 3,072 MiB | 24 GiB | MongoDB、NRF、ADRF、Root NWDAF |
| `path-a` | 2 | 2,048 MiB | 20 GiB | Area A primary／replacement Branch、A1／A2 Leaves |
| `path-b` | 2 | 2,048 MiB | 20 GiB | Area B Branch、B1／B2 Leaves |
| `path-c` | 2 | 2,048 MiB | 20 GiB | Area C Branch、C1／C2 Leaves |

四台 VM 合計宣告 8 vCPU、9,216 MiB RAM、84 GiB 虛擬磁碟容量；這些是 VM 配置值，8 vCPU 不表示獨占 8 個 Host 實體核心，磁碟容量也不是實際佔用。管理網段為 `192.168.56.0/24`，SBI 網段為 `192.168.57.0/24`，均為 private network。本拓樸不啟動 UPF、UE 或完整 5GC user-plane 流程。`webconsole` 是 optional service，不屬於訓練必要路徑。

每個 NWDAF Guest service 對應一個在 Host 上執行的 PyMTLF Docker container：1 Root、4 Branch（含 A replacement）、6 Leaves，共 11 組。Root container 限制為 1 vCPU／1,024 MiB，四個 Branch 各 0.75 vCPU／768 MiB，六個 Leaves 各 1 vCPU／2,048 MiB；這些是容器上限，不是保留量或實際使用量。Root 與六個 Leaves 指向 `cuda:0`；四個 Branch 使用 CPU。基礎部署來源：`5G_NWDAF_Infrastructure/testbed.protocol-hierarchical.yaml`。

### 2.3 聚合及替換流程

Root 接受三個 Area Branch 的更新；各 Branch 接受自己兩個 Leaves 的更新，Branch 先做一次聚合再回傳 Root 做二次聚合。Root／Branch 的方法設定為 `fedProx`，`proximal_mu: 0.01`，模型權重以樣本數加權聚合（`sampleWeighted`）。本輪沒有設定多次 Branch-local 聚合後才回傳 Root。Area A primary priority 為 100、replacement 為 50；正常時使用 primary，故障後 Root 依 priority 選到 replacement。Root 至少需要兩個 Branch，允許少一個 Branch 仍接受 round；Branch 側需其兩個 Leaves。`roundTimeoutSeconds` 與 `preparationTimeoutSeconds` 均設定為 300 秒。

## 3. 模型、訓練參數與資料配置

### 3.1 模型架構與訓練方法

兩個資料集使用相同的 PyMTLF `SmallCNN` 層次；差異只有第一層的輸入 channel 數與影像尺寸。下表依實際 forward 順序列出各層輸出，尺寸省略 batch 維度；最後的十個值是分類 logits，模型內沒有額外的 softmax 層。

| 順序 | 操作 | MNIST 輸出尺寸 | CIFAR-10 輸出尺寸 |
| ---: | --- | --- | --- |
| 輸入 | 已正規化影像 | `1 × 28 × 28` | `3 × 32 × 32` |
| 1 | `Conv2d(input_channels → 32, kernel=3, padding=1)` → `ReLU` | `32 × 28 × 28` | `32 × 32 × 32` |
| 2 | `MaxPool2d(2)` | `32 × 14 × 14` | `32 × 16 × 16` |
| 3 | `Conv2d(32 → 64, kernel=3, padding=1)` → `ReLU` | `64 × 14 × 14` | `64 × 16 × 16` |
| 4 | `AdaptiveAvgPool2d(1)` → `Flatten` | `64` | `64` |
| 5 | `Linear(64 → 10)` | `10` | `10` |

影像在送入模型前由 uint8 轉為 float32 並除以 255。Leaf 本地訓練使用 Adam optimizer、cross-entropy loss、shuffle；FedProx proximal term 使用本輪取得的參考全域模型。每輪 Root validation 也計算 cross-entropy loss 與 accuracy。MNIST seed model ID 為 `1001`，CIFAR-10 為 `1002`；同資料集的配對使用相同 seed artifact。架構、輸入尺寸與訓練實作分別見 `5G_NWDAF_Infrastructure/ML/PyMTLF/seed_models/image_classification/{mnist,cifar10}/model.py`、`ML/PyMTLF/src/py_mtlf/core/workloads.py` 與 `ML/PyMTLF/src/py_mtlf/core/`。

| 資料集／版本 | accepted rounds | 每 Leaf local epochs／round | Leaf batch size | learning rate | partition／training random seed | Root／Branch μ | 故障觸發點 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| MNIST 原正式版 | 24 | 4 | 16 | 0.001 | 42 | 0.01 | treatment 在第 12 個 accepted round 後 |
| CIFAR-10 原正式版 | 40 | 5 | 16 | 0.001 | 42 | 0.01 | treatment 在第 20 個 accepted round 後 |
| CIFAR-10 全類別不等量版 | 40 | 5 | 16 | 0.001 | 42 | 0.01 | treatment 在第 20 個 accepted round 後 |

每列的 baseline 不注入故障，treatment 才有 fault 與 observation 設定；後者以 250 ms poll 與 30 s heartbeat 觀測。Root validation 的 batch size 為 128，不等於 Leaf 訓練 batch size。上述設定依目前 scenario、generated config 及 run snapshot 核對；scenario 現行位置在 `5G_NWDAF_Infrastructure/experiments/protocol-hierarchical/{mnist,cifar10}/`。`run.json.scenario.definition` 保留執行當時搬移前的舊路徑，不應以目前檔案位置改寫歷史證據。

對應的現行 scenario 檔名為 MNIST 的 `formal-baseline.yaml`／`formal-replacement.yaml`，CIFAR-10 原正式版的同名兩檔，以及 CIFAR-10 全類別不等量版的 `all-class-skew-baseline.yaml`／`all-class-skew-replacement.yaml`；各檔均在前述資料集子目錄下。執行時的實際參數仍以各 run 的 `run.json.scenario` snapshot 為準。

### 3.2 官方資料集、切分與評估

| 資料集 | 官方 train | 六個 Leaf 訓練總數 | 固定 validation | 官方 test／final held-out | 未使用的官方 train |
| --- | ---: | ---: | ---: | ---: | ---: |
| MNIST | 60,000 | 48,000（每 Leaf 8,000） | 2,000（每類 200） | 10,000，全量 | 10,000 |
| CIFAR-10 | 50,000 | 48,000（每 Leaf 8,000） | 2,000（每類 200） | 10,000，全量 | 0 |

固定 validation 從官方 **train** 留出，與六個 Leaf 的訓練來源索引互斥；每個 accepted round 後，Root 完整評估同一份 2,000 筆。官方 **test** 不參與逐輪訓練／validation，只在 final model 完成後評估一次。原正式版與全類別不等量版的 CIFAR-10 使用相同 48,000 筆 train 來源集合、相同 2,000 筆 validation 及完整 test，差別是 train 樣本分到哪個 Leaf。兩版分別執行，跨版本 accuracy 差異只作探索性觀察，不視為資料切分的單一變因效果。

原正式版採五類／Leaf 的 label-skew。表內「0–4」或「5–9」表示每個列出的類別各 1,600 筆，其他類別為 0；不同 Leaf 的來源樣本互斥。

| Leaf | 原正式版持有類別 | 樣本數 |
| --- | --- | ---: |
| A1、A2、B1 | 0–4 | 各 8,000 |
| B2、C1、C2 | 5–9 | 各 8,000 |

全類別不等量版的每個 Leaf 都有十類。下表數字是**單一 Leaf 對該組每一類**的筆數；同 Area 的兩個 Leaves 配額相同，但持有不同來源樣本。

| Leaves | 類別 0–4：各類筆數 | 類別 5–9：各類筆數 | 每 Leaf 總數 |
| --- | ---: | ---: | ---: |
| A1、A2 | 1,300 | 300 | 8,000 |
| B1、B2 | 800 | 800 | 8,000 |
| C1、C2 | 300 | 1,300 | 8,000 |

同資料集／版本的 baseline 與 treatment 共用相同切分、partition seed 與 seed model；Area A replacement 仍服務 A1／A2，不改分配資料。全類別不等量配對的 dataset ID 是 `cifar10-all-class-skew`；原正式配對保留各自的歷史邏輯資料路徑，重複內容已共用實體儲存，run evidence 不變。

## 4. 保存的 run 與重現資訊

下表中的完整 run name 位於 `5G_NWDAF_Infrastructure/runs/protocol-hierarchical/`，依 `mnist/` 或 `cifar10/` 分類。每個有效 run 目錄都含 `run.json`、`events.jsonl` 與 `final-root-model.tar.gz`。run name 內的日期是命名標籤，不應代替 `run.json` 的 UTC timestamp；因此標籤為 `20260914` 的 CIFAR-10 run 可以在 UTC 的 2026-09-13 執行。原始 run 目錄屬本機實驗產物，不等同於已提交到 Git 的 verified record。

| 配對 | Baseline run | Treatment run | Testbed revision／dirty |
| --- | --- | --- | --- |
| MNIST 原正式 | `mnist-formal-baseline-20260913-a` | `mnist-formal-replacement-20260913-b` | `d2e8ba3`；baseline clean，treatment dirty（runner 時序修正） |
| CIFAR-10 原正式 | `cifar10-formal-baseline-20260914-a` | `cifar10-formal-replacement-20260914-a` | `d2e8ba3`；兩組 dirty（同一 runner 修正） |
| CIFAR-10 全類別不等量診斷 | `cifar10-skew-baseline-20260914-a` | `cifar10-skew-replacement-20260914-a` | `740e9c7`；兩組 clean |

六次有效 run 的 component revision 相同：NWDAF `be3fa57`、PyMTLF `bdbd2a9`、NRF `0dd4024`、ADRF `905f059`；完整 revision、dirty flag、GPU UUID／driver、seed artifact key、scenario snapshot 均以各自 `run.json` 為準。原 MNIST treatment 第一次嘗試 `mnist-formal-replacement-20260913-a` 因故障注入與事件寫入時序問題中止，**不計入有效配對**；失敗目錄已依先前確認清理，診斷摘要留於主計畫 Section 10。另有 `cifar10-mu01-baseline-20260914-a`，只用於 μ=0.1 探索性診斷，沒有對應 treatment，不列入三組配對結果。

## 5. 實驗結果摘要與故障觸發

### 5.1 最終官方 test 與逐輪階段

| 配對 | Baseline final test accuracy | Treatment final test accuracy | Treatment − Baseline | Treatment accepted phases | 第一次 replacement 貢獻 |
| --- | ---: | ---: | ---: | --- | --- |
| MNIST 原正式 | 8,884／10,000 = 88.84% | 8,899／10,000 = 88.99% | +0.15 百分點 | 12 normal、2 degraded、10 restored | 第 15 輪 |
| CIFAR-10 原正式 | 3,862／10,000 = 38.62% | 3,874／10,000 = 38.74% | +0.12 百分點 | 20 normal、2 degraded、18 restored | 第 23 輪 |
| CIFAR-10 全類別不等量診斷 | 5,806／10,000 = 58.06% | 5,812／10,000 = 58.12% | +0.06 百分點 | 20 normal、2 degraded、18 restored | 第 23 輪 |

三組 baseline 的所有 accepted rounds 都是 normal；六個有效 runs 的 `run.json` 均為 `status=successful`、`finalized=true`、無 runner failures。MNIST 每組 24 個 accepted outcomes、25 次 Root validation（含 round 0），CIFAR-10 每組 40 個 outcomes、41 次 validation；每次 validation 均為 2,000 筆。final test accuracy 是獨立的官方 10,000 筆 test 評估，**不是**逐輪 validation accuracy；final test loss 沒有保存，不得用最後一輪 validation loss 代替。

### 5.2 Treatment 故障與恢復時間線

| 配對 | 故障觸發／effective stop | 兩邊 hard-stop 完成 | 故障→Root 偵測 | 故障→replacement ready | 故障→首次 accepted 貢獻 | 故障後 accepted rounds |
| --- | --- | --- | ---: | ---: | ---: | --- |
| MNIST 原正式 | 第 12 輪後；2026-09-13 15:36:12.405374 UTC | 15:36:20.689112 UTC | 297.868 s | 298.735 s | 313.285 s | 第 13–14 輪 degraded；第 15 輪起 restored |
| CIFAR-10 原正式 | 第 20 輪後；2026-09-13 16:28:55.146406 UTC | 16:29:03.398877 UTC | 297.970 s | 298.840 s | 317.483 s | 第 21–22 輪 degraded；第 23 輪起 restored |
| CIFAR-10 全類別不等量診斷 | 第 20 輪後；2026-09-13 19:25:03.760955 UTC | 19:25:12.047060 UTC | 298.005 s | 298.859 s | 317.328 s | 第 21–22 輪 degraded；第 23 輪起 restored |

`effective stop` 是 Host 確認 Guest 收到 `SIGSTOP` 成功回覆的時間，不宣稱是 Guest kernel 真正送出訊號的精確時刻；`hard-stop` 才表示 Guest／對應 PyMTLF 均完成停止。Root 隨後在一個約 300 秒的 round timeout 後發現 primary 未回報，該輪只接受 Area B／C，並選擇次順位 replacement。表中的三個延遲皆相對於 `effective stop`，是單次觀測值，非系統保證；`ready` 也不等於已在 accepted round 貢獻。

### 5.3 曲線解讀與既有圖表

原正式 MNIST 兩組在故障前的逐輪 validation 完全相同；第 24 輪 baseline／treatment 都是 86.90%，loss 分別 0.412442／0.410491。原正式 CIFAR-10 第 20 輪兩組為 37.60%；第 21、22 輪 treatment 暫降至 36.00%、36.50%（baseline 37.50%、37.85%）；第 23 輪 treatment 為 42.55%（baseline 38.25%），第 40 輪為 40.35%／40.15%。全類別不等量 CIFAR-10 第 20 輪兩組為 54.10%；第 21、22 輪 treatment 為 51.30%、51.75%（baseline 54.30%、55.10%）；第 23 輪為 55.55%（baseline 55.50%），第 40 輪為 59.85%／59.70%。這些只是對已保存曲線的描述，不代表 treatment 在整體模型品質上更好。

三組離線分析各自保存在 `5G_NWDAF_Infrastructure/runs/protocol-hierarchical/analysis/{mnist-formal-comparison,cifar10-formal-comparison,cifar10-all-class-skew-comparison}/`；每組有 `rounds.csv`（含 round 0、每輪 loss／accuracy、成功 Branch IDs）、`summary.csv`（最終 test 與恢復延遲）及 `comparison.svg`（論文樣式曲線）。下節將 CSV 的核心數值直接轉錄為 Markdown；原始檔仍是逐輪 Branch identity 與更多精度的準據。

## 6. 逐輪 Root validation 原始數值轉錄

欄位中的 accuracy 單位為 %，loss 是固定 2,000 筆 validation 的平均 cross-entropy；`0` 為訓練前初始 validation，並非 accepted round。Phase 只描述 treatment；baseline 全程 normal。表格是現有 `rounds.csv` 的轉錄，不是 Leaf local training loss，也沒有把 final official test 當成額外一輪。若圖表或文字與表格不一致，回查各 run 的 `events.jsonl`／`run.json` 以及對應 `rounds.csv`。

### 6.1 MNIST 原正式配對

| 輪次 | Baseline acc (%) | Baseline loss | Treatment acc (%) | Treatment loss | Treatment phase |
| ---: | ---: | ---: | ---: | ---: | --- |
| 0 | 10.00 | 2.308266 | 10.00 | 2.308266 | initial |
| 1 | 28.65 | 2.023457 | 28.65 | 2.023457 | normal |
| 2 | 35.95 | 1.661642 | 35.95 | 1.661642 | normal |
| 3 | 43.60 | 1.419100 | 43.60 | 1.419100 | normal |
| 4 | 56.10 | 1.187199 | 56.10 | 1.187199 | normal |
| 5 | 66.55 | 1.009553 | 66.55 | 1.009553 | normal |
| 6 | 70.65 | 0.893413 | 70.65 | 0.893413 | normal |
| 7 | 73.20 | 0.815665 | 73.20 | 0.815665 | normal |
| 8 | 75.20 | 0.757800 | 75.20 | 0.757800 | normal |
| 9 | 76.45 | 0.713326 | 76.45 | 0.713326 | normal |
| 10 | 77.50 | 0.676718 | 77.50 | 0.676718 | normal |
| 11 | 78.65 | 0.646768 | 78.65 | 0.646768 | normal |
| 12 | 79.60 | 0.619588 | 79.60 | 0.619588 | normal |
| 13 | 80.90 | 0.596170 | 60.60 | 1.476310 | degraded |
| 14 | 81.65 | 0.572364 | 58.75 | 1.682654 | degraded |
| 15 | 82.15 | 0.549474 | 82.20 | 0.564679 | restored |
| 16 | 83.20 | 0.529142 | 83.60 | 0.523397 | restored |
| 17 | 84.25 | 0.510827 | 84.25 | 0.505987 | restored |
| 18 | 84.45 | 0.494692 | 84.75 | 0.490743 | restored |
| 19 | 84.85 | 0.478907 | 85.15 | 0.475670 | restored |
| 20 | 85.45 | 0.464341 | 85.45 | 0.460877 | restored |
| 21 | 85.85 | 0.450645 | 86.00 | 0.446528 | restored |
| 22 | 86.20 | 0.436245 | 86.35 | 0.433844 | restored |
| 23 | 86.40 | 0.423767 | 86.55 | 0.422159 | restored |
| 24 | 86.90 | 0.412442 | 86.90 | 0.410491 | restored |

### 6.2 CIFAR-10 原正式配對

| 輪次 | Baseline acc (%) | Baseline loss | Treatment acc (%) | Treatment loss | Treatment phase |
| ---: | ---: | ---: | ---: | ---: | --- |
| 0 | 9.95 | 2.303253 | 9.95 | 2.303253 | initial |
| 1 | 18.05 | 2.159814 | 18.05 | 2.159814 | normal |
| 2 | 23.45 | 1.985219 | 23.45 | 1.985219 | normal |
| 3 | 28.00 | 1.880583 | 28.00 | 1.880583 | normal |
| 4 | 30.75 | 1.822518 | 30.75 | 1.822518 | normal |
| 5 | 31.90 | 1.787715 | 31.90 | 1.787715 | normal |
| 6 | 32.55 | 1.768001 | 32.55 | 1.768001 | normal |
| 7 | 33.10 | 1.755557 | 33.10 | 1.755557 | normal |
| 8 | 33.05 | 1.745708 | 33.05 | 1.745708 | normal |
| 9 | 32.75 | 1.738136 | 32.75 | 1.738136 | normal |
| 10 | 32.90 | 1.729661 | 32.90 | 1.729661 | normal |
| 11 | 33.35 | 1.722591 | 33.35 | 1.722591 | normal |
| 12 | 34.00 | 1.716736 | 34.00 | 1.716736 | normal |
| 13 | 34.30 | 1.711135 | 34.30 | 1.711135 | normal |
| 14 | 34.70 | 1.706767 | 34.70 | 1.706767 | normal |
| 15 | 35.80 | 1.702307 | 35.80 | 1.702307 | normal |
| 16 | 36.35 | 1.699945 | 36.35 | 1.699945 | normal |
| 17 | 36.65 | 1.697979 | 36.65 | 1.697979 | normal |
| 18 | 37.30 | 1.697795 | 37.30 | 1.697795 | normal |
| 19 | 37.35 | 1.700292 | 37.35 | 1.700292 | normal |
| 20 | 37.60 | 1.703825 | 37.60 | 1.703825 | normal |
| 21 | 37.50 | 1.708391 | 36.00 | 2.379329 | degraded |
| 22 | 37.85 | 1.714341 | 36.50 | 2.535666 | degraded |
| 23 | 38.25 | 1.713246 | 42.55 | 1.573642 | restored |
| 24 | 38.45 | 1.716018 | 39.55 | 1.680067 | restored |
| 25 | 39.00 | 1.717339 | 39.15 | 1.697923 | restored |
| 26 | 38.85 | 1.717281 | 39.45 | 1.706526 | restored |
| 27 | 39.25 | 1.717268 | 39.50 | 1.710951 | restored |
| 28 | 39.75 | 1.722315 | 40.00 | 1.719765 | restored |
| 29 | 39.70 | 1.727788 | 40.30 | 1.722094 | restored |
| 30 | 39.70 | 1.729624 | 40.05 | 1.724777 | restored |
| 31 | 39.90 | 1.725357 | 40.25 | 1.725606 | restored |
| 32 | 40.15 | 1.727848 | 40.15 | 1.726440 | restored |
| 33 | 40.15 | 1.730802 | 39.85 | 1.735491 | restored |
| 34 | 40.15 | 1.730882 | 40.10 | 1.737372 | restored |
| 35 | 40.30 | 1.728381 | 40.30 | 1.732551 | restored |
| 36 | 40.10 | 1.730262 | 40.55 | 1.733390 | restored |
| 37 | 40.15 | 1.729976 | 40.35 | 1.729859 | restored |
| 38 | 39.80 | 1.741430 | 40.25 | 1.727040 | restored |
| 39 | 40.30 | 1.755242 | 40.25 | 1.732143 | restored |
| 40 | 40.15 | 1.756131 | 40.35 | 1.746720 | restored |

### 6.3 CIFAR-10 全類別不等量診斷配對

| 輪次 | Baseline acc (%) | Baseline loss | Treatment acc (%) | Treatment loss | Treatment phase |
| ---: | ---: | ---: | ---: | ---: | --- |
| 0 | 9.95 | 2.303253 | 9.95 | 2.303253 | initial |
| 1 | 26.65 | 1.993983 | 26.65 | 1.993983 | normal |
| 2 | 30.80 | 1.807056 | 30.80 | 1.807056 | normal |
| 3 | 34.10 | 1.730280 | 34.10 | 1.730280 | normal |
| 4 | 36.15 | 1.690028 | 36.15 | 1.690028 | normal |
| 5 | 37.95 | 1.659188 | 37.95 | 1.659188 | normal |
| 6 | 39.65 | 1.631098 | 39.65 | 1.631098 | normal |
| 7 | 41.05 | 1.603176 | 41.05 | 1.603176 | normal |
| 8 | 42.85 | 1.574776 | 42.85 | 1.574776 | normal |
| 9 | 44.20 | 1.545506 | 44.20 | 1.545506 | normal |
| 10 | 45.35 | 1.516603 | 45.35 | 1.516603 | normal |
| 11 | 46.35 | 1.488322 | 46.35 | 1.488322 | normal |
| 12 | 48.10 | 1.461587 | 48.10 | 1.461587 | normal |
| 13 | 49.85 | 1.436920 | 49.85 | 1.436920 | normal |
| 14 | 50.25 | 1.414992 | 50.25 | 1.414992 | normal |
| 15 | 51.00 | 1.395480 | 51.00 | 1.395480 | normal |
| 16 | 51.90 | 1.378409 | 51.90 | 1.378409 | normal |
| 17 | 52.20 | 1.363200 | 52.20 | 1.363200 | normal |
| 18 | 53.30 | 1.348822 | 53.30 | 1.348822 | normal |
| 19 | 53.70 | 1.335498 | 53.70 | 1.335498 | normal |
| 20 | 54.10 | 1.322685 | 54.10 | 1.322685 | normal |
| 21 | 54.30 | 1.310802 | 51.30 | 1.342097 | degraded |
| 22 | 55.10 | 1.299277 | 51.75 | 1.331093 | degraded |
| 23 | 55.50 | 1.288364 | 55.55 | 1.287004 | restored |
| 24 | 55.70 | 1.277773 | 55.80 | 1.276895 | restored |
| 25 | 55.95 | 1.267565 | 55.90 | 1.266750 | restored |
| 26 | 56.05 | 1.257672 | 56.00 | 1.256835 | restored |
| 27 | 56.05 | 1.248137 | 55.95 | 1.247291 | restored |
| 28 | 56.15 | 1.238914 | 56.40 | 1.238114 | restored |
| 29 | 56.35 | 1.230074 | 56.60 | 1.229142 | restored |
| 30 | 56.85 | 1.221334 | 57.00 | 1.220476 | restored |
| 31 | 57.15 | 1.213012 | 57.45 | 1.212075 | restored |
| 32 | 57.50 | 1.204938 | 57.75 | 1.203895 | restored |
| 33 | 57.75 | 1.196639 | 57.60 | 1.196156 | restored |
| 34 | 58.10 | 1.189243 | 58.10 | 1.188460 | restored |
| 35 | 58.45 | 1.181904 | 58.40 | 1.181165 | restored |
| 36 | 58.55 | 1.174826 | 58.80 | 1.174013 | restored |
| 37 | 58.80 | 1.167893 | 59.00 | 1.167027 | restored |
| 38 | 59.10 | 1.161170 | 59.30 | 1.160352 | restored |
| 39 | 59.65 | 1.154758 | 59.50 | 1.153852 | restored |
| 40 | 59.70 | 1.148617 | 59.85 | 1.147720 | restored |

## 7. 撰稿與審查時仍須注意

- 本文僅是原正式四條件與新增診斷配對的撰稿用 evidence index，並非已歸檔的 `records/`。
- 各條件只執行一次，沒有重複試驗的變異、信賴區間或顯著性檢定。圖中的短暫下降與後續回升可描述，不可推導為普遍的模型品質改善。
- CIFAR-10 五類／Leaf 與全類別不等量版是分別執行的實驗版本；跨版本 final accuracy 僅作探索性對照，不視為只改資料切分的受控比較。μ=0.1 診斷也不納入正式結論。
- 完整數據保存在本機 `runs/`，不是可由 Git 自動取得的論文附檔；提交論文或提供他人重現前，需另決定原始 run／dataset 的保存與提供方式。本文轉錄逐輪數字，但不取代完整事件與模型 artifact。
