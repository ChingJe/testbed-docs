# Hierarchical FL 實驗環境公開發布準備主計畫

日期：2026-09-27

狀態：Slice 3 Visibility Partially Complete／ADRF Pending；Slice 2 的正式 component tracking、protocol-only cleanup 與
Slice 3 公開文件已建立 candidate commits。Gate B/C 發現的 GPU config checker 與 runner／FL-control stale
interfaces 均已完成最小修正。Final exact candidate `92b3e90beec38e8f53bc79b787d9926ec341f099` 的
clean-checkout verification 已通過，fresh-provision GPU MNIST smoke 也已完成 2 個 accepted rounds、final
collection、held-out evaluation 與 scoped reset。Remote feature branch 與 `main` 已 fast-forward 到
candidate，canonical remote clean clone 亦已通過。Testbed、NWDAF、PyMTLF 與 NRF 已公開；ADRF 尚未公開，因此完整匿名
recursive clone 驗證尚未完成

## 1. 背景與目標

本次 E0、E1、E2a、E2b 正式實驗已完成 2 個 workload × 4 個 condition × 5 個 paired seeds，共 40 個有效 run，
並已保存實驗材料與環境摘要。`NWDAF`、`NRF`、`PyMTLF`、`ADRF` 的相關實作已在另一個 workspace 完成 merge
與 push。目前 NWDAF、NRF 與 PyMTLF 已公開，ADRF 尚待 repository owner 處理 visibility。

本計畫的目標是在公開前完成下列工作：

1. 將 testbed 使用的 component tracking 改到已合併、預定公開的正式來源；
2. 保留重現本次 E0–E2b 實驗所需的 source、scenario、runner、分析工具與文件；
3. 移除或明確歸檔不再支援、重複、過時或只服務舊流程的內容；
4. 讓 repository 內的 README、操作文件、實驗說明與實際 source 一致；
5. 從乾淨 checkout 證明公開版本可取得依賴、產生設定並走到適當的啟動前邊界；
6. 完成公開內容與 Git history 的暴露風險檢查後，才將相關 repository 改為 public。

公開不是本計畫一開始的動作。component 與 testbed repository 應維持 private，直到 tracking、清理、文件與乾淨建置
驗證皆通過 review；若發現已提交的敏感資料，須先撤銷或輪替受影響的 credential，再決定是否需要重寫 history。

## 2. 範圍與 repository 邊界

### 2.1 主要範圍

- `5G_NWDAF_Infrastructure`：component source tracking、build context、deployment／experiment tooling、scenario、
  repository-local 文件與測試，以及公開版本的乾淨建置能力。
- `testbed-docs`：只保存本次內部計畫與決策紀錄，不是本輪公開候選，也不納入內容清理、license 或 history exposure
  acceptance。
- `NWDAF`、`NRF`、`PyMTLF`、`ADRF`：本輪先視為已完成 merge／push 的預定公開 upstream；需核對公開 remote、branch、
  revision 與 testbed consumer，但不因本計畫自動取得修改 component behavior 或重寫其 history 的權限。

若盤點發現 testbed 還直接依賴其他 repository、registry、package source 或 private asset，應先回填 owner、用途與公開策略，
不能靜默保留私人依賴，也不能未經決策便將其納入清理。

### 2.2 不在初始範圍

- 不重新設計 hierarchical FL protocol、模型或 E0–E2b 實驗語意；
- 不因公開整理而重跑 40 組正式訓練；
- 不公開 raw runs、final models、生成資料集、初始模型 cache、local config、VM image、container volume 或交換 ZIP；
  論文只使用整理後的表格、圖形與敘述呈現實驗結果；
- 不為公開準備新增第二套 build、deployment、experiment 或 validation pipeline；
- 不把舊 `5G_Infrastructure` repository 的全面整理混入本輪；若實際公開內容仍引用它，另列依賴與處置決策；
- 不整理或公開 `testbed-docs`／`nwdaf-docs`；公開使用者所需說明應由各 source repository 自身文件承擔；
- 不因檔名老舊或搜尋到特定字詞就直接刪除；每個 material removal 必須先確認 owner、consumer 與替代路徑。

## 3. 已確認原則

1. **先整理，後公開**：第一次公開應以完成清理與驗證的 revision 為準，不先公開再補救目前已知的不一致。
2. **正式實驗是保留基線**：E0–E2b 的正式 scenario、paired-seed preparation、共用 runner、fault lifecycle、原始紀錄格式、
   離線分析與必要操作文件不得因清理舊內容而失效。
3. **tracking 以實際 consumer 為準**：不能只改 submodule pointer 或 README；須盤點 build、Compose、Vagrant、script、config、
   nested repository、image source、CI 與文件中的實際追蹤點，再決定唯一公開來源。
4. **source 與本地 artifact 分離**：公開 source 應說明如何產生或取得 runtime input；本機的生成物與正式 run evidence
   另行保存，不以提交大型 artifact 取代可重建流程。
5. **刪除前建立 disposition**：每個候選項先分類為保留、適配、歸檔、移除或需要決策；只有確認不再有 supported consumer、
   不承載唯一證據且替代路徑成立後才移除。
6. **不為清理增加多餘驗證**：驗證只保護公開版本實際支援的 build、config、lifecycle 與實驗入口；一次性 inventory、
   history inspection 與移除確認保留在 plan／review evidence，不新增永久 meta-test。
7. **repository 分開 review 與 commit**：tracking、source cleanup、文件與 component repository 各依 owning repository
   分別檢查、提交及推送。

### 3.1 已確認的公開內容決策

- Testbed 將 NRF、PyMTLF、ADRF pin 到已確認包含正式實驗 revision 的 default-branch exact heads；NWDAF pin 到
  `origin/master` 的 `3f30ccc25ad916df37660feacc8f33a743d83df3`。該 commit 直接接在原正式基準 `e2beba9...` 後，唯一
  source 差異是讓 `go:generate` 從 `PATH` 解析 `mockgen`。`.gitmodules` 記錄 `master`／`main`，gitlink 與
  `components.lock.yaml` 保存 exact commit。
- 移除 full 5GC、UE／UPF、gtp5g、PyAnLF、subscription／PseudoDriver、static FL 與過時 CIFAR-10 flows；共用
  renderer、checker、provider safety、lifecycle、reset、runner、artifact collection 與 analysis pipeline 保留並縮減 legacy branches。
- WebConsole 不列入公開 testbed，連同其 submodule、config、build、lifecycle、tests 與 repository-local 文件入口移除。
- 只保留 `experiments/protocol-hierarchical/mnist/smoke.yaml` 作最小 protocol smoke；正式 E0、E1、E2a、E2b scenarios
  全部保留，其他 condition-specific smoke 移除。
- `5G_NWDAF_Infrastructure` 採 Apache License 2.0，與四個 component 的 default heads 一致。
- 不重寫 component history；NWDAF current tree 的個人 `go:generate` path 改成可由 `PATH` 解析的 `mockgen`，NRF／ADRF
  舊 commits 的非敏感個人路徑接受保留。
- 公開 source 與重建方法，但不發布本次正式 raw runs、final models、failed runs、generated inputs、local configs 或交換 ZIP；
  本地實驗 evidence 在 source cleanup 期間保持原位。

## 4. 實作階段

### Slice 1：公開基線、tracking 與保留範圍盤點

本階段只建立可信 inventory 與決策材料，不切換 branch、不更新 submodule、不改 remote、不刪除檔案。
[Slice 1 詳細計畫與第一輪盤點](./public-release-preparation/slices/Slice%201%20Public%20Baseline%20Tracking%20and%20Retention%20Inventory.md)
另行維護實際 evidence、已確認 disposition、Slice 2 exact change list 與後續發布關卡。

主要工作：

- 記錄各相關 repository 的目前 branch、remote、revision、dirty state，以及另一個 workspace 已 merge／push 的候選公開來源；
- 從 testbed 的實際 build、config generation、deployment、container image 與 experiment entrypoint 反向追蹤所有 component consumer；
- 區分 Git submodule／nested repository、一般 clone、build context、editable source、container image 與純文件引用；
- 建立 E0–E2b 重現所需資產清單，包含正式 scenario、dataset／seed-model preparation、runner、analysis、必要 config 與文件；
- 盤點 raw runs、生成資料、交換 ZIP 與其他本地 artifact 的位置、owner、備份需求及是否受 Git 追蹤；
- 建立 cleanup disposition 表，逐項記錄保留、適配、歸檔、移除或待決策的理由與 consumer；
- 檢查公開候選 tree 與 reachable history 是否存在 credential、private key、token、私人 remote、個人絕對路徑、
  非預期大型資料或不具公開授權的第三方內容；一般實驗室 topology 資訊不因「看似內部」自動判定為敏感。

Slice 1 的交付是經使用者確認的 inventory 與 Slice 2 implementation proposal，不授權實際刪除或公開。若候選公開
branch／revision、正式實驗保留範圍、history 處置或第三方授權仍不明確，必須停在此階段。

### Slice 2：切換正式 tracking 與最小清理

本階段依已核准的 inventory 執行 repository 變更。
[Slice 2 詳細計畫與 exact source disposition](./public-release-preparation/slices/Slice%202%20Formal%20Tracking%20and%20Minimal%20Cleanup.md)
記錄 upstream target evidence、file／consumer 級移除範圍、測試 disposition 與 verification boundary。

主要工作：

- 將 testbed 的 component references 切換至核准的正式 branch／revision／public remote 目標；
- 同步更新真正消費該 identity 的 build、container、config 與 developer workflow，不建立另一份平行追蹤真相；
- 依 disposition 移除已被替代且沒有 supported consumer 的舊 scenario、工具、workaround、重複設定與無意義測試；
- 只有發現 Git history 之外仍有唯一 provenance 或操作價值時才使用既有 archive 邊界；目前盤點的 legacy source 直接移除，
  不另建 archive tree；
- 核對 `.gitignore`、local config 邊界與 artifact 說明；只有發現實際暴露缺口才修改 ignore rules；
- 保持 E0–E2b 正式流程的 authoritative source、入口與 evidence contract 不變；若清理必須改變這些契約，先更新計畫並請使用者決策。

驗證以受影響的實際 consumer 為中心：tracking 能解析、必要 source 可取得、設定可生成、既有 checker 與 focused tests 通過。
不因刪除舊內容而為每個被刪檔案建立永久「不存在」測試。

Slice 2 tracking／source cleanup 與 Slice 3 repository-local 公開文件已一併建立 testbed candidate commit；Gate A user review
亦已完成，實際驗證、finding 與 remaining release gates 記錄在 Slice 3 詳細計畫。

### Slice 3：公開文件、乾淨 checkout 與發布關卡

本階段將公開使用者看到的說明與最終 source 對齊，並驗證首次公開候選 revision。

[Slice 3 詳細盤點與計畫](./public-release-preparation/slices/Slice%203%20Public%20Documentation%20Clean%20Checkout%20and%20Release%20Gates.md)
記錄逐檔文件 disposition、commit 前後的驗證分界、final exposure／license review、real smoke 建議與 default-branch／visibility
發布關卡。

主要工作：

- 更新 root README、repository map、prerequisites、component version／branch 策略、build、deployment、experiment 與 analysis 指引；
- 清除指向舊 feature branch、舊 component ownership、已移除入口、私人 workspace 路徑或過時 topology 的說明；
- 說明公開 source 不包含哪些本地 artifacts，以及使用者如何準備 dataset、seed model 與 runtime config；
- 將歷史／不維護內容從正式導覽移除，並讓 archive 明確不代表目前支援能力；
- 從新的乾淨 checkout 取得核准的 component revisions，完成 dependency／build 準備、config generation、靜態 checker
  與不啟動 real provider 的啟動前檢查；
- 若公開承諾包含 real testbed 可執行性，再依 provider safety boundary 執行一個最小代表性 smoke，而不是重跑正式矩陣；
- 對最終公開 tree 與將被公開的 reachable history 完成一次 exposure review，記錄已解決項目與仍需接受的公開內容。

完成上述 review 後，另提出逐 repository 的公開與 push proposal。將 Git hosting visibility 改為 public 是獨立的外部狀態變更，
仍須使用者最後明確批准；計畫或 implementation 完成不自動授權公開。

目前 Gate A 已完成：公開文件只導向 retained protocol flow，Host-only repository suite 與文件連結檢查通過，版本固定且由官方
checksum 驗證的 Gitleaks v8.30.1 已掃描 testbed 與四個 component 的 current public trees。命中項目只有 component-native
model artifact identity 與 MongoDB signing-key fingerprint，均已依實際 consumer 判定為非 credential。Initial review 另發現
`preflight.sh` 在 optional-service contract 移除後仍使用舊函式簽章；已改成目前兩參數 contract 並通過 focused check 與完整
`make test`。

首次 Gate B 從 `b5cc0af979be85e1a9d28b83557acacba0d98f5c` 建立 clean checkout，完成 exact
submodules／lock 對照、Python environments、repository suite、官方 MNIST dataset 產生／驗證、三個 Go builds、
PyMTLF image build，以及五個 repositories 的 Git/history scan。掃描命中已分類為公開 identity 或已從
current tree 移除的 lab／demo fixtures，沒有 live credential 需輪替或 history rewrite。GPU config validation 同時
承認 checker 誤要求 Branch 使用 CUDA；修正為 GPU policy 下對照每個 authoritative backend device，並以
CPU／GPU render checks 與完整 `make test` 驗證通過。修正已 amend 為
`ecf8030ed23b389edf3c0961327018e7318a611b`，並從新的 clean checkout 完整重跑 Gate B；
exact dependencies、suite、GPU config／dataset pre-runtime flow、component builds 與 final history scan 均通過。Fresh
VM smoke 又承認 runner 仍呼叫已縮減的 FL-control 舊介面；移除無 consumer 的 Controller argument，並改由
runner 已記錄的 preparation／round／delay deadlines 計算 closure budget。Final source amend 為
`92b3e90beec38e8f53bc79b787d9926ec341f099`；新 exact checkout 重跑受影響的 Gate B 通過，而
fresh-provision GPU smoke 也完成 2 個 accepted rounds、final model／raw observations 收集、held-out evaluation、
process stop 與 scoped reset。使用者核准 Gate D promotion／push 後，remote feature branch 與 `main` 均以
fast-forward 更新到同一 candidate；從 canonical remote 的 `main` 重新 clean clone 可取得四個 exact submodules，
URL、branch metadata、gitlinks、lock revisions 與 clean state 全部一致。

### Slice 4：論文對應文件、分析指標移除與 release tag

本階段在 Slice 3 驗證過的 candidate 之上，移除論文未採用的故障後 AUC 分析指標，新增對應論文的 repository-local 文件，
並在通過驗證的 `main` revision 建立論文引用的 release tag。它不修改 component、scenario、runner 或 runtime lifecycle。

[Slice 4 詳細計畫](./public-release-preparation/slices/Slice%204%20Paper%20Companion%20Documentation%20and%20Release%20Tag.md)
記錄範圍、驗證、已確認決策與各 Git／tag 關卡。計畫內容已由使用者確認，實作尚未開始。

## 5. 保留、歸檔與移除的判定方式

盤點不以「這次實驗有沒有直接呼叫」作為唯一標準。候選項應依下列問題判斷：

- 它是否是 E0–E2b 或目前正式 testbed flow 的 authoritative input／入口／consumer？
- 它是否仍由 README、Make target、script、config、CI 或其他 supported path 引用？
- 它是否保存唯一的 architecture decision、操作知識、failure evidence 或 migration provenance？
- 若移除，現有替代路徑是否已存在、可找到且語意相同？
- 它是否只是 generated、downloaded、cache、runtime state 或可重新產生的本地結果？
- 它是否包含不能公開、缺乏授權或會讓公開使用者誤以為仍受支援的內容？

正式 source 與入口若仍被支援則保留或適配；唯一歷史證據但不再支援者才考慮歸檔；可重建本地 artifact 應移出 Git 邊界；
沒有 consumer、沒有唯一證據且已有替代者的內容才進入移除清單。最終刪除清單須在執行前交由使用者確認。

## 6. 公開前 acceptance

公開候選版本至少須滿足：

1. 每個 component 的正式 remote、branch／tag 策略及 testbed 實際使用 revision 都有單一明確來源；
2. testbed 不依賴未提交修改、另一個 workspace、私人 feature branch、個人絕對路徑或未說明的本地 artifact；
3. E0–E2b 正式實驗所需的 source 與操作入口仍存在，且文件能從乾淨 checkout 導向正確流程；
4. 核准的 cleanup disposition 已執行，正式導覽不再暴露已移除或不維護流程；
5. 受影響的 build／config／checker／focused tests 通過，且驗證主張不超過實際執行邊界；
6. raw runs 與生成資料的保存位置、公開策略及還原方式已決定，不會因 repository cleanup 遺失唯一實驗證據；
7. 最終 tree 與將公開的 reachable history 沒有未處理的 credential／private material；第三方內容具備可接受的公開依據；
8. 各公開 source repository 的 README 與實際 source 入口、ownership、依賴及限制一致；
9. 每個 repository 已完成 user review、獨立 commit proposal 與 push approval；
10. 使用者另行確認 hosting visibility 變更後，才執行公開。

## 7. 後續發布關卡

Source promotion 與 remote clean-clone verification 已完成。Testbed、NWDAF、PyMTLF 與 NRF 已可公開存取；
實際公開前只剩 ADRF owner 完成 visibility change，再執行一次完整的匿名 recursive clone 驗證。

Hosting visibility 變更始終需要獨立明確批准；上述設計決策也不授權 commit、VM destruction、merge、push 或公開。
