# Slice 4：論文對應文件、分析指標移除與 Release Tag

日期：2026-10-09

狀態：User Reviewed／Candidate Commits Created；3.1、3.2、3.3 已實作並通過 `make test`、文件連結檢查與 independent
review（含 targeted follow-up），使用者已 review 並要求 commit（第 10 節）。Push、remote clean clone、建立與推送 tag、
匿名 recursive clone 驗證尚未執行，各自仍需使用者指示

上層依據：[公開發布準備主計畫](../../Public%20Release%20Preparation%20Master%20Plan.md)與
[Slice 3 公開文件、Clean Checkout 與發布關卡](./Slice%203%20Public%20Documentation%20Clean%20Checkout%20and%20Release%20Gates.md)。

## 1. 背景與目標

論文（free5GC World Forum 2026 投稿）的參考文獻以 `5G_NWDAF_Infrastructure` 的 GitHub 網址加上 release tag
`free5gc-world-forum-2026` 引用本 prototype。該 tag 目前不存在。2026-10-08 與指導老師的會議對公開 source 提出下列要求：

1. 論文附錄的候選規格不必與實作完全相同，但差異要在 source repository 內說明；
2. 建立一個對應這篇論文的文件目錄，寫明開源 free5GC 缺少哪些 Release 18 功能、需要加入什麼才能執行本 prototype，
   以及論文沒有交代的操作步驟；
3. 整理與文件完成後，只在 `5G_NWDAF_Infrastructure` 打一個 tag，依靠 submodule 鎖定相依版本；讀者以該 tag 重現論文，
   之後的開發與論文無關；
4. Flat（一般 Release 18）流程的相容性與補齊附錄尚未實作的部分，排在 tag 之後。

另外，論文已不再使用「故障後 12 輪 accuracy 面積」指標（論文 repository commit `d03d17e`，已向老師回報），使用者於
2026-10-09 決定同步從離線分析工具移除該指標。

本 Slice 的目標是讓 tag 所指的 revision 同時滿足：分析工具的輸出與論文實際採用的指標一致；repository 內有一份公開讀者
可以從論文對照到 source、指令與已知差異的文件；tag 建立在經過驗證的 `main` revision 上。

## 2. 目前基線

- `5G_NWDAF_Infrastructure` 的 `origin/main` 與 `feat/hierarchical-fl-protocol-extension` 都指向
  `92b3e90beec38e8f53bc79b787d9926ec341f099`，即 Slice 3 Gate A–D 驗證過的 candidate。本地 checkout 位於 feature branch；
  本地 `main` 落後 `origin/main`，未更新。Repository 沒有任何 tag。
- 四個 submodule 的 gitlink 與 `components.lock.yaml` 一致：NRF `0dd4024`、NWDAF `3f30ccc`、ADRF `6e2d5a9`、PyMTLF `f8e6313`。
- Slice 1–3 已完成 legacy 清理、repository-local 文件重寫、clean-checkout 驗證與 fresh-provision GPU smoke。本 Slice
  不重做這些工作。
- Slice 3 記錄 ADRF 截至 2026-09-28 尚未公開。2026-10-09 使用者告知已公開，並以未認證的 GitHub API 查詢確認
  `5G_NWDAF_Infrastructure`、`NWDAF`、`PyMTLF`、`nrf`、`adrf` 五個 repository 皆可匿名存取。完整匿名 recursive clone
  尚未執行。
- 上游 free5GC（2026-10-09 查詢）：`free5gc/free5gc` 的 `main` 有 16 個 submodule（amf、ausf、bsf、chf、n3iwf、nef、nrf、
  nssf、pcf、scp、smf、tngf、udm、udr、upf、webconsole），不含 NWDAF 與 ADRF；`free5gc` organization 的 62 個公開
  repository 中也沒有 `nwdaf` 或 `adrf`。
- `Intelligent-Systems-Lab/nrf` 是 `free5gc/nrf` 的公開 fork。Pinned commit `0dd4024` 的祖先 `8e567d4` 已在上游 `main`；
  其後兩筆 commit（`e5d6df3`、`0dd4024`）不在上游 `main`，共改動 13 個檔案（discovery 與 NF profile 處理）。上游 `main`
  在分歧點之後另有 17 筆 commit 未被本 fork 納入。
- 40 組正式 run 的 `run.json` 記錄的執行 revision 與 tag 將鎖定的 revision 不完全相同（2026-10-09 核對全部 40 組）：
  - NRF：`0dd4024`，與 pin 相同；
  - PyMTLF：`c31b8b6`，pin `f8e6313` 晚一筆，只改文件；
  - ADRF：`905f059`，pin `6e2d5a9` 晚一筆，只改文件；
  - NWDAF：`de8b385`，pin `3f30ccc` 晚兩筆，一筆只改文件，另一筆只改 `pkg/mockapp/app.go` 中一行 mock 產生路徑；
  - 四個 component 在全部 40 組都是 clean；
  - `5G_NWDAF_Infrastructure`：`a5f10f4`，其中 32 組記錄為 dirty、8 組為 clean。Tag 將位於 `92b3e90` 之後，中間包含
    `44aa06a` 與公開前清理 `92b3e90`（共 167 個檔案）。清理後的 revision 由 Slice 3 的兩輪 smoke 驗證，沒有重跑正式矩陣。
  - Dirty 的原因（使用者 2026-10-09 說明，並對照 commit 與 run 時間確認相符）：最先完成的 8 組（MNIST seed 1、2 的四個
    條件）使用事先建好的資料集，working tree 為 clean；series 自動接續後續 seed 時，暴露出資料集與 seed model 的準備
    問題，當場修正後繼續執行，其餘 32 組因此記錄為 dirty。該修正在全部 run 結束後 commit 為 `44aa06a`，共 6 行：
    `fl-series-run.py` 在每次 run 前呼叫 `make dataset-generate`；`seed-model.py` 把 seed model 目錄權限設為 `0755`，
    讓 Root 容器可以讀取。兩者都是實驗準備流程，不涉及訓練、聚合或 protocol 行為。全部 8 組 clean run 的開始時間都
    早於第一組 dirty run。`run.json` 只記錄 dirty flag，不記錄當時的 diff，因此「dirty 內容即 `44aa06a`」是依說明與
    時間順序判斷，不是逐位元證明。
- 故障後 AUC 指標在公開 source 的範圍只有兩個檔案：`scripts/host/fl_series_analysis.py` 與 `tests/fl-analysis.py`。
  `README.md`、`docs/` 與 Make targets 都沒有提到這個指標或其輸出欄位。
- Repository 內沒有任何文件說明本 source 與論文、論文附錄或上游 free5GC 的關係。

## 3. 工作範圍

### 3.1 移除故障後 AUC 指標

`scripts/host/fl_series_analysis.py` 的 `analyze()`：

- 移除每個 run 的 `auc` 計算（名義故障邊界後 12 個 accepted round 的 validation accuracy 總和）；
- 移除與同 seed E0 配對的 AUC 差值；
- `details.json`：每個 run 不再有 `postFaultAuc12`，`definitions` 不再有 `postFaultAuc`；
- `summary.csv`：不再有 `postFaultAuc12Mean／CiLower／CiUpper` 與 `pairedAucDeltaVsE0Mean／CiLower／CiUpper` 六個欄位；
- 迴圈變數 `boundary` 不再使用，改為只走訪 `BOUNDARIES` 的 key；`BOUNDARIES` 本身仍被圖形的故障邊界線、
  `nominalFaultAfterRound` 定義與 recovery 判讀使用，保留不變。

`tests/fl-analysis.py`：移除對 `pairedAucDeltaVsE0Mean` 的一行 assertion；同一個 test 對 paired endpoint 差值、issue
回報、事件計數與 Leaf coverage 的檢查保留。

這是離線分析輸出 contract 的縮減，不是 runtime 行為變更：不影響 scenario、runner、run schema、原始紀錄或訓練。保留的
欄位（endpoint、held-out、paired endpoint 差值、recovery、事件與 timing 細節）計算方式不變。

### 3.2 論文對應文件

在 `5G_NWDAF_Infrastructure` 新增文件目錄 `docs/world-forum-2026/`，英文撰寫，並在 `docs/README.md` 的索引與 root
`README.md` 各加入一個入口。四份文件的內容已由使用者於 2026-10-09 確認如下。檔名在實作時決定。

**文件一：`README.md`（論文與 source 的對照）**

- 論文標題、此 tag 的用途，以及四個 component 由 gitlink 與 `components.lock.yaml` 鎖定。
- 執行 revision 與 tag revision 的關係，只寫兩點：四個 component 在 tag 的 revision 與論文實驗相比只多了文件類的
  commit；testbed tooling 在實驗後為公開發布整理過，整理後以 smoke run 驗證，沒有重跑完整實驗。不提 dirty flag、
  不列實驗當時的 commit：raw runs 不公開，讀者無從對照；論文也已在 commit `43ec675` 以相同理由移除對應敘述。第 2 節
  的細節只留在本計畫作內部紀錄。
- 實驗條件對照：E0／E1／E2a／E2b 分別對應 MNIST 的 `formal-baseline`、`formal-replacement`、`formal-reparent`、
  `formal-partial-reparent`，以及 CIFAR-10 的 `all-class-skew-baseline`、`all-class-skew-replacement`、
  `all-class-skew-reparent`、`all-class-skew-partial-reparent`；`mnist/smoke.yaml` 不屬於論文實驗。上述對應已以各 scenario 檔的
  `experiment.condition` 核對。
- 論文的表與圖對應到 Make target 與分析輸出檔。
- 如實列出公開分析工具能直接產生與不能直接產生的論文數值（使用者確認照實寫入）。初步盤點如下，撰寫時逐項對照
  source 與輸出欄位後定稿：

  | 論文內容 | 公開工具的輸出 |
  | --- | --- |
  | 學習結果表：validation、held-out、loss 的五 seed 平均與區間 | `summary.csv` 直接提供 |
  | 相對 E0 的 validation endpoint 配對差 | `summary.csv` 直接提供 |
  | 相對 E0 的 held-out 配對差與區間 | 不提供；只有各條件自身的 held-out 平均與區間 |
  | Timing 表的分段（F–D、D–I、I–E、E–C） | 沒有現成欄位；`details.json` 提供每個 run 的事件時間線與故障到首次修復貢獻的總時間，分段須由時間線另行計算 |
  | 軌跡圖：相對 E0 的逐輪配對差 | `rounds.csv` 提供逐 seed、逐輪數值可供計算；工具輸出的 `curves.svg` 是絕對值曲線 |
  | E1 與 E2a 最終模型的相對 L2 差 | 不提供 |

- 不複製腳本：執行與分析腳本位於 `scripts/host/` 與 `experiments/`，由既有 Make targets 驅動，本文件以對照表指向它們。

**文件二：候選規格與實作的差異**

以論文附錄 A、B 為基準分四類陳述，每項附 component 檔案位置：

- 一致：新增成員的名稱、型別與 required 清單；feature 編號；16 層與 1,024 節點上限；receiver 與 reporter 身分檢查；
  merge patch 後重新驗證完整 schema。
- 有出入：CREATE 的 feature 協商與「無法執行」的處理（第 5.1 節）；三個狀態 enum 在兩個 wire model 中是可擴充字串；
  Go NWDAF 沒有獨立的 Patch 型別。
- 有規格但 prototype 未使用：沒有送出方以 PATCH 傳送 `flTopology`；`INACTIVE`、`OTHER`、`RESPONSE_TIMEOUT` 不會出現在
  上行報告；`statusCause` 沒有 consumer；retained-result 成員只驗證、不執行；root 只產生固定的三層結構。
- 實作有而附錄未列：額外的驗證規則、計時器預設值與容量限制。

以上清單來自唯讀比對，除第 5.1 節一項外尚未複核。寫入前逐項對照 source；無法對上的項目不寫，由 code path 推論而未
執行的項目標示為未驗證。

**文件三：相對於上游 free5GC 的需求**

- 本 testbed 只執行 NRF、NWDAF、ADRF 與 PyMTLF，不執行 AMF、SMF、UPF 等其他 5GC network function。
- 上游 free5GC 提供 NRF，但不提供 NWDAF 與 ADRF（第 2 節的查詢結果，文件中註明查詢日期）。
- NRF：使用 `free5gc/nrf` 的 fork；說明不在上游的兩筆 commit 各自改了什麼。Review 查證後確認只有第一筆（`e5d6df3`，
  NWDAF／ADRF profile 成員與 discovery filter）是 prototype 所需；第二筆（`0dd4024`）只影響 UDM internal group 的
  discovery，屬於 UE 資料蒐集路徑，本 testbed 不執行 UDM，文件標示為 prototype 未使用。
- NWDAF、ADRF、PyMTLF：各自提供的 Release 18 service 與在本 prototype 中的角色。
- MongoDB 等非 Git 相依項目，連到既有 `docs/components.md` 與 `provisioning.lock.yaml`，不重複內容。

**文件四：論文未涵蓋的操作與限制**

- 從 clone 到完成一次 series 與離線分析的步驟順序，每一步連到既有 `docs/installation.md`、`docs/configuration.md` 與
  `docs/operations.md`，不重複內容。
- 環境需求摘要（VM 數量、GPU 路徑），數值以既有文件與 source 為準。
- 補上既有文件缺少的三項（使用者 2026-10-09 確認放入；缺口已於同日對照 `README.md` 與 `docs/` 全部檔案確認）：
  1. 取得 source 的指令：clone、checkout 本 tag、初始化 submodule。既有文件沒有任何 clone、checkout 或 tag 的說明，
     都從 `git submodule update` 開始。
  2. 論文實驗使用的版本。既有文件只提到 Guest 為 Ubuntu 22.04 與 Python 的最低版本。只列有依據的項目，並標明來源：
     - source 中的 pin：Guest box `ubuntu/jammy64` `20241002.0.0`、Guest Go `1.26.2`、MongoDB `8.0` 系列、
       PyTorch `2.5.1+cu121`；
     - `run.json` 的紀錄：GPU driver `535.183.01`、容器內 CUDA `12.1`；
     - Host 套件資料庫（2026-10-09 查詢；dpkg log 自 2026-08-09 起沒有這些套件的安裝或升級紀錄）：VirtualBox `6.1.50`、
       Vagrant `2.4.3`、Docker `27.4.1`、Docker Compose `v2.32.1`、NVIDIA Container Toolkit `1.19.1`；Host 為
       Ubuntu 20.04；
     - Host 的 Go 與 `uv` 在實驗當時的版本沒有紀錄，不列。
     查詢過程沒有執行任何 `vagrant` 或 `VBoxManage` 指令。
  3. 解釋「approved Host context」。這個用語在 `README.md` 與四份 `docs/` 文件共出現八次，唯一的說明是 README 提到
     guard 會在 Host VirtualBox device namespace 不可用時拒絕。實際檢查是 `/dev/vboxdrv` 是否以 character device
     可見（`scripts/host/lib.sh`）。文件四以一句話說明：直接在安裝了 VirtualBox 的 Host 上執行，不要在容器或
     sandbox 內。
- 不放入：時間預估、需要網路下載的項目清單（使用者 2026-10-09 決定）。
- 不公開的內容：raw runs、final models、generated dataset 分割與 local config。
- 重跑數值不會與論文逐位相同的原因。

**Root `README.md` 的入口**

論文的參考文獻連到 repository 首頁，讀者最先看到的是 root `README.md`，而它目前沒有提到論文。在專案簡介之後、
`Repository layout` 之前新增一個短節（約三至四行）：

- 本 repository 是哪篇論文的 prototype（論文標題）；
- 重現論文時使用 `free5gc-world-forum-2026` tag；
- 連到 `docs/world-forum-2026/`。

既有的建置、smoke、series、artifact boundary 與檢查說明不修改，也不在 README 重複文件目錄的內容。README 提到的 tag
在 commit 與 push 之後才建立，中間會短暫指向尚不存在的 tag；第 3.4 節的順序是 push 後立即驗證並建立 tag，這段落差可
接受。

文件邊界：

- 論文 repository README 的 ToDo 要求「放一個子目錄收錄論文用的執行與分析腳本」；本 Slice 以文件一的對照表滿足這項
  需求，不另建腳本副本。
- 不手動維護 component SHA 清單；revision 的 authoritative source 仍是 gitlink 與 `components.lock.yaml`。
- 不寫入實驗結果數值、本機 run 路徑或開發進度；結果以論文為準。
- 「差異」文件只陳述可由 source 直接查證的事實。無法由 source 判定的項目標示為未驗證，不以推論填補。
- 目錄以論文發表場合命名。這是對外發表物的識別，不是內部 work-tracking 名稱；若日後計畫或 slice 改名，該目錄不需要
  跟著改。

### 3.3 `testbed-docs` 的對應紀錄

- 在[論文實驗 Slice 3 配對 Seed 與離線分析計畫](../../slices/Slice%203%20Paired%20Seeds%20and%20Offline%20Analysis%20Initial%20Plan.md)
  加註 `K=12` 故障後 AUC 已於 2026-10-09 自分析工具移除及其理由；原有決策與證據文字保留。
- [E0–E2b 正式實驗資料與環境整理](../../E0-E2b%20Formal%20Experiment%20Materials%20Draft.md)的 `AUC₁₂` 欄位是當時的
  分析結果紀錄，保留原值並加註該指標未被論文採用。
- 更新本目錄索引與主計畫的 Slice 清單。

### 3.4 Release tag

- 名稱：`free5gc-world-forum-2026`，與論文參考文獻一致。
- 對象：只有 `5G_NWDAF_Infrastructure`；四個 component 由 gitlink 鎖定，不另外打 tag。
- 目標 revision：包含 3.1 與 3.2 變更、已推送到 `origin/main` 並通過第 6 節驗證的 commit。
- 形式：annotated tag，訊息說明對應的論文。
- 順序：commit → push `main` → 從 remote 的 clean clone 驗證 → 建立並推送 tag → 確認 remote 可由 tag 解析到同一
  commit 與四個 exact submodule。Commit、push、建立 tag、推送 tag 各自需要使用者的明確指示。

## 4. Slice 責任對照

| 項目 | 本 Slice 的情況 |
| --- | --- |
| Operator-visible behavior | `fl-series-analysis` 的 `summary.csv` 少六個欄位、`details.json` 少一個 run 欄位與一條定義；其餘指令與輸出不變 |
| Authoritative inputs 與 generated artifacts | 不變。分析仍只讀 finalized run records |
| VM、Guest process、Host container、network、storage、state owners | 不變，本 Slice 不接觸 |
| External／component-native／private contracts | Component 行為與 wire contract 不變；本 Slice 只以文件描述它們 |
| Start、status、logs、stop、reset、recovery、failure paths | 不變 |
| 驗證 | 見第 6 節；不需要 real-environment evidence，理由見第 6.2 節 |
| 延後 | 見第 5 節 |

本 Slice 不新增 deployment mode、topology、role、config version、lifecycle procedure 或 experiment type，因此不需要
baseline stage map。沒有新增或移除 hash-like mechanism 或 derived evidence。

## 5. 不包含與延後

明確不在本 Slice：

- 修改 Go NWDAF、PyMTLF、NRF、ADRF 的 source 或更新任何 submodule pin；
- 修改 scenario、runner、run schema、recovery 判讀或其他分析公式；
- 其他程式整理或重構（Slice 2、3 已完成公開前清理；再動會使其驗證失效）；
- 重跑正式實驗，或搬動、刪除 `runs/` 等本地 evidence；
- 修改論文。差異比對的結果與使用者的決定見第 5.1 節。

依老師指示延後到 tag 之後（`future-phase handoff`）：

- Flat（一般 Release 18）流程與 hierarchical 流程並存、不破壞上游 free5GC 行為；
- 讓實作補上論文附錄已規定但尚未實作的部分；
- 接收方支援 `HierarchicalFLOrch` 但無法執行該次 CREATE 的合約時，改以 HTTP 403 回應（見第 5.1 節）。

外部狀態：使用者告知 ADRF repository 已公開（見第 8 節）；匿名 recursive clone 驗證在 tag 推送後一併執行。

### 5.1 紀錄：CREATE 的 feature 協商與「無法執行」的處理

差異比對發現 prototype 在 CREATE 時對兩種不同情況使用同一種回應。以下為 2026-10-09 的查證與決定，供 3.2 的「差異」
文件與 tag 之後的工作使用。

Prototype 的行為（PyMTLF `f8e6313`，`core/fl_client.py` 的 `create()` 與 `_can_accept_protocol_resource()`；送出方在
`core/fl_server.py`）：

- 收到帶有 `flTopology` 的 CREATE 時，接收方若不接受，回 201，回傳的 `suppFeats` 不含 `HierarchicalFLOrch`，資源停在
  閒置狀態；送出方檢查回傳的 `suppFeats`，刪除該資源並記錄 `FEATURE_NOT_SUPPORTED`。
- 「不接受」的條件除了能力本身，還包含只針對該次請求的條件：`mLPreFlag` 不為 true、帶有 `mLModelInfos`、NRF 登記的
  FL capability 與角色不符、本地 dataset 或 interoperability ID 不符、Leaf 缺少 `strategy`／`reportAfter` 或單位不是
  `epoch`、Intermediate 無法解析出可執行的 contract。
- 對未協商該 feature 的既有資源送出帶有這些成員的 PUT／PATCH，回 403 `ML_MODEL_TRAINING_REQS_NOT_MET`。

對照 Release 18 規格：

- TS 29.500 第 6.6.2 節：接收方比對雙方支援的 feature，在確認資源建立的回應中回傳兩者的交集；未知的屬性與值由接收方
  忽略。因此「不支援該 feature 時回 201 並回傳交集」符合規格，送出方檢查回傳值並清理的路徑也是必要的，因為未實作
  本 extension 的 Release 18 NWDAF 會有相同表現。
- TS 29.520 第 5.5.7 節已定義 `ML_MODEL_TRAINING_REQS_NOT_MET`（403 Forbidden，表示 ML model training requirements
  未被滿足，並以 `invalidParams` 指出不符合的屬性）。因此「支援該 feature 但該次請求的條件不符」較貼近規格的做法是
  403，而不是回報不支援。Prototype 在這一點把兩種情況合併處理，送出方無法區分原因，且多一次建立後刪除的往返。

這不是功能缺陷：40 組正式 run 都在此行為下完成，正常流程與實驗結果不受影響。

論文的對應文字只有附錄 B 限制條件段落的兩句：

- 「The members are accepted only on a resource that negotiated … feature 3 …; otherwise the receiver returns HTTP 403」：
  理解為對既有資源的後續操作時與 prototype 一致；若理解為涵蓋 CREATE 則不一致。屬於語意不夠明確，不是錯誤。
- 「A receiver rejects contracts it cannot execute」：沒有指定操作或狀態碼；prototype 以不接受 feature 的方式達到相同效果。

使用者決定（2026-10-09）：論文不因此修改；實作的調整排在 tag 之後；本節作為紀錄，並在 3.2 的「差異」文件如實說明
CREATE 時 prototype 以不接受 feature 的方式拒絕。

## 6. 驗證

### 6.1 驗證項目

| 主張 | 直接證據 | 執行條件 |
| --- | --- | --- |
| 移除指標後分析工具仍正確執行 | `tests/fl-analysis.py` 通過 | Host-only，synthetic 輸入 |
| 保留的輸出沒有改變 | 以本機 40 組正式 run 重跑 series 分析到暫存目錄，與既有分析輸出比對：`summary.csv` 共同欄位、`rounds.csv`、`details.json` 的 issues 與每個 run 的其餘欄位相同 | Host-only，只讀 `runs/`，輸出寫在 repository 外 |
| Repository suite 未受影響 | `make test` 通過 | Host-only／synthetic；使用者已同意執行（見第 8 節） |
| 新文件與 root `README.md` 新增段落的指令、路徑、scenario 與連結存在 | 逐項對照 source、`make help-all` 與 Markdown 連結檢查 | Host-only |
| 「差異」文件的每項陳述可追溯到 source | 每項附 component 檔案位置，並由 independent review 抽查 | 唯讀 |
| Tag 可從 remote 取得且指向預期 revision | Push 後從 canonical remote clean clone，核對 tag、`main`、gitlinks 與 `components.lock.yaml` | 需要網路；在 tag 推送後執行 |

### 6.2 不重跑 real-environment smoke 的理由

Slice 3 Gate C 的 fresh-provision smoke 驗證的是 provisioning、Guest／Host process lifecycle 與最小訓練流程。本 Slice 只修改
離線分析腳本、其測試與文件，沒有任何檔案位於 Vagrant、provisioning、config rendering、service lifecycle 或 runner 的
執行路徑上，因此 Gate C 的結果對新 revision 仍然適用。若實作過程中必須修改上述路徑的任何檔案，這個前提不再成立，
須停止並回到決策。本 Slice 不執行任何 real provider operation。

### 6.3 Review

程式與測試變更（3.1）完成 focused verification 後，啟動 independent subagent review。3.2 的文件雖是 prose，但「差異」與
「上游需求」兩份包含對 component 行為的事實陳述，一併交給 reviewer 對照 source 抽查。

## 7. Acceptance 與停止點

本 Slice 在下列條件滿足後進入 `Ready for User Review`，變更保持 unstaged／uncommitted：

1. 3.1 的變更完成，6.1 前三項驗證通過；
2. 3.2 的四份文件與 root `README.md` 的入口完成，指令、路徑與連結檢查通過，差異文件的陳述皆有 source 依據或明確標示未驗證；
3. 3.3 的加註完成；
4. Independent review 的 admitted findings 已關閉。

使用者要求 commit 後，才分別在兩個 repository 建立 commit。Push、建立 tag 與推送 tag 是後續獨立關卡。Tag 推送並完成
remote clean-clone 驗證後，本 Slice 才可標示 `Completed`。該次驗證以未認證方式執行 recursive clone，同時補上主計畫尚缺的
匿名 recursive clone evidence。

## 8. 決策

已確認：

- 從離線分析工具移除故障後 AUC 指標（使用者，2026-10-09）；
- 會改動既有邏輯的工作排在 tag 之後（使用者，2026-10-09，依老師 2026-10-08 的指示）；
- 只在 `5G_NWDAF_Infrastructure` 打一個 tag，名稱為 `free5gc-world-forum-2026`（老師，2026-10-08）；
- 文件目錄使用 `docs/world-forum-2026/`（使用者，2026-10-09）；
- ADRF 已公開（使用者告知，2026-10-09；同日以未認證 API 查詢確認可匿名存取，recursive clone 尚未執行）；
- 同意執行 `make test`（使用者，2026-10-09）；
- 保留計畫建立前已存在的 3.1 變更，作為本 Slice 的實作（使用者，2026-10-09）；
- 3.2 四份文件的範圍與內容，包含在文件一照實列出公開工具無法直接產生的論文數值（使用者，2026-10-09）；
- 上游 free5GC 的公開版本沒有 NWDAF 與 ADRF、只有 NRF（使用者告知，2026-10-09；同日查詢確認，見第 2 節）；
- 論文不因第 5.1 節的發現而修改，實作調整排在 tag 之後（使用者，2026-10-09）；
- 文件一以兩句話說明執行 revision 與 tag revision 的關係，不提 dirty flag（使用者，2026-10-09；範圍見 3.2 文件一）；
- Root `README.md` 新增一個短節，說明對應的論文、tag 與文件目錄的連結，既有內容不修改（使用者，2026-10-09）；
- 文件四補上取得 source 的指令、論文實驗使用的版本與「approved Host context」的解釋；不放時間預估與下載清單
  （使用者，2026-10-09）。

採用建議、使用者未提出異議的事項（執行相關 Git 操作時仍各自需要明確指示）：

1. **Branch**：本地 checkout 在 feature branch，而 remote 的 feature branch 與 `main` 相同。沿用 Slice 3 的做法，在目前
   branch commit 後以 fast-forward 同時推送到兩者。
2. **Tag 前的 clean-clone 範圍**：只重跑 exact submodule／lock 對照與 `make test`，不重跑 component build 與 dataset
   下載，因為相關輸入沒有改變。

尚未取得的授權：開始實作 3.2 與 3.3、commit、push、建立與推送 tag。

## 9. Initial conformance map

| 主張 | Owner／evidence | 狀態 |
| --- | --- | --- |
| 分析輸出不再包含故障後 AUC | `fl_series_analysis.py` diff、focused test | 完成；focused test 通過，independent review 無 finding |
| 保留的分析輸出不變 | 40 組正式 run 的新舊輸出比對 | 通過（第 10 節）；diff 若再變更須重做 |
| Repository suite 通過 | `make test` | 通過（`REPOSITORY_TEST status=passed`，2026-10-09） |
| 論文對應文件存在且與 source 一致 | `docs/world-forum-2026/`、root `README.md`、`docs/README.md`；連結與引用路徑檢查 | 完成；六個檔案的相對連結與引用路徑全部存在 |
| 候選規格與實作的差異有 source 依據 | 附錄與 pinned component source 的逐項比對 | 完成；寫入的每一項由實作者對照 source，並經 independent review 逐項查核 |
| `testbed-docs` 對應紀錄已加註 | 3.3 所列文件 | 完成 |
| Tag 建立於已驗證的 `main` revision | commit、push、remote clean clone、tag | 候選 commit 已建立於 feature branch；push 之後的步驟待授權 |

## 10. 進度

- 2026-10-09：建立本計畫草案。
- 2026-10-09：在本計畫建立之前，3.1 的兩個檔案已被修改（未 stage、未 commit）。當時已執行的檢查：
  `tests/fl-analysis.py` 通過；以 40 組正式 run 重跑 series 分析到 repository 外的暫存目錄，與
  `runs/protocol-hierarchical/e0-e2b/analysis/full-20260922/` 比對，`summary.csv` 只少第 3.1 節所列六個欄位、其餘欄位
  逐值相同，`rounds.csv` 位元組相同，`details.json` 只少 `postFaultAuc` 定義與各 run 的 `postFaultAuc12`，
  `curves.svg` 僅 matplotlib 產生的元素 ID 不同。使用者其後確認保留該變更，上述結果計入本 Slice 的 evidence。
- 2026-10-09：一個唯讀 subagent 完成論文附錄 A、B 與 Go NWDAF `3f30ccc`、PyMTLF `f8e6313` 的逐項比對，沒有執行任何程式
  或測試。結論摘要：附錄 B 的 property 名稱、型別、required 清單與兩個 component 的 wire model 一致；application-level
  constraint 有出入，其中影響最大的是「未協商 feature 時回 HTTP 403」對 CREATE 不成立（PyMTLF 回 201 並把回傳的
  `suppFeats` 清空，由送出方刪除該資源）。這一項已由實作者對照 `fl_client.py` 的 `create()` 與
  `test_candidate_create_returns_lossless_persistent_contract_without_feature` 抽查確認；其餘項目尚未複核，其中數項被
  subagent 自己標為「由 code path 推論、未執行」。完整清單作為 3.2「差異」文件的輸入，寫入前逐項對照 source。
- 2026-10-09：使用者確認第 8 節所列決策。同日以未認證的 GitHub API 與 `free5gc/free5gc` 的 `.gitmodules` 查證上游
  free5GC 的組成、NRF fork 與上游的關係，以及五個 repository 的匿名可存取性，結果記於第 2 節。
- 2026-10-09：確認既有建置文件的三項缺口並記入文件四的大綱；核對 40 組正式 run 記錄的 revision 與 pin 的差異，
  結果記於第 2 節。唯讀比對（附錄與實作）是在 pinned revision 上進行；由於 PyMTLF 與 NWDAF 在執行 revision 與 pin
  之間沒有 wire model 或行為的變更，其結論同樣適用於執行 revision。
- 2026-10-09：使用者指示開始實作。
  - `make test` 通過。執行前確認 suite 內的 provider 呼叫全部以 mock 取代，未執行任何 real provider operation。
  - 新增 `docs/world-forum-2026/` 的 `README.md`、`specification-differences.md`、`free5gc-requirements.md`、
    `reproduction-notes.md`；root `README.md` 新增 `Paper` 一節；`docs/README.md` 新增索引項目。差異文件只寫入實作者
    已對照 source 的項目，唯讀比對中屬於推論的項目（PyMTLF 可能回 500、`statusCause` 可能帶自由文字）未寫入。
  - 3.3 的兩份文件已加註。
  - Independent review：程式變更無 finding，並確認 diff 未觸及 Vagrant、provisioning、rendering、lifecycle 或 runner
    路徑。文件有六項 admitted finding，均為文字修正：(1) NRF 第二筆 commit 被寫成 prototype 所需，實際只影響 UDM
    group discovery；(2)「CREATE 不回 403」過於絕對，true `retainedResultReq` 的 CREATE 仍回 403；(3)「無法解讀
    extension」沒有 source 依據，PyMTLF 一律先解析並驗證成員；(4) 不接受 feature 的條件清單不完整；(5)
    `recoveryRound` 只在判定為 recovered 時輸出；(6) 差異文件部分項目缺 source 位置與 identity 檢查，重現說明缺環境
    摘要。六項修正後交回同一 reviewer 做 targeted follow-up，全部關閉；follow-up 另指出一處措辭（論文未指明 403），
    已修正。
  - 採納的非必要建議：落差表移除論文未單獨列出的 validation endpoint 配對差一列並併入軌跡圖一列；新增
    protocol-success 判定一列；PATCH 成員補上 preparation flag；註明 NWDAF 的 Go module path。
  - 未採納：`docs/README.md` 結尾段落與新索引項目之間的輕微語意張力；自由文字 `statusCause` 路徑因未經確認而不寫入。
- 2026-10-09：使用者 review 四份文件的摘要後確認內容，並同意 commit 的切分方式。`5G_NWDAF_Infrastructure` 在
  `feat/hierarchical-fl-protocol-extension` 上建立兩筆 commit：`18c9fa77daa5b46aa6775678ea3f4a50d8c2ae0f`（移除故障後 AUC）與
  `fd3d95392564e6ae70a821bfb5b2e0a98da3ee0a`（論文對應文件與兩個 README 入口）。後者是目前的 tag 候選 revision。
- 尚未執行：push、remote clean clone、建立與推送 tag、匿名 recursive clone 驗證。
