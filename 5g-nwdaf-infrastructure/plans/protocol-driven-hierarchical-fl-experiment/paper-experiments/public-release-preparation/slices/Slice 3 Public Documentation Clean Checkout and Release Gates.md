# Slice 3：公開文件、Clean Checkout 與發布關卡

日期：2026-09-28

狀態：Remote Promotion Complete／Visibility Pending；repository-local 公開文件、Gate A、final exact candidate
`92b3e90beec38e8f53bc79b787d9926ec341f099` 的 Gate B clean-checkout verification，以及 Gate C
fresh-provision GPU MNIST smoke 均已通過。GPU config checker 與 runner stale-interface findings 已修正並納入
candidate；private remote 的 feature branch 與 `main` 已 fast-forward 到 candidate，remote clean clone 已通過，
visibility change 尚未執行

上層依據：[公開發布準備主計畫](../../Public%20Release%20Preparation%20Master%20Plan.md)、
[Slice 1 公開基線盤點](./Slice%201%20Public%20Baseline%20Tracking%20and%20Retention%20Inventory.md)與
[Slice 2 正式 Tracking 與最小清理](./Slice%202%20Formal%20Tracking%20and%20Minimal%20Cleanup.md)。

## 1. 目標與停止邊界

本 Slice 將 repository-local 文件對齊 Slice 2 收斂後的唯一支援路徑，並把首次公開前的驗證拆成可被實際執行與誠實主張的
release gates。完成後，公開使用者應能從 root README 找到環境需求、component identity、設定生成、單次 smoke、正式 series、
觀測、停止、reset、失敗收集與離線分析入口，而不會被導向已移除的 legacy flow。

本 Slice 的 implementation 範圍是：

- 重寫 `5G_NWDAF_Infrastructure` 既有 repository-local 文件，使其只描述 protocol-driven hierarchical FL；
- 移除只服務已刪 PseudoDriver traffic contract 的文件；
- 對 final candidate 執行文件一致性、source/build/config、license、secret 與 reachable-history review；
- 在使用者決定後，從真正的 clean checkout 執行 release-candidate verification；
- 若 real-environment smoke 納入 acceptance，依 provider safety boundary 執行一個最小 MNIST smoke。

本 Slice 不會修改 hierarchical FL protocol、E0–E2b scenario 參數、dataset partition、runner schema、分析公式或 component behavior，
也不重跑 40 組正式實驗。`runs/`、`.generated/`、`.cache/`、`config/local/` 與 workspace ZIP 仍是本地 evidence／artifact，
不是本輪清理目標。Git commit、testbed branch promotion、push、hosting visibility change 與 VM destruction 各自保留獨立批准關卡。

## 2. 目前盤點結論

### 2.1 Public candidate source 與 branch 狀態

- `5G_NWDAF_Infrastructure` local checkout 仍位於 `feat/hierarchical-fl-protocol-extension`，Gate B/C findings
  修正後的 committed HEAD 是 `92b3e90beec38e8f53bc79b787d9926ec341f099`；使用者核准後，private remote 的
  feature branch 與 `main` 均已 fast-forward 到此 exact candidate，沒有 merge commit 或 force push。
- Slice 2 的 tracking、cleanup、license 與 test changes 已和 Slice 3 repository-local 公開文件一併建立 testbed candidate
  commit；Gate A review 已由使用者確認。
- current tree 只保留 `NFs/nwdaf`、`NFs/nrf`、`NFs/adrf`、`ML/PyMTLF` 四個 submodules，checkout 與
  `components.lock.yaml` 指定的 exact commits 一致。
- root 已有 Apache License 2.0；四個 retained component 的 current exact heads 也各有 Apache License 2.0。Root 與
  retained component current trees 都沒有 `NOTICE`。是否需要新增 attribution 不能只由檔案不存在推論，仍須在 final candidate
  對實際納入／再散布的內容做一次人工 review。
- parent repository 的初步 native Git scan 未發現 private-key header、常見 token prefix、個人絕對路徑或大型 raw artifact；
  最大 reachable blob 約 380 KiB。這只延續 Slice 1 inventory，不取代專用 secret scanner。
- Gate A/B 已使用一次性、版本固定的 Gitleaks v8.30.1，binary 與 reports 只保存在 `/tmp`；
  repository 沒有因此新增 scanner-specific permanent test、baseline 或第二套 release pipeline。

### 2.2 現行 operator contract

公開文件必須從 source 所擁有的單一路徑導出：

1. `testbed.protocol-hierarchical.yaml` 定義四台 VM、private networks、Guest service placement、十一個 Host PyMTLF
   containers、capacity、operation timeout 與 reset ownership。
2. 一個 committed／local scenario 加上 `NAME`、`DEVICE`、正式實驗才需要的 `SEED`，由 `config-create` 產生
   `config/local/<name>/`；該目錄包含 native configs、topology、manifest 與 generated Compose。
3. `dataset-generate` 從官方 MNIST／CIFAR-10 source 下載至 `.cache/image-datasets/`，依 scenario 產生共用或 per-seed
   `.generated/image-datasets/`；正式 seed model 產生於 `.generated/seed-models/`。
4. `vm-up` 管理四台 provider VMs；`experiment-start` 啟動 Guest NRF／ADRF／NWDAFs 與 Host PyMTLF；
   `fl-experiment-run` 執行 training、可選 fault、collection、held-out evaluation、stop 與 scoped reset。
5. training 已完成而 collection 失敗時，`fl-experiment-collect` 可沿用既有 checkpoint 重做後處理，不重跑 training。
6. `fl-series-run` 依 selected workload、seeds 與 conditions 順序呼叫相同 config、dataset 與 single-run pipeline；
   `fl-series-analysis` 只讀 finalized run records，不需要正在運行的 VM。

Real provider 相關 target 即使名稱看似唯讀，也只能在 approved Host context 執行。`make test` 是 host-only／synthetic suite，
不會啟動 Vagrant、VirtualBox、VM 或 container；文件不得把兩者的 evidence 混為一談。

### 2.3 Repository-local 文件 disposition

Gate A 前，root `README.md` 只是暫時入口，`docs/README.md` 是 holding page，其餘詳細文件仍大量描述已移除的
`testbed.yaml`、static Flat／Hierarchical、full 5GC、UE／UPF、gtp5g、PyAnLF、PseudoDriver、Consumer
subscription 與 WebConsole。已實作的 disposition 如下；完成證據見第 6.1 節。

| 文件 | Slice 3 處置 | 公開後責任 |
| --- | --- | --- |
| `README.md` | 重寫並移除 release-preparation 暫時文字 | 專案目的、唯一支援 topology、快速開始、單次／series 入口、artifact boundary、文件導覽與 license |
| `docs/README.md` | 重寫 | 穩定文件索引，不保存 current phase status |
| `docs/architecture.md` | 重寫 | 四 VM／Host container placement、management／SBI network、Root／Branch／Leaf 關係、lifecycle 與 state owner |
| `docs/components.md` | 重寫 | 四個 component、gitlink／`.gitmodules`／lock 的角色、exact revision 策略、Guest／Host build boundary |
| `docs/installation.md` | 重寫 | Linux x86-64、VirtualBox／Vagrant、Docker Compose v2、Python／`uv`、GPU optional path、實際四 VM resource budget 與首次 provisioning |
| `docs/configuration.md` | 重寫 | 單一 testbed source、scenario、render options、generated config、local override 與不提交 artifact 的邊界 |
| `docs/commands.md` | 重寫 | 目前 Make targets、provider／host-only 分界、必要參數與 operator-visible effect |
| `docs/operations.md` | 重寫 | config／dataset、VM、single run、series、status／logs、collect-only、stop／reset、offline analysis 與 failure recovery |
| `docs/troubleshooting.md` | 重寫 | component/config mismatch、provider guard、VM inventory、Docker／GPU、dataset、service readiness、collection retry 與 reset safety |
| `docs/configuration/testbed-reference.md` | 重寫 | 現行 schema 的 VM、network、placement、analytics、ML runtime、capacity、operations 與 reset fields |
| `docs/configuration/scenario-reference.md` | 重寫 | image workload、partition、training、E0–E2b metadata、fault、observation 與 seed override semantics |
| `docs/configuration/native-config-reference.md` | 重寫 | renderer 產物、各 consumer、manifest／Compose、Guest activation 與 advanced-edit boundary |
| `docs/configuration/dataset-reference.md` | 重寫 | 官方 source cache、Leaf shards、validation／held-out、dataset identity、per-seed sharing 與 regenerate／validate flow |
| `docs/configuration/traffic-profile-reference.md` | 移除 | 只描述已刪除的 PseudoDriver JSON／Parquet traffic contract，現行 image pipeline 沒有 consumer |

不另建 `public/`、`release/` 或第二份 operations tree。文件 prose 維持現有 repository 的 English；plan 與 review evidence
繼續使用本系列的中文。

### 2.4 Artifact 與歷史邊界

公開 source 應說明如何重建 inputs，但不包含下列本地內容：

- `config/local/` generated native config；
- `.cache/` 官方 dataset download cache；
- `.generated/` partitioned datasets 與 per-seed model；
- `runs/` raw records、observations 與 final models；
- `.vagrant/`、container volume、VM disk／image、logs、PID 與交換 ZIP。

目前 workspace 內仍有舊 config、cache、正式 run 與其他 ignored local artifacts。它們不會出現在 clean clone，也不應為了
public tree cleanliness 在 Slice 3 直接刪除。Final exposure review 必須分清 tracked tree／reachable history 與 ignored local
evidence，不能把掃描 ignored evidence 的結果誤稱為將被公開的內容。

Current-tree cleanup 不會從 Git history 抹除舊 topology 或被刪 source。除非 final scanner 發現真正敏感 material，維持已確認的
「不重寫 history」方向；單純過時、已不在導覽或 current tree 的程式不構成 rewrite history 的理由。

## 3. 文件實作順序

### 3.1 先建立穩定導覽與架構

1. 以 current source／Make interface 重寫 root `README.md` 與 `docs/README.md`，清除 interim wording。
2. 重寫 architecture 與 components，先固定公開使用者應理解的 owner、placement、identity 與 lifecycle。
3. 重寫 installation，所有 prerequisite、VM count、RAM／CPU／disk 與 GPU 敘述直接來自 current source；不沿用舊三台 VM
   或五個 container 數字。

### 3.2 再重寫 operator workflow

1. 重寫 configuration 與四份 retained reference，使 testbed／scenario／generated artifacts 的 ownership 一致。
2. 移除 `traffic-profile-reference.md`，並清除所有 navigation／cross-link。
3. 重寫 commands 與 operations，範例只使用保留的 MNIST smoke 或 E0–E2b scenarios；provider target 清楚標示 Host-only approval
   boundary。
4. 重寫 troubleshooting，只保留現行 owner 與可操作 recovery；不把已刪功能改寫成「optional」。

文件不得複製 active plan 的進度、目前 branch／commit 或本機結果。Component exact revision 的 authoritative source仍是
gitlink 與 `components.lock.yaml`；README 解釋策略與查詢方式，不手動維護另一份 SHA 清單。

## 4. Final exposure 與 license review

### 4.1 Scan 對象

Final candidate 要分 repository 檢查：

- `5G_NWDAF_Infrastructure` 的 final committed tree 與所有將由 public default branch 可達的 history；
- retained NWDAF、NRF、ADRF、PyMTLF exact commits 的 tree 與將公開的 default-branch reachable history。

`testbed-docs`、本地 raw runs、generated inputs、config cache 與 ZIP 不在公開候選；不得因本輪 scan 直接刪除它們。

### 4.2 方法與 finding 處理

1. Commit 前先以 source search 與專用 scanner 的 directory mode 覆蓋 final working tree，包括 uncommitted documentation。
2. Release candidate commit 建立後，以同一個版本固定的一次性 scanner 對五個 repositories 執行 Git／history scan。
3. 另以 native Git inventory 檢查 credential-like filenames、remotes、個人絕對路徑與異常大型 blobs，避免單一 scanner 規則
   漏掉非 secret 的公開風險。
4. 人工檢查 root／component licenses、Dockerfile 直接取用的 base／wheel、Vagrant box、MongoDB repository 與文件外部連結，
   判定是否存在需要保留的第三方 notice 或 attribution。這是發布盤點，不是法律保證。
5. 每個 scanner hit 依實際 material、consumer 與公開影響分類；sample credential、測試拒絕案例、private subnet 或一般
   GitHub URL 不因字面命中就自動成為 blocker。

若發現 live credential／private key，先停止發布並撤銷或輪替；若 material 已在 reachable history，才提出 history rewrite
決策。若只有已確認的非敏感歷史個人路徑，依 Slice 1 決策接受保留。Scan 結果記錄在本 Slice 的 review evidence，不新增
scanner-only permanent test 或 committed allowlist，除非發現跨 release 必須持續維護的實際 contract，再另行決策。

## 5. 驗證與發布順序

真正的 clean checkout 只能指向已存在的 commit。為遵守使用者「全部完成後再 commit」的要求，又不虛構尚未存在的 release
revision，本 Slice 將驗證分成下列關卡。

### Gate A：Commit 前的 final working-tree review

文件完成後，在原 working tree：

- 將每個 README／docs command、path、scenario、service count 與 source／Make target 逐一對照；檢查 Markdown links；
- 執行 `make help`、`make help-advanced`、`make help-dev`，確認文件導向的 operator surface 真實存在；
- 執行 `make test`、必要的 config render／check、dataset generate／check，以及 retained component focused build；
- 執行 `git diff --check`、broken-symlink check、component checkout／lock alignment 與 final working-tree exposure scan；
- 完成 initial review、plan conformance、文件 language-consistency pass，保持所有變更 unstaged 等待 user review。

這一關可以證明 final diff 與目前 workspace 的 host-only behavior，但不能宣稱 clean checkout、remote clone、fresh VM provisioning
或 public accessibility 已驗證。

### Gate B：核准 commit 後、push 前的 clean checkout

使用者 review 並核准 combined commit proposal 後，先在 feature branch 建立 exact candidate commit，再於獨立 temporary directory：

1. 從該 commit 建立不帶現有 ignored artifacts、`.venv`、submodule checkout 或 runtime state 的 clean clone；
2. 依 `.gitmodules` 初始化四個 exact submodules，確認 `components.lock.yaml`、gitlinks 與 checkout 一致；
3. 以 lockfile 準備 parent 與 PyMTLF Python environments；
4. 執行 repository suite、MNIST smoke config render、dataset download／generation／validation、config check、Go component builds 與
   PyMTLF image build；
5. 對 actual candidate commit 與四個 component histories 完成第 4 節 final scan。

這些步驟可能需要 network、Docker daemon 與較大 build resource，執行時另走相應 approval。Temporary checkout 與 build output
不得加入 source commit。

### Gate C：Real-environment smoke（已確認納入 acceptance）

使用者已確認將 real smoke 納入首次公開 acceptance，因 Slice 2 實際修改了 Vagrant placement、Guest provisioning、service lifecycle、
generated config 與 container inventory；host-only tests 與 clean builds不能證明 fresh Guests 可完整啟動。

已確認的 smoke 邊界是：

- 使用 clean checkout 的 `experiments/protocol-hierarchical/mnist/smoke.yaml`；
- `DEVICE=gpu`，因正式 E0–E2b runner 固定使用 GPU，這最接近本次公開的主要實驗路徑；CPU render／host-only contract 留在 Gate A/B；
- fresh provision 四台 VMs，啟動所有 selected Guest／Host processes，完成兩個 accepted rounds、final collection、held-out evaluation、
  stop 與 scenario-scoped reset；
- 不注入 fault、不重跑正式 matrix，也不從 smoke accuracy 推論論文結果；其證據只涵蓋 clean provisioning 與最小 distributed
  training lifecycle。

由於 VM names 與 provider state 是固定 identity，clean checkout 無法安全地和現有四台同名 VMs 並存。執行前
必須在 approved Host context 解析 exact provider inventory，確認現有 runtime 已停止，並另外取得刪除現有四台 selected VMs
與重建的明確批准；不以新增 namespaced topology 或修改 Vagrantfile 繞過。Local runs、datasets、config 與 ZIP 不在 VM destroy
scope。這項設計決策本身不授權刪除或重建 VM；若後續無法取得 real-provider 或 destructive-action approval，Slice 3 應保持
`Verification Incomplete`，並把公開聲明限制為 host-only／build／config 已驗證，不能寫成 fresh testbed 已跑通。實際 Gate C
已在另行取得明確批准後執行，結果見第 6.3 節。

### Gate D：Default branch、push 與 visibility

使用者已確認 public default branch 使用 `main`。盤點時 `main` 是 feature branch 的 ancestor，因此在 remote 沒有新 divergence 且 final
review 通過時，可採 fast-forward promotion，避免額外 content-changing merge；實際操作前仍須 fetch 後重新確認 ancestry。
若 remote 已分歧，或使用者要求 PR／merge commit，須先更新 promotion proposal，不能自行 rebase、force-push 或 rewrite history。

Branch promotion 與 push 必須另提完整 proposal。Private remote 上的 final default-branch commit 建立後，應從 canonical remote
再做一次 clean clone，確認 remote availability、submodule URLs 與 candidate identity；若 tree 未改變，可沿用 Gate B/C 的 build／
runtime evidence，只補 remote-boundary evidence。Hosting visibility 改為 public 仍是最後一個獨立批准；公開後再以未認證 clone
做最小 accessibility check，不把 visibility change 視為 source verification。

使用者已核准 source promotion 與 push。執行時先 fetch 並再次確認 remote `main` 與 feature branch 都是 candidate 的
ancestors，再以 atomic、非 force push 將兩者 fast-forward 到 `92b3e90...`。後續 remote clean clone evidence 見第 6.4 節；
hosting visibility 仍未授權。

## 6. Initial conformance map

| 主張 | Direct owner／evidence | 目前狀態 |
| --- | --- | --- |
| 公開文件只描述 retained protocol flow | rewritten root／`docs/` 對照 source、Make targets 與 scenarios | Gate A 通過 |
| 公開使用者可理解 source、generated 與 local artifact boundary | README、configuration、dataset、operations | Gate A 通過 |
| Legacy traffic reference 不再出現在正式導覽 | remove file、link scan、semantic docs review | Gate A 通過 |
| Final tree／history 沒有未處理 sensitive material | version-fixed Gitleaks、native Git inventory、人工 triage | `92b3e90...` 與四個 component histories 通過；命中皆已分類為公開 identity 或歷史 lab／demo fixture |
| Retained licensing／attribution 可接受 | root／component licenses 與 direct external-input inventory | source candidate review 通過 |
| Candidate 可由 clean checkout 初始化 exact dependencies | post-commit clean clone、submodule／lock check | `92b3e90...` 通過 |
| Candidate 可完成 build／config／dataset pre-runtime flow | working-tree suite 與 Slice 2 focused builds；clean checkout 重跑 | `92b3e90...` 通過 |
| Fresh real testbed 可完成最小 distributed training lifecycle | approved Host context 的 fresh-provision GPU MNIST smoke | Gate C 通過；2 個 accepted rounds、collection、evaluation、stop 與 reset 完成 |
| Public default branch 與 canonical remote 可取得 candidate | approved promotion／push 後 remote clean clone | Private remote `main` 已 promotion，authenticated clean clone 通過 |
| Anonymous public access 可用 | visibility approval 後 unauthenticated clone | 尚未授權 |

### 6.1 Gate A implementation 與 review evidence

Repository-local 文件已依第 2.3 節完成：

- root `README.md` 與 `docs/README.md` 現在提供唯一 retained protocol flow 的穩定導覽；
- architecture、components、installation、configuration、commands、operations 與 troubleshooting 已由 current source 重寫；
- testbed、scenario、native config 與 dataset 四份 reference 已對齊目前 schema、owner 與 artifact boundary；
- `docs/configuration/traffic-profile-reference.md` 已移除，文件與 current source 搜尋未留下已刪 PseudoDriver／full-5GC／static
  flow 的正式入口；
- 13 份公開 Markdown 文件的 repository-relative links 全部存在。

Initial review 確認一項 admitted finding：

- **S3-F1 / closed**：Slice 2 移除 optional-service contract 後，`scripts/host/preflight.sh` 仍以三個參數呼叫目前只接受
  `testbed, runtime` 的 `selected_component_paths()`，real `experiment-validate` 會在 provider validation 前發生
  `TypeError`。修正只移除已不存在的第三參數；focused call 回傳 `ML/PyMTLF`、`NFs/adrf`、`NFs/nrf`、`NFs/nwdaf`
  四個 selected paths，後續完整 `make test` 通過。此修正保留既有 architecture、ownership、scope 與 verification level，
  未新增 permanent meta-test。

Gate A verification：

- `make help-all` 通過，文件使用的 command surface 存在；
- `make test` 通過，涵蓋 shell/Python syntax、testbed definition、execution policy、network config、provisioning lock、FL control、
  experiment behavior、ML status、offline analysis、runtime renderer／config、image dataset 與 synthetic provider/reset checks；
- renderer／config 與 dataset generate／reuse／native validation 由 suite 的 temporary artifacts 直接執行；Slice 2 已通過且其後
  未變更的 NRF、ADRF、NWDAF Go builds 與 PyMTLF Docker build evidence 仍適用；
- `git diff --check`、public-candidate broken-symlink check、component working-tree cleanliness，以及 checkout／
  `components.lock.yaml` exact revision 對照通過；
- 未執行任何 real Vagrant／VirtualBox、VM 或 container operation。

Current-tree exposure review 使用官方 Gitleaks release v8.30.1；Linux x64 archive 先以 release checksum 驗證，binary 及 report
只放在 `/tmp`，未加入 repository。為避免把 ignored raw runs、cache 與 generated artifacts 誤算成公開內容，directory scan
使用 current tracked／intended-untracked files 的暫存 public-tree projection，並分別掃描四個 component current trees：

- NWDAF、NRF、ADRF：無命中；
- testbed：10 個 `generic-api-key` 命中，為九份 scenario 的 PyMTLF `seedArtifactKey` 與
  `provisioning.lock.yaml` 的 MongoDB signing-key fingerprint；
- PyMTLF：2 個 `generic-api-key` 命中，為 example config 的 model `artifact_key`。

上述值是公開 component-native model identity 或公開 signing-key identity，具有直接 consumer，不是 credential；不移除、不建立
scanner allowlist。Native filename／path／remote inventory也未發現 credential-like filename、個人 workspace path 或私人
component remote。Root 與四個 retained component current trees 都是 Apache License 2.0 且沒有 `NOTICE`；本次 public source
candidate 只保存 external dataset／Go／MongoDB／base image／wheel 的取得與 build instructions，不再散布其下載內容，因此本輪
未辨識出必須新增到 root 的第三方 notice。若未來另外發布 built VM box 或 container image，需對該 binary distribution
另做 license／notice review。

Gate A 只證明 commit 前的 working tree 與 host-only boundary。後續 Gate B 實際執行與 finding 如下。

### 6.2 Gate B clean-checkout evidence 與 admitted finding

首次 Gate B 從 candidate commit `b5cc0af979be85e1a9d28b83557acacba0d98f5c` 在獨立 `/tmp` 目錄建立 clean
clone，沒有沿用 workspace 的 ignored artifacts、`.venv` 或 component checkout。直接 evidence 為：

- 四個 submodules 由 `.gitmodules` 的 canonical GitHub URLs 初始化，checkout 均 clean，並與 gitlinks 及
  `components.lock.yaml` 的 exact revisions 一致；
- parent 與 PyMTLF 的 lockfile environments 均由 clean clone 成功建立；
- exact candidate 的 `make test` 通過；官方 MNIST source 成功下載，smoke dataset 產生與
  native validation 通過，產出 600 train、200 validation 與 200 held-out samples；
- NWDAF、NRF、ADRF 的 explicit Go command builds 通過；PyMTLF Dockerfile 的 `pymtlf` target 完成
  build，未啟動 container；
- 全程未執行 real Vagrant／VirtualBox、VM 或 service-container lifecycle。

GPU smoke config 驗證承認一項 finding：

- **S3-F2 / closed**：`config-check.py` 在 `mlDevicePolicy=gpu` 時將所有 PyMTLF
  clients 一律預期為 `cuda:0`，因此拒絕 renderer 正確產生的 Branch `training.device: cpu`。Current
  architecture 的 authoritative placement 是 Root／Leaves 使用 GPU、Branches 使用 CPU；問題只在 checker
  將 global device policy 錯當成每個 backend 的 device。修正後，CPU render 仍要求全部為 CPU；GPU render 改為對照
  testbed source 中每個 backend 的 `device`。現有 runtime-inventory test 以同一 smoke scenario 同時跑 CPU
  與 GPU render/check，沒有新增 test framework 或寫死特定 unit names。Focused CPU／GPU checks、實際 MNIST
  generated GPU config validation 與原始 repository 的完整 `make test` 均通過。

Gitleaks v8.30.1 以 Git/history mode 掃描 candidate 與四個 retained component reachable histories。ADRF／NRF／NWDAF
無命中；testbed 與 PyMTLF 的 `generic-api-key` 命中由人工對照原始檔案與 consumer，分為公開
model artifact identity、MongoDB signing-key fingerprint，以及已從 current tree 移除的 full-5GC／UERANSIM
lab/demo cryptographic fixtures。後者是固定測試網路輸入，在歷史 reference／resource trees 也可對照，不是可存取現行
service 的 live credential；保留可達 history，不觸發 credential rotation 或 history rewrite。Native Git inventory
另確認沒有非預期大型 raw artifact 或私人 remote；已知的歷史個人絕對路徑沿用 Slice 1 的接受決策。

S3-F2 修正首先 amend 為 `ecf8030ed23b389edf3c0961327018e7318a611b`。從該 exact commit 建立的新
clean checkout 重建四個 submodules 與 parent／PyMTLF environments，再次通過完整 `make test`、GPU smoke
config validation、官方 MNIST download／generation／validation、NWDAF／NRF／ADRF builds、PyMTLF image build 與
五個 repositories 的 Gitleaks Git/history scan。Gate C 後續 finding 修正後，final source 再 amend 為
`92b3e90beec38e8f53bc79b787d9926ec341f099`，另建 clean checkout 重跑 exact submodules／lock、兩套
Python environments、完整 `make test`、GPU config／官方 MNIST dataset validation 與五個 history scans；結果均通過。
Component source 未變，前一個 exact candidate 的 Go／PyMTLF image builds 仍是直接 evidence。Final clean checkout 的
tracked tree 與 submodules 均 clean，Gate B 關閉。

### 6.3 Gate C fresh-provision GPU smoke evidence

使用者授權後，四台舊 VM 先經 guarded lifecycle graceful halt，確認皆為 `poweroff` 後才刪除；刪除範圍
只有 `5g-nwdaf-core`、`5g-nwdaf-path-a`、`5g-nwdaf-path-b`、`5g-nwdaf-path-c` 與 associated
drives，本地 runs、dataset、config 與 ZIP 未刪除。Clean checkout 隨後從 Ubuntu Jammy base box fresh provision
四台 VM，Guest network、provisioning lock、MongoDB／Go dependencies 與 component builds 全部通過。

Real smoke 承認並關閉兩項 finding：

- **S3-F3 / closed**：`fl-experiment-run.py` 建立現行 `FL_CONTROL.Controller(contract, http)` 時仍傳入已移除、
  也沒有 consumer 的 `poll_seconds`，導致 runtime ready 後、training request 前發生 `TypeError`。修正只移除
  stale argument；poll interval 仍由 runner 的 `FLExperimentContract.poll_seconds` 用於實際 loop。
- **S3-F4 / closed**：送出 training request 後，runner 仍讀取已從 FL-control request contract 移除的
  `closure_budget_seconds`。修正不重建第二份 config truth；總 budget 改由 `run.json` 已記錄的
  `preparationTimeoutSeconds + acceptedRounds × roundTimeoutSeconds + maximumTotalSeconds` 計算，來源仍是實際
  generated Root／testbed deadline contract。

兩項修正均保留 protocol、timeout 值、polling、dataset 與 acceptance；每次 remediation 後均通過完整
`make test`。最後的 `release-gate-c-mnist-smoke-20260928-c` 在 GPU 模式成功完成：

- runtime ready，Root 與 6 Leaves 使用 `cuda:0`，3 Branches 使用 CPU；
- 2/2 accepted rounds，每輪 Root 的 3 個 Branch contributors 全部成功，無 failed contributor；
- final model、10 個 PyMTLF raw observation files 與 held-out evaluation 成功收集；
- process cleanup 與 restart-policy restoration 通過，scenario-scoped reset 驗證為 empty；
- `run.json` 狀態為 `successful`、`finalized: true`、`failures: []`，總執行約 174 秒。

Smoke 只使用每個 Leaf 100 筆 training samples、1 local epoch 與 2 rounds；held-out accuracy 為 0.10 只證明
evaluation path 可執行，不是模型品質 acceptance 或論文結果。前兩次失敗 run 只在 temporary checkout 保存診斷
evidence，不納入 public source 或本次成功 smoke 紀錄。

### 6.4 Gate D source promotion 與 remote-boundary evidence

Push 前重新 fetch 後，remote `main` 仍為 `7d0a36c...`，remote feature branch 仍為 `44aa06a...`；兩者都是
candidate `92b3e90...` 的 ancestors。核准後以同一次 atomic、非 force push 將
`feat/hierarchical-fl-protocol-extension` 與 `main` fast-forward 到 candidate，沒有建立 merge commit 或改寫既有
history。`testbed-docs` 的既有 approved commit 也另行推送到其 `main`。

隨後從 canonical GitHub remote 的 `main` 建立新的 authenticated clean clone，確認：

- parent HEAD 是 `92b3e90beec38e8f53bc79b787d9926ec341f099`，tracked tree 與 submodules clean；
- `.gitmodules` 可取得 PyMTLF、ADRF、NRF、NWDAF 四個 canonical HTTPS repositories；
- 四個 submodule checkouts 分別為 `f8e6313...`、`6e2d5a9...`、`0dd4024...`、`3f30ccc...`，與 gitlinks 及
  `components.lock.yaml` 一致。

這項 remote-boundary evidence 證明 private canonical remote 可取得 candidate 與 exact dependencies；它不證明 anonymous
public access。Repository visibility 仍保持未變，須在獨立批准後公開，再執行 unauthenticated clone check。

## 7. 已確認決策與保留授權關卡

使用者於 2026-09-28 確認採用下列建議：

1. **Real smoke**：fresh-provision、GPU、兩 round MNIST smoke 納入首次公開 acceptance；現有 selected VMs 的
   exact destruction／rebuild 已另行取得批准並完成。
2. **Testbed branch promotion**：`main` fast-forward promotion 與 push 已在重新確認 remote ancestry 後完成；沒有 merge
   commit、force push 或 history rewrite。
3. **One-off scanner**：Gate A/B 使用暫存、版本固定的 Gitleaks，不把 binary、cache、baseline 或 scanner config 提交進
   repository；工具取得與任何 network operation 仍依執行環境另走 approval。

## 8. Slice 3 acceptance 與停止點

Slice 3 只有在下列項目滿足後才可進入 final user review：

1. root README 與 retained detailed docs 已依第 2.3 節完成，沒有指向已移除 owner、command、scenario 或 topology；
2. 文件 command／path／link 和 current source 一致，source／generated／local artifact boundary 清楚；
3. Gate A 完成且沒有未處理 current-slice finding；
4. real smoke、branch promotion 與 scanner 三個決策已依第 7 節記錄；Gate A／B／C 與 Gate D source promotion／
   remote clean clone 已執行並關閉，hosting visibility 與 anonymous access evidence 仍保持未完成；
5. Gate A handoff 時所有變更保持 unstaged／uncommitted，並向使用者提供兩個 repository 的 diff、verification 與 remaining
   release gates；user review 與 commit proposal 另行通過後才建立 candidate commits。

Gate A／B／C 與 source promotion／remote clean clone 已關閉。在 visibility approval 與 anonymous accessibility check
完成前，整體公開發布計畫仍不得標示 `Completed`。
