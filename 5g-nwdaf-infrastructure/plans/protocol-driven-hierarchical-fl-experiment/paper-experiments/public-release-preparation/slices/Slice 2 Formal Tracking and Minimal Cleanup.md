# Slice 2：正式 Tracking 切換與最小公開清理

日期：2026-09-27

狀態：User Reviewed／Candidate Commit Created；file／consumer 級範圍已由使用者確認並完成實作，tracking、source cleanup、
focused verification 與 final fresh-read conformance gate 已通過；本 Slice 已依原決策和 Slice 3 repository-local 公開文件
一併建立 testbed candidate commit，後續 release verification 由 Slice 3 gates 接續

上層依據：[公開發布準備主計畫](../../Public%20Release%20Preparation%20Master%20Plan.md)與
[Slice 1 公開基線盤點](./Slice%201%20Public%20Baseline%20Tracking%20and%20Retention%20Inventory.md)。

## 1. 目標與停止邊界

本 Slice 將 `5G_NWDAF_Infrastructure` 收斂為只支援 protocol-driven hierarchical FL 與論文 E0、E1、E2a、E2b
正式實驗的公開候選 source。實作包含 component tracking 切換、舊 deployment／runtime branch 移除、正式 scenario 保留、
共用 runner／analysis pipeline 縮減，以及受影響測試的合理化。

本 Slice 不會：

- 改變 hierarchical FL protocol、E0–E2b 實驗參數、資料切分、fault timing、run schema 或分析公式；
- 重跑 40 組正式訓練；
- 刪除或搬動 `runs/`、`.generated/`、`.cache/`、`config/local/`、本地 ZIP 或其他既有實驗 evidence；
- 公開 raw runs、final models、failed runs、generated inputs、local configs 或交換 ZIP；
- 修改 `testbed-docs`／`nwdaf-docs` 的公開屬性，或把它們納入 source cleanup；
- 改變 hosting visibility、merge testbed branch、push、重寫 component history，或執行 real provider lifecycle。

若實作發現正式流程仍直接依賴預計移除的 component、設定或 runtime branch，必須先停止並更新本計畫；不能以保留一套
隱藏的 legacy pipeline 或弱化正式實驗契約來繞過問題。

## 2. 盤點結論：公開版本的唯一支援路徑

公開候選保留的 authoritative flow 是：

1. `testbed.protocol-hierarchical.yaml` 定義四台 VM、NWDAF／NRF／ADRF placement 與 Host PyMTLF inventory；
2. `config-create` 從一個保留的 scenario 產生完整 local config、runtime manifest 與 generated Compose；
3. `dataset-generate` 與 seed-model preparation 依 scenario／seed 產生 image-classification inputs；
4. `experiment-validate`、provider lifecycle、`experiment-start` 啟動 Guest NWDAF／NRF／ADRF 與 Host PyMTLF；
5. `fl-experiment-run` 執行 training、選定 fault、收集逐節點 observation、final artifact 與 held-out evaluation；
6. `fl-experiment-collect` 可在 training 已完成而收集失敗時重做後處理，不重新訓練；
7. `experiment-stop`／`reset` 管理 runtime state；`fl-series-run` 逐次呼叫相同 single-run flow；
8. `fl-series-analysis` 從 finalized raw run records 產生論文使用的彙整結果。

盤點確認正式 protocol flow 的 direct source components 只有 `NFs/nwdaf`、`NFs/nrf`、`NFs/adrf` 與 `ML/PyMTLF`。
root `compose.yaml`、`config/default/`、PseudoDriver dataset、Consumer subscription、WebConsole、full 5GC、UE／UPF、gtp5g、
PyAnLF 與 static FL 都不在此 flow 上。它們目前仍透過 renderer branches、Make targets、Guest installers、tests 與文件互相引用；
因此不能只刪 submodule，而要依下列 consumer map 一併縮減。

## 3. Component tracking 實作計畫

### 3.1 NWDAF 的修正 commit 與最終 pin

NWDAF portability 修正已由 component workspace 完成並推送到 `origin/master`：

- exact target：`3f30ccc25ad916df37660feacc8f33a743d83df3`；
- direct parent：`e2beba9a5564c26be7938d3a03865a6b87d703e4`；
- commit：`chore: make mock generation path portable`；
- source diff：只將 `pkg/mockapp/app.go` 的 `/home/x81u/go/bin/mockgen` 改為 `mockgen`，生成參數與 runtime behavior 不變。

本 workspace 已 fetch `origin/master`，並以 ancestry、commit metadata、file diff 與 `git diff --check` 確認上述結果；NWDAF
checkout、預計提交的 parent gitlink、`.gitmodules` 與 `components.lock.yaml` 均已對齊此 exact commit，branch metadata 使用
`master`。變更在 user review 前保持 unstaged，因此當時 parent index 仍保存既有 commit；正式 gitlink 已隨核准後的 parent
candidate commit 一起建立。

### 3.2 其餘 component targets

| Component | Branch metadata | Exact target |
| --- | --- | --- |
| NRF | `main` | `0dd4024d4ab75b6630e04901968228b9b9718cf5` |
| PyMTLF | `main` | `f8e63137c4f9ae9e7e046e629f8895d9154f139c` |
| ADRF | `main` | `6e2d5a9787111b31ebb531e67311ff0fe130b25e` |

四個 component 的 parent gitlink、`.gitmodules` URL／branch metadata 與 `components.lock.yaml` 必須一次對齊。實作不改
component remote URL；目前四者都已指向 `Intelligent-Systems-Lab` 的預定公開 repository。

### 3.3 Testbed 授權

在 repository root 新增標準 Apache License 2.0 `LICENSE`。不修改 component license，也不把第三方 submodule source 複製進
testbed tree。授權與 attribution 的完整人工 review 留在 Slice 3 的最終公開候選關卡。

## 4. Exact source disposition

### 4.1 Submodule、lock 與 build context

保留下列四個 gitlinks：

- `NFs/nwdaf`
- `NFs/nrf`
- `NFs/adrf`
- `ML/PyMTLF`

從 parent git index、`.gitmodules` 與 `components.lock.yaml` 同步移除下列十二個已無 supported runtime consumer 的項目：

- `ML/PyAnLF`
- `NFs/amf`、`NFs/ausf`、`NFs/nssf`、`NFs/pcf`、`NFs/smf`、`NFs/udm`、`NFs/udr`、`NFs/upf`
- `RAN/UERANSIM`
- `kernel/gtp5g`
- `webconsole`

`containers/ml/Dockerfile` 保留 PyMTLF image 與正式實驗使用的 seed-model／evaluation tools，移除 PyAnLF stage 與 static／legacy
volume target。`.dockerignore` 隨之只暴露 PyMTLF build context。Git history 已保存被移除的來源關係，本輪不另建 source archive。

### 4.2 Deployment definitions 與 committed config

保留：

- `testbed.protocol-hierarchical.yaml`
- `config/templates/` 下 protocol renderer 使用的 `nrfcfg.yaml`、`adrfcfg.yaml`、`nwdafcfg-root.yaml` 與
  `nwdafcfg-branch-leaf.yaml`
- `config/local/.gitkeep`
- `Vagrantfile`，但 service-to-component mapping 只接受 MongoDB、NRF、ADRF 與 NWDAF
- `provisioning.lock.yaml` 的通用 Host／Guest dependency lock

移除：

- `testbed.yaml`
- `testbed.static-flat.yaml`
- `testbed.static-hierarchical.yaml`
- `config/default/` 的舊 production-flat generated example；其中仍由 protocol renderer 使用的四個 native config 基底移至
  `config/templates/`，不隨舊 deployment 刪除
- root `compose.yaml`、`compose.cpu.yaml`、`compose.cpu-smoke.yaml`

正式 runtime 仍使用 `config-render.py` 產生在 selected `config/local/<name>/compose.yaml` 的 Compose；移除 root Compose 不會改變
正式 runner 的 Compose input。`config/templates/` 是 renderer-owned native skeleton，不是另一套 operator-selectable deployment
source；使用者仍只選擇 testbed definition、scenario 與 output name，正式 generated config 的欄位和值維持不變。
`testbed.protocol-hierarchical.yaml` 同時移除 `optionalServices.webconsole`，不以永遠為 `false` 的 dead schema 保留已刪功能。

### 4.3 Scenario set

保留九個 repository scenarios：

- `experiments/protocol-hierarchical/mnist/smoke.yaml`
- MNIST：`formal-baseline.yaml`、`formal-replacement.yaml`、`formal-reparent.yaml`、`formal-partial-reparent.yaml`
- CIFAR-10：`all-class-skew-baseline.yaml`、`all-class-skew-replacement.yaml`、`all-class-skew-reparent.yaml`、
  `all-class-skew-partial-reparent.yaml`

移除：

- `experiments/examples/` 的 PseudoDriver／full-core scenarios 與 traffic profiles；
- CIFAR-10 的 `formal-baseline.yaml`、`formal-replacement.yaml`、`proximal-mu-0.1.yaml`、`replacement-smoke.yaml`、`smoke.yaml`；
- MNIST 的 `healthy-comparison-smoke.yaml`、`partial-reparent-smoke.yaml`、`reparent-leaves-smoke.yaml`、
  `replacement-smoke.yaml`、`seeded-smoke.yaml`。

`experiments/local/.gitkeep` 保留給未提交的 operator-local scenario。測試若需要 fault fixture，直接使用保留的正式 scenario 或在
temporary directory 由 `mnist/smoke.yaml` 建立最小變體；不為測試重新加入 condition-specific smoke files。

### 4.4 Dataset 與 input preparation

保留 `scripts/host/image_dataset.py`、`seed-model.py` 與正式 runner 使用的 image-classification preparation。現有
`scripts/host/dataset.py` 改成 image-classification CLI facade，維持 `dataset-generate`、`dataset-validate`、`dataset-show`
operator interface，但移除 PseudoDriver dispatch、Go generator build/cache、`locate` 與 Path VM upload 語意。

移除：

- `scripts/host/datasetlib.py`
- `scripts/host/dataset-stage.sh`
- `scripts/guest/dataset-activate.sh`
- `tools/datasetgen/`
- Make `dataset-load` target

這項修改不改變 `.generated/image-datasets/` 的 layout、scenario dataset identity 或正式 run 對 dataset evidence 的引用。

### 4.5 Renderer、checker 與 runtime manifest

`configlib.py`、`config-render.py`、`config-check.py` 與 `ml-compose-check.py` 收斂成 protocol-hierarchical contract：

- deployment kind 只接受 `protocol-hierarchical`；
- 移除 production-flat／static-flat／static-hierarchical renderer 與 checker branches；
- 移除 mobile subscriber identity、UE／UPF／RAN、PyAnLF、Consumer、PseudoDriver 與 WebConsole schema／output；
- generated config 只包含 NRF、ADRF、各 NWDAF、各 PyMTLF、protocol topology、四台 VM network config、manifest 與 Compose；
- runtime manifest 不再保存永遠為 `none` 的 `subscriptions` 或永遠 disabled 的 `optionalServices.webconsole`；
- 保留 scenario、active／inactive branch selection、fault targets、capacity、reset scope、seed restoration 與 generated-file identity；
- `selected_component_paths()` 的合法結果收斂為四個保留 component，不再接受已刪 component kind。

這不是另寫一套新 renderer；應在現有 authoritative functions 上刪除 legacy branches，保持正式 config 的欄位與值不變。

### 4.6 Guest provisioning 與 service execution

保留並縮減：

- `common.sh`：保留 OS／Go／MongoDB prerequisites、runtime user、network 與共用 runtime tools；移除 dataset／Consumer 殘留；
- `core.sh`：只接受 `mongodb`、`nrf`、`adrf` 與 `nwdaf-*`；
- `path.sh`：只建置 `nwdaf-*`，移除 `kernel` action、gtp5g、UPF 與 UERANSIM build；
- `runtime-tools-install.sh`、`guest-tools-sync.sh`：只安裝／傳輸 protocol runtime 真正使用的 generic tools 與 systemd units；
- `service-run.sh`：只保留 MongoDB、NRF、ADRF 與 NWDAF mapping；
- `config-activate.sh`、network scripts、generic `5g-nwdaf@.service`、network unit、stack target、experiment reset scripts。

移除：

- `scripts/guest/gtp5g-check.sh`
- `scripts/guest/subscriber-data.js`
- `scripts/guest/webconsole-build.sh`
- `scripts/guest/systemd/5g-nwdaf-consumer.service`
- `tools/nwdaf-consumer/`

`experiment-reset.sh` 的 Host／Guest active-service guard 移除 Consumer special case，但保留 NRF／ADRF scoped reset、PyMTLF volume
清理、seed restoration identity 與「runtime 必須停止」的安全邊界。

### 4.7 Host lifecycle、觀測與 Make interface

完整移除下列 legacy host entrypoints：

- `gtp5g-preflight.sh`
- `subscriber-data.sh`
- `subscriptions-start.sh`、`subscriptions-status.sh`、`subscriptions-stop.sh`
- `webconsole-prepare.sh`、`webconsole-start.sh`、`webconsole-status.sh`、`webconsole-stop.sh`

在共用 scripts 中移除對上述入口的 conditional branches：

- `services-start.sh` 不再 stage PseudoDriver、執行 gtp5g preflight 或套用 subscriber data；
- `experiment-start.sh`／`experiment-stop.sh` 不再查詢 subscription mode 或 WebConsole；
- `observe.sh` 只並行收集 VM、Guest service 與 ML sections；
- `services-status.sh`／`experiment-status.sh` 移除 UE readiness 與 PyAnLF device branch；
- `lib.sh` 移除固定 legacy unit arrays、Consumer helpers、UE log interpretation、WebConsole／subscription getters 與
  `cpu-smoke` Compose override，保留 provider guard、selected-machine inventory、SSH transport、config identity、ML lifecycle、
  network／Guest identity 與 reset inventory helpers。

Make interface 保留正式流程及通用診斷入口，移除：

- `WEBCONSOLE` option 與 WebConsole targets；
- subscription／subscriber-data targets；
- `dataset-load`；
- `test-containers`；
- protocol runtime 不接受的 `fl-collection-*`；
- 與通用 runner 重複的 `fl-branch-replacement-run`／`fl-branch-replacement-collect` aliases。

`fl-experiment-run`、`fl-experiment-collect` 仍同時處理 E0 正常 run 與 E1／E2a／E2b fault run；不拆成 condition-specific
runner。

## 5. Test disposition

測試以公開版本真正支援的 contract 為單位，不替被刪檔案建立「必須不存在」的永久 meta-test，也不因 scenario 名稱或固定
leaf 名單重建過度驗證。

### 5.1 保留並聚焦 protocol behavior

| Test | 實作處置 |
| --- | --- |
| `clock-skew.py` | 保留 protocol clock evidence parser |
| `execution-policy.py` | 保留 provider guard、startup safety 與資源 gate；移除 production／WebConsole／PseudoDriver cases |
| `experiment-reset.js` | 保留 NRF／ADRF scoped reset contract |
| `fl-analysis.py` | 保留 single-pair 與 E0–E2b offline analysis behavior |
| `fl-control.py` | 移除 static contracts；保留 protocol training、runtime identity 與 response semantics |
| `fl-experiment.py` | 保留 runner、fault barrier、evidence／collect-only contract |
| `image-dataset.py` | 保留 scenario-driven shard、validation、held-out 與 deterministic generation behavior |
| `ml-status.py` | 移除 Flat／static log cases；保留 protocol container／runtime identity status，training request 進度由 `fl-training-status` 負責 |
| `network-config.py` | 改用 protocol generated／temporary fixture，不再依賴 `config/default` |
| `provider-runtime-preflight.sh` | 改用四台 protocol machine inventory，仍只使用 synthetic provider fixtures |
| `provisioning-lock.py` | 保留 Guest dependency resolution contract |
| `runtime-inventory.py` | 只覆蓋 MNIST smoke 與保留的正式 fault scenarios、四 component selection、generated Compose |
| `testbed-definition.py` | 改驗證唯一 protocol definition 的 machine／placement／network ownership |
| `repository.sh` | 收斂為 shell syntax、provider guard 與上述 focused tests 的薄入口，不保留大型 legacy scenario script |

`config-contract.py` 的現有內容完全由 production-flat／PyAnLF／WebConsole 契約構成；protocol config contract 已由
`runtime-inventory.py`、`fl-control.py`、`image-dataset.py` 與實際 `config-check.py` 覆蓋，因此直接移除，不建立同內容的重複檔案。

### 5.2 隨 owner 一併移除

- `consumer-state.py`
- `dataset-determinism.sh`
- `dataset-summary.py`
- `ml-container-lifecycle.sh`
- `mobile-identity.py`
- `static-topologies.py`
- `tests/support/ml-cpu-config.py`
- `tests/support/pymtlf-smoke-health.py`

刪除理由分別是 Consumer、PseudoDriver、Flat／static container lifecycle、UE identity 與 static topology owner 已被移除，並非
因測試失敗或為縮短測試時間而省略仍受支援的行為。

## 6. Repository-local 文件與 artifact 邊界

Slice 2 只做維持 source coherence 所需的文件調整：移除已刪 command／file 的入口與 dead links，並將 dedicated
PseudoDriver traffic reference 移出目前導覽。root README、architecture、installation、configuration、operations、commands、
troubleshooting 與 component map 的完整公開版重寫仍屬 Slice 3，不能在 Slice 2 結束時宣稱文件已可發布。

`.gitignore` 目前已忽略 `config/local/`、`.generated/`、`.cache/`、`runs/`、logs 與 runtime state，盤點未發現需要靠新增
廣泛規則才能保護的 tracked artifact；沒有實際 gap 就不修改。`.dockerignore` 則因 build context 移除 PyAnLF 而必須調整。
本地 ignored evidence 與 workspace root ZIP 不是 source cleanup target。

## 7. 實作順序與 repository checkpoints

1. **NWDAF upstream target verification**：已確認 `origin/master` 的 `3f30ccc25ad916df37660feacc8f33a743d83df3`
   只包含預期的 `go:generate` portability 修正，parent 可直接使用該 exact SHA。
2. **Parent tracking**：在 `5G_NWDAF_Infrastructure` 對齊四個 gitlink、`.gitmodules` 與 `components.lock.yaml`，先證明四個
   checkout 與 selected-component resolution 一致。
3. **Source removal**：依第 4 節由 owner 外向內移除 submodules、legacy definitions／configs、renderer branches、Guest／Host
   lifecycle 與 Make targets；不建立平行 public-only scripts。
4. **Scenario與測試縮減**：保留九個 scenarios，調整 focused protocol tests，刪除只服務已移除 owner 的 tests。
5. **最小文件 coherence**：修正因刪除造成的 dead command／link，明確標示完整 public documentation 尚待 Slice 3。
6. **Initial review**：檢查 testbed repository 的完整 diff、verification、remaining gaps 與 plan conformance；所有本地變更
   保持 unstaged／uncommitted，等使用者 review。NWDAF upstream commit 只作 tracking evidence，不納入本 workspace 的 commit。
7. **Commit proposal**：review 通過後提出 testbed repository 的完整 commit message；commit approval 不授權 push。

## 8. 驗證計畫

### 8.1 不啟動 provider 的 focused verification

- shell syntax 與 Python import／compile checks 只覆蓋保留 source；
- synthetic provider guard 與四台 VM inventory fixture；
- `components.lock.yaml`、gitlink、selected component checkout 與 generated image revision 一致性；
- MNIST smoke 的 config render、config check、dataset generation／check、seed-model preparation 與 generated Compose validation；
- 一個保留的 fault scenario 證明 inactive replacement selection、fault target、runner contract 與 collect-only path 仍可解析；
- E0–E2b formal scenario matrix 的靜態 identity／seed override／analysis slot mapping；
- focused runner、FL control、image dataset、ML status、reset 與 analysis tests。

### 8.2 Build 與 runtime 邊界

- PyMTLF container image 可由保留的 submodule revision build，OCI revision 與 lock 一致；
- NRF、ADRF、NWDAF 的 Guest build input 可由保留 source 與 lock 解析；
- real provider command 不在 sandbox 內執行；若 Slice 2 review 前需要 real VM build／smoke，另依 provider approval flow 執行，
  並只驗證代表性 MNIST smoke，不重跑正式矩陣。

不要求重新訓練即可驗證 source cleanup，因本 Slice 不改 component behavior、scenario parameters 或 formal evidence schema。
若 config、dataset、seed 或 runner output identity 因清理而改變，視為實作偏離並停止，而不是接受新的實驗基線。

## 9. Acceptance 與剩餘關卡

Slice 2 可進入 user review 的條件是：

1. NWDAF portability 修正的 exact commit 已經遠端驗證，且 parent pin 到 `3f30ccc25ad916df37660feacc8f33a743d83df3`；
2. parent testbed 只追蹤四個保留 components，三層 identity 一致；
3. legacy submodule、deployment、PseudoDriver、Consumer、WebConsole、full 5GC、UE／UPF、gtp5g、PyAnLF 與 static FL consumers
   已按本文件完整移除；
4. 九個保留 scenario、single-run／series runner、collect-only、reset 與 analysis flow 維持原契約；
5. focused protocol tests 通過，且沒有為清理新增 absence test、scenario-name test 或被刪 owner 的替代 mock pipeline；
6. 本地 raw evidence、generated inputs、local configs、cache 與 ZIP 未被修改；
7. repository-local 文件沒有指向已刪入口的明顯 dead link，且完整 public documentation gap 明確交接給 Slice 3。

Slice 2 完成仍不代表 repository 可公開。乾淨 checkout、最終 secret／license／history review、完整 README／operations rewrite、
testbed branch merge／release commit、push 與 hosting visibility change 都是 Slice 3 或其後的獨立關卡。

## 10. 實作與 initial review evidence

Slice 2 user-review handoff 時的 working tree 已完成下列項目，並依 review gate 保持 unstaged／uncommitted：

- `.gitmodules`、`components.lock.yaml`、四個 retained checkout 與預計提交的 gitlink target 已對齊第 3 節 exact revisions；
  十二個無 supported consumer 的 submodule 已從 source tree 移除；
- renderer、checker、Guest／Host lifecycle、Make interface、dataset facade、container build context 與 tests 已收斂為單一
  protocol-driven hierarchical flow，正式九個 scenario 保持 tracked；
- root `README.md` 已提供目前最小可執行入口；尚未重寫的 detailed docs 已移出 current navigation，完整文件工作仍交由
  Slice 3；
- initial review 修正了重寫 CLI／test entrypoint 的 executable mode、已刪 smoke fixture 引用、Python environment routing、
  不可達 stop／registration branches，以及 capacity gate 與 FL idempotent／ambiguous-response evidence 缺口；
- repository host-only suite 通過；synthetic provider test 只使用 mock process／state inventory，未啟動 real Vagrant 或
  VirtualBox；
- 最後修正後已重跑完整 `make test`，並另外通過 `Vagrantfile` Ruby syntax、兩個 repository 的
  `git diff --check`、無 broken symlink 與 retained component checkout clean-state 核對；
- NWDAF、NRF、ADRF 皆以 retained exact revisions 實際完成 `go build ./cmd`，PyMTLF Dockerfile 的 `pymtlf` stage 以
  Buildx `cacheonly` output 完成建置；
- 以正式 MNIST seed 1 E0 scenario 重建的 NRF、ADRF、NWDAF、PyMTLF native configs 與 protocol topology，和正式 paper run
  config 逐檔一致；差異只存在於已核准移除的 manifest dead fields、scenario mapping order，以及 PyMTLF revision metadata；
- working-tree source scan 未發現仍可到達已刪 component／command 的 consumer；保留的 network alias migration code 仍有
  防止舊 Guest alias 干擾現行網路的直接 failure consumer，因此不隨字詞掃描刪除。

Slice 2 handoff 時尚未執行 real provider lifecycle、VM provisioning 或 training smoke。這些不是本 Slice source cleanup 的
必要完成證據；後續已由 Slice 3 完成公開文件、Gate A exposure／license review 與 candidate commit。Clean checkout、final
history scan、real-environment smoke、push 與 visibility change 仍依 Slice 3 的獨立 approval／evidence gates 保持未完成。
