# Slice 1：公開基線、Tracking 與保留範圍盤點

日期：2026-09-27

狀態：Inventory Complete／User Confirmed；component targets、cleanup disposition、license、history、artifact 發布策略與 Slice 2
exact change list 已確認，NWDAF portability commit 已由 component workspace 完成，testbed 尚未執行 tracking 切換或刪除

上層依據：[公開發布準備主計畫](../../Public%20Release%20Preparation%20Master%20Plan.md)。

## 1. 目標與邊界

本 Slice 在任何 tracking 切換或清理前，建立目前 repository、component source、正式實驗資產、本地 artifact 與公開風險的
可信 inventory，並將每個候選項分類為保留、適配、歸檔、移除或待決策。交付是供使用者核准 Slice 2 實際變更範圍的
decision package，不是公開 release 本身。

本階段允許 read-only source／Git inspection、經使用者核准的 `fetch` 與本計畫文件更新；不執行 `pull`、切 branch、改 remote、更新
submodule、stage／commit／push、刪檔、history rewrite、provider operation、VM／container lifecycle 或 hosting visibility
變更。另一個 workspace 已完成 merge／push 是使用者提供的前提；本輪已更新四個 component 的 remote refs，但沒有改變
任何 checkout。後續仍以 exact commit 與 ancestry 為依據，不以 branch 名或先前 cached `origin/*` 猜測公開基線。

## 2. 第一輪 repository inventory

以下是 2026-09-27 在本 workspace 的觀測；component 結果包含本輪經核准更新的 remote refs。

| Repository／path | Local branch | HEAD | Working tree／用途 | 初步結論 |
| --- | --- | --- | --- | --- |
| `5G_NWDAF_Infrastructure` | `feat/hierarchical-fl-protocol-extension` | `44aa06a1579aa07317e637d7e7c62c5f16f786c7` | clean；testbed owner | 仍在 feature branch；須確認公開 default branch 與新 pin |
| `NFs/nwdaf` | `feat/hierarchical-fl-protocol-extension` | `de8b385977eeed3c641564de08ff1fc6390d62d3` | clean；parent gitlink | `origin/master` 已包含此 revision；default head 為 `3f30ccc...` |
| `NFs/nrf` | `feat/r18-nwdaf-discovery` | `0dd4024d4ab75b6630e04901968228b9b9718cf5` | clean；parent gitlink | `origin/main` 與目前 pin 相同 |
| `ML/PyMTLF` | `feat/hierarchical-fl-protocol-extension` | `c31b8b6129d355f4ba65d927a1f97d7b04f95739` | clean；parent gitlink | `origin/main` 已包含此 revision；default head 為 `f8e6313...` |
| `NFs/adrf` | `feat/r18-federated-learning` | `905f0599f68fe389bba14ed56db0ef9abeab5ccd` | clean；parent gitlink | `origin/main` 已包含此 revision；default head 為 `6e2d5a9...` |
| `testbed-docs` | `main` | `9b114f4ab37e35d4292ddba12f9417316fe76d5a` | 僅有本次主計畫、導覽與 Slice 文件未提交 | 本輪文件 owner；不得和其他 repository 混合提交 |
| `nwdaf-docs` | `main` | `6c429da333ecee614f7ca7b472cee4be86a73eb0` | 有既存未提交文件變更 | 不屬於本次修改；盤點與後續操作須排除、保留該變更 |

四個 component 的 current `origin` URL 均指向 `Intelligent-Systems-Lab` 下相應 GitHub repository；parent testbed 的
`origin` 亦在同一 organization。URL 本身不證明 repository visibility；使用者已明確說明這些 repository 尚未公開。
2026-09-27 經核准執行 `git fetch --prune origin` 後，四個 remote default branches 都出現 force-update 記錄，因此本輪沒有
以 ref 名或 fast-forward 假設下結論，而是逐一執行 exact ancestry 檢查。結果均為 current experiment revision 是新 default
branch head 的 ancestor；所有 component checkout 與 working tree 保持不變。隨後已用 `git fetch --unshallow` 補齊四個
component histories，供本 Slice 的 history inventory 使用；Slice 3 仍須對最終公開候選執行專用 exposure scan。

## 3. Component tracking 實際契約

目前 component identity 不是單一 `.gitmodules` 欄位，而是三層共同作用：

1. parent Git index 的 gitlink commit 是 checkout 的 executable source lock；
2. `.gitmodules` 保存 clone URL 與 branch hint；
3. `components.lock.yaml` 重複 path、remote、branch／tag 與 commit，供可讀 inventory 與 runtime validation 使用。

`components.lock.yaml` 不是無 consumer 的說明檔：`config-render.py` 用它生成 image revision；`ml-compose-check.py` 核對
ML build args；`preflight.sh` 核對 submodule HEAD 與 dirty state；Guest Core／Path build scripts 及 provisioning check 也讀取它。
Docker image label 另保存 selected component revision。因此 Slice 2 若改 pin，至少須讓 gitlink、`.gitmodules` hint、lock metadata
與真正 build consumer 一致；不能只更新 README 或 branch 字串。

四個目標 component 的目前 tracking 如表：

| Component | `.gitmodules` branch hint | `components.lock.yaml` branch | Parent gitlink／lock commit |
| --- | --- | --- | --- |
| NRF | `feat/r18-nwdaf-discovery` | `feat/r18-nwdaf-discovery` | `0dd4024d...` |
| NWDAF | `feat/r18-hierarchical-federated-learning` | `feat/hierarchical-fl-protocol-extension` | `de8b3859...` |
| ADRF | `feat/r18-federated-learning` | `feat/r18-federated-learning` | `905f0599...` |
| PyMTLF | `feat/r18-hierarchical-federated-learning` | `feat/hierarchical-fl-protocol-extension` | `c31b8b61...` |

NWDAF 與 PyMTLF 的 branch hints 已和 readable lock 不同，雖然 exact commit 仍一致；這是公開整理要收斂的 metadata drift，
但不是證明 runtime 曾用錯 revision。正式新 pin 必須指向含有本次實驗 commit 的已合併公開 branch／tag。目前可供 Slice 2
實作的 base／exact targets 如表；使用者已核准這組方向，但尚未授權跳過 Slice 2 implementation review gate。

| Component | Default branch target | Experiment revision ancestry | Target 相對目前 pin 的 production 差異 |
| --- | --- | --- | --- |
| NWDAF | `origin/master` → `3f30ccc25ad916df37660feacc8f33a743d83df3` | `de8b385...` 是 ancestor；`3f30ccc...` 直接接在 `e2beba9...` 後 | default branch 相對目前 pin 新增 `LICENSE`、更新 `README.md`，並修正 `pkg/mockapp/app.go` 的 `mockgen` 絕對路徑 |
| NRF | `origin/main` → `0dd4024d4ab75b6630e04901968228b9b9718cf5` | exact match | 無差異 |
| PyMTLF | `origin/main` → `f8e63137c4f9ae9e7e046e629f8895d9154f139c` | `c31b8b6...` 是 ancestor | 僅新增 `LICENSE`、更新 `README.md` 與 `docs/api.md` |
| ADRF | `origin/main` → `6e2d5a9787111b31ebb531e67311ff0fe130b25e` | `905f059...` 是 ancestor | production code 無差異；新增授權／現行架構文件並移除 component 內舊規格與流程文件 |

使用者已核准 Slice 2 將四個 parent gitlinks pin 到上述 default-branch exact heads。NWDAF portability 修正已在 component
workspace 建立並推送為 `3f30ccc25ad916df37660feacc8f33a743d83df3`；本 workspace 已 fetch 並確認該 commit 的 parent 是
`e2beba9a5564c26be7938d3a03865a6b87d703e4`，diff 只有預期的一行 `go:generate` executable 變更。四者的 `.gitmodules`
branch hints 與 `components.lock.yaml` branch／commit 同步為 `master` 或 `main`。保留 exact commit pin，不把浮動 branch 名當成
可重現的 runtime identity。

## 4. Maintained topology 與 dependency 差距

`testbed.protocol-hierarchical.yaml` 是目前 maintained definition。其直接部署需求是：

- Guest：MongoDB、NRF、ADRF、Root／Branch／Leaf NWDAFs；
- Host containers：Root、四個 Branch 身分與六個 Leaves 的 PyMTLF；
- 四台 VMs、private management／SBI networks、generated config、dataset／seed-model preparation、provider guard、
  lifecycle、artifact collection、reset 與離線分析；
- WebConsole 不屬於公開版本，正式 E0–E2b 不啟動也不依賴它。

但 parent 目前仍 pin 16 個 submodules，另外包含 AMF、AUSF、NSSF、PCF、SMF、UDM、UDR、UPF、PyAnLF、UERANSIM、gtp5g
與 WebConsole。README 也明確保留 full 5GC、UE／RAN、PseudoDriver、subscription consumer、static Flat／Hierarchical
與舊 experiment examples 作 `legacy`／`unverified`。這些不是 maintained protocol topology 的直接 runtime dependency，
但仍被 Make targets、shared renderer／checker、Guest scripts、tests 與文件引用，不能只刪 submodule entry。

以下是已確認的高層 disposition；exact file／consumer change list 已在 Slice 2 詳細計畫展開：

| 分類 | 第一輪候選 | 下一輪必做 trace |
| --- | --- | --- |
| 保留 | protocol testbed definition、NRF／NWDAF／ADRF／PyMTLF pins、共用 config／provider／lifecycle、安全 guard | 確認每個 shared helper 不暗含 legacy-only owner |
| 保留 | E0–E2b 正式 scenario、paired seed model／dataset preparation、single-run／series runner、artifact collection 與 analysis | 建立從 scenario 到 raw evidence／reset 的完整 preservation map |
| 適配 | `.gitmodules`、`components.lock.yaml`、component docs、README、installation、build／preflight metadata | 使用核准的 default branch metadata 與 exact commits，保持 direct consumers 一致 |
| 移除 | full-5GC／UE／UPF／PyAnLF submodules 與其 deployment、subscription、PseudoDriver、traffic example、static topology 路徑 | 從 Make target、renderer、Guest unit、test、docs 追到最後 consumer；保留共用 protocol 部分 |
| 移除 | CIFAR-10 舊五類／Leaf `formal-baseline.yaml`、`formal-replacement.yaml` 與 `proximal-mu-0.1.yaml` 診斷情境 | 正式 series mapping 不引用；公開版本只保留 all-class-skew E0–E2b |
| 移除 | optional WebConsole | 使用者已決定公開版本不支援；Slice 2 移除其 submodule、config、build、lifecycle、tests 與文件入口 |
| 保留／移除 | MNIST／CIFAR-10 smoke | 只保留 `mnist/smoke.yaml` 作最小 protocol smoke；其餘 condition-specific smoke 移除 |

初步搜尋顯示，legacy consumer 跨越 `Makefile`、`config/default`、Host／Guest scripts、systemd unit、`tools/`、多個 tests
與 13 份 repository-local docs。`testbed-docs` 只保存本計畫，不是公開或 cleanup target。Slice 2 不能只按檔名執行
removal，必須遵循下節 owner／consumer disposition。

## 5. E0–E2b 正式實驗 preservation map

目前至少須保留並在 Slice 2／3 變更後驗證的範圍：

- authoritative deployment：`testbed.protocol-hierarchical.yaml`；
- 正式 scenario：MNIST 的 `formal-{baseline,replacement,reparent,partial-reparent}.yaml`，以及 CIFAR-10 的
  `all-class-skew-{baseline,replacement,reparent,partial-reparent}.yaml`；
- config／dataset／seed model：`config-render.py`、`configlib.py`、`config-check.py`、image-only `dataset.py` facade、
  `image_dataset.py`、`seed-model.py` 及其 direct tests；PseudoDriver-only `datasetlib.py`／`dataset-stage.sh` 不在保留鏈；
- runtime：PyMTLF Dockerfile／entrypoint、ML Compose rendering／checking、Guest NWDAF／NRF／ADRF build 與 service lifecycle、
  provider safety、selected process start／status／stop／reset；
- experiment：`fl-control.py`、`fl-experiment-run.py`、`fl_experiment.py`、`fl-series-run.py`、artifact collection retry 與所需 tests；
- analysis：`fl-analysis.py`、`fl_series_analysis.py`、analysis dependency 與保存的 raw schema；
- documentation：正式操作入口、E0–E2b definition／材料、component ownership 與公開環境建置說明。

這是第一輪 preservation map，不表示列出的整個檔案都原封不動保留；同檔中若混合 legacy branch，Slice 2 可在維持上述
behavior 的前提下縮減。反之，若下一輪 trace 發現未列出的共用依賴，必須補入而不能為了符合目前清單而刪除。

### 5.1 正式流程 consumer trace

目前正式單次 run 的 production flow 已從 entrypoint 追到結果 owner：

| 階段 | Authoritative entrypoint／consumer | 必須保留的行為 | 與 legacy 的關係 |
| --- | --- | --- | --- |
| Scenario selection | `fl-series-run.py` → `make config-create` → `config-render.py`／`configlib.py` | 依 workload、condition、seed 產生唯一 local config 與 manifest | renderer／config library 仍混有 production-flat、static 與 WebConsole branches，須在 Slice 2 拆除不支援分支而非整檔刪除 |
| Dataset／seed preparation | `dataset-generate` → `dataset.py`／`image_dataset.py`；generated manifest 指向 shared dataset 與 seed model | paired conditions 使用同 seed input，生成 Leaf split、held-out set 與 seed model | 舊 `datasetgen/`、Parquet／PseudoDriver path 與正式 image-dataset flow 不同，可成組移除 |
| Runtime admission | `fl-experiment-run.py` → `experiment-validate.sh`、provider state、GPU snapshot | 驗證 selected VMs 已啟動、config／dataset／capacity 可用 | 不需要 full-5GC、UE、UPF 或 subscription runtime |
| Build／start | `experiment-start.sh` → `services-start.sh`、`ml-start.sh`、`backend-check.sh` | 依 manifest 只啟動 MongoDB、NRF、ADRF、NWDAFs 與 PyMTLF containers | `experiment-start.sh` 仍含 WebConsole／Consumer branches；protocol manifest 令其 disabled／none，可於 Slice 2 收斂 |
| Runtime identity | Guest active config、service inventory、NRF registrations、container／image metadata | 證明本次 run 使用 selected source、config 與 process inventory | `components.lock.yaml`、image revision、gitlink 是同一 tracking contract，不可只刪其中一層 |
| Training／fault | `fl-control.py` 發出 training request；`fl-experiment-run.py` poll status、讀 Root observations 並依 scenario stop selected Branch／container | E0 不注入 fault；E1／E2a／E2b 依 authoritative scenario 執行 replacement／reparent | static Flat／Hierarchical controller branches 不是正式 protocol flow，可從共用 controller 移除，但須保留 protocol branch |
| Artifact collection | runner 複製 Root final model，停止 process，平行蒐集 PyMTLF observations 與執行 held-out evaluation | 即使蒐集失敗仍保留 retryable checkpoint，不重跑訓練 | 不依賴舊 subscription consumer 或 PyAnLF evidence |
| Cleanup／reset | `experiment-stop.sh`、`experiment-reset.sh apply/verify` | 停止 selected processes、清除 selected MongoDB／volume state、還原 canonical seed artifact | reset 必須保留 manifest-driven scope；舊 subscriber／subscription cleanup 可隨 production-flat path 移除 |
| Offline analysis | `fl-analysis.py`／`fl_series_analysis.py` 只讀 raw run directories | 由 40 個有效 runs 產生 paired comparison、latency、round metrics 與圖表 | 不依賴正在運行的 VM、component checkout 或 legacy scenarios |

這條 trace 顯示清理的主要形式應是縮減既有 shared renderer、checker、lifecycle 與 controller 的未支援 branches，而不是另外
建立一套 public-only pipeline。正式 flow 仍以目前 Make targets 和 runner 為唯一入口。

### 5.2 Cleanup groups 的 consumer-based disposition

| Cleanup group | 目前 consumers | 已確認 disposition／理由 |
| --- | --- | --- |
| Full 5GC／UE／UPF／gtp5g | `testbed.yaml`、`config/default`、Core／Path build scripts、subscriber fixtures、PseudoDriver dataset、production docs／tests | **移除**；不在 protocol manifest 或正式 run flow，且會把大量不再維護的 NF、RAN 與 kernel dependencies 帶入公開版本 |
| Production subscription／PyAnLF | Consumer tool／unit、subscription Host scripts、PyAnLF Compose／config、production-flat config branches、相關 docs／tests | **移除**；正式 protocol training 使用 private NWDAF／PyMTLF control，不消費這條 subscription chain |
| Static Flat／Hierarchical | 兩個 testbed definitions、topology configs、`fl-control.py`／`ml-status.py` branches、static tests 與 docs | **移除**；目前已標示 legacy／unverified，且其 seed-import contract 已無法通過現行 PyMTLF validation |
| 舊 CIFAR-10 五類與 `proximal-mu-0.1` scenarios | 手動 scenario selection 與歷史診斷；formal series mapping 不引用 | **移除**；不屬於最新 all-class-skew E0–E2b 定義，保留會讓公開使用者誤選 |
| Smoke scenarios | 手動短跑與部分 repository／container checks | **保留一個／移除其餘**；保留 `mnist/smoke.yaml` 作最小 protocol smoke，其他 condition-specific smoke 移除 |
| WebConsole | optional Make／Host／Guest lifecycle 與 config branches；正式 scenarios 一律 disabled | **移除**；公開版本不支援，不繼續 pin、build 或暴露操作入口 |
| Shared provider／config／lifecycle helpers | protocol flow 與 legacy flows 共用 | **保留並縮減**；刪除 legacy callers 後移除其分支，不複製 public-only helper |
| Repository-local docs | `5G_NWDAF_Infrastructure` 的 README、installation、commands、configuration 與 operations | **保留並更新**；只說明公開 source 實際支援的 protocol flow，不保留已移除入口 |

## 6. 本地 artifact 與實驗證據

目前 `runs/`、`.generated/`、`.cache/`、`config/local/*` 均由 `.gitignore` 排除，只有 `config/local/.gitkeep` 受追蹤。
第一輪容量與內容摘要：

| Path／artifact | 約略大小／內容 | Git 狀態與處置方向 |
| --- | --- | --- |
| `runs/` | 86 MiB | ignored；不隨 source cleanup 刪除 |
| `runs/protocol-hierarchical/e0-e2b/` | 80 MiB；42 個 `run.json`，包含 40 個有效 run 與 2 個已知失敗嘗試 | 本次正式實驗的 authoritative local raw evidence；先保留 |
| `.generated/image-datasets/` | 1.6 GiB | ignored；可由官方資料與 scenario 重建，但公開前先確認重建說明與是否另備份 |
| `.generated/seed-models/` | 約 1 MiB | ignored；可按 seed 重建，仍須保存生成契約 |
| `.cache/image-datasets/` | 174 MiB | ignored download cache；不作 source release 內容 |
| `config/local/` | 12 MiB | ignored generated configs；正式 run metadata 才是已執行設定證據，不能把現存 config 當公開 source |
| workspace root ZIP | `e0-e2b-formal-runs-20260922.zip` 與較廣的 `e0-e2b-complete-20260922.zip`，各約 8 MiB | 不在 repository；不公開，保留為本地 evidence |

Slice 2 的 source cleanup 不得刪除或搬動這些本地 evidence。公開內容只包含 source、正式 scenarios、生成／runner／analysis
工具與重建說明；實驗結果由論文內整理過的表格、圖形與敘述呈現，不發布 raw runs、final models、failed runs、generated
inputs、local configs 或交換 ZIP。若日後要清理 workspace artifact，須另列 exact targets 並取得 destructive-action 批准。

## 7. 公開風險第一輪結果

本輪已補齊 parent testbed 與四個 component 的 Git history，並對目前公開候選 refs 做候選檔名、內容 pattern、個人絕對路徑
與大型 blob inventory。環境沒有 `gitleaks`、`trufflehog`、`detect-secrets` 或 `git-secrets`，本階段沒有為盤點臨時安裝依賴；
因此以下是可重現的原生 Git pattern scan，不取代 Slice 3 在乾淨公開候選上執行的專用 secret scanner 與人工 license review。
結果如下：

- parent testbed 與四個 current component trees 未發現 private-key header、常見 GitHub token prefix、`/home/chingje`
  絕對路徑或 credential-like tracked filename；本地 reachable history 的常見 key／env／secret filename 查找也沒有候選。
- parent testbed 的完整 history 未發現上述 secret patterns、credential-like filenames 或個人絕對路徑，最大歷史 blob 約為
  380 KiB 的 `uv.lock`；沒有大型 raw dataset、model 或 run 被提交至該 source repository。
- component 完整 history 未發現 private-key header 或常見 GitHub／AWS／Slack token patterns。NWDAF 與 PyMTLF 中的
  `http://user:password@...example` 是測試拒絕 userinfo URL 的固定假資料，不是 credential。
- 個人絕對路徑確實存在於 reachable history：NWDAF 的 `e2beba9...` 仍可追溯到 `/home/x81u/go/bin/mockgen`，NRF history
  曾有 `/home/kun/openapi` replace，ADRF history 曾引用 `/home/king25158986/...`。使用者已決定不重寫 history；NWDAF current
  `origin/master` 已由 `3f30ccc...` 改成從 `PATH` 解析 `mockgen`，後兩者既有非敏感歷史字串接受保留。
- `5G_NWDAF_Infrastructure` 目前未發現 tracked license file，仍是公開 blocker。四個 component 的新 default heads 均有
  `LICENSE`；NWDAF、PyMTLF、ADRF 是在目前實驗 pin 後的文件整理 commit 補入，NRF 的目前 pin 已包含授權。
- `testbed-docs` 依使用者決策不是本輪公開候選；其 license、舊路徑、歷史 logs 與內容清理均不列為 release blocker，
  也不在後續 Slice 2／3 處理。
- PyMTLF history 的最大 blob 約 402 KiB，是 current tree 仍使用的 authoritative initial seed model；它不是本次 testbed
  generated cache，不能因副檔名是 model artifact 就自動移除。其他 component 未見非預期大型歷史 blob。
- GitHub organization URL、private subnet、NF identity 與 lab topology 不自動等同 secret；是否公開應依 credential、個資、
  授權與實際風險判斷，不做無根據的字串清除。

上述結果已覆蓋目前完整 reachable history，但仍缺專用 scanner 與最終候選 revision 的人工 review，因此不能據此宣稱
「可安全公開」。

## 8. Slice 1 交接結果

1. **整理 Slice 2 exact change list**：依已確認 decisions 將 tracking、legacy flow、WebConsole、smoke、license、NWDAF path 與
   repository-local docs 展開為 owner／file／consumer 級變更範圍；**已完成並建立 Slice 2 詳細計畫**。
2. **完成第一輪 exposure inventory**：已檢查目前候選 refs 的 current tree、reachable history、license 與大型 artifact；最終
   candidate 的專用 scanner 與人工 license review 明確交接 Slice 3，不新增永久 scanner test。
3. **提出 Slice 2 implementation proposal**：已依 Slice 2 文件向使用者列出 exact pin、刪除／適配清單、預計 repositories 與
   驗證範圍，並取得實作批准。

## 9. Slice 1 acceptance 與停止點

Slice 1 只有在下列項目都有直接證據時才可進入 user review：

1. 四個 component 的 merged target 與 current experiment commit ancestry 已確認；**已滿足**；
2. gitlink、`.gitmodules`、`components.lock.yaml` 與所有 direct build／runtime consumers 已列出；
3. E0–E2b preservation map 已覆蓋 input、generation、build、runtime、fault、artifact collection、reset 與 analysis；
4. 每個 cleanup group 都有 consumer-based disposition、替代路徑與文件／測試影響；
5. raw evidence、generated inputs、cache 與 ZIP 的保留／公開策略可區分，且不會在 source cleanup 中被誤刪；
6. 目前候選 refs 的 credential、license、第三方內容與 history 已完成第一輪盤點，最終 candidate scan 已明確交接 Slice 3；
7. 所有需要使用者決策的項目已明確列出，沒有用實作預先替使用者決定。

若 merged refs 無法取得、current experiment commit 不在預定公開 history、license owner 不明、正式 raw evidence 只有單一未備份
來源，或某個 cleanup 候選仍是 maintained flow 的必要 consumer，保持 `Decision Pending` 並停止於 Slice 1。不得為了進入
Slice 2 而弱化公開 acceptance、刪除證據或建立平行 workaround。

## 10. 已確認決策與後續發布關卡

使用者已確認：

- NRF、PyMTLF、ADRF 採 default branch metadata 加上述 exact gitlink／lock commits；NWDAF 採 `origin/master` 的
  `3f30ccc25ad916df37660feacc8f33a743d83df3`；
- `5G_NWDAF_Infrastructure` 採 Apache License 2.0；
- 不重寫 component history，只修正 NWDAF current `go:generate` 個人路徑；
- 移除 WebConsole；
- 只保留 `mnist/smoke.yaml` 作最小 protocol smoke，保留全部正式 E0–E2b scenarios；
- 移除 full 5GC、UE／UPF、gtp5g、PyAnLF、subscription／PseudoDriver、static FL 與過時 CIFAR-10 flows；
- 不公開 raw runs、final models、failed runs、generated inputs、local configs 或 ZIP；本地 evidence 不由 Slice 2 清理。

是否把最小 real-environment smoke 納入首次公開 acceptance，可在 Slice 3 根據最終 public candidate 再決定；各 repository
visibility change 仍須另外取得明確批准。
