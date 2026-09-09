# Slice 1 Legacy Experiment And Hash Cleanup Detailed Plan

日期：2026-09-09

狀態：Ready for User Review；尚未授權 implementation

上層計畫：

- [Multi-host Branch Replacement Experiment Plan](../multi-host-branch-replacement-experiment-plan.md)
- [5G NWDAF Infrastructure Development Policy](../../../development_policy.md)

本文件只細化目前即將執行的 Slice 1。共同 topology、repository ownership、single pipeline、正式 campaign 與
跨-slice conformance 仍以上層主計畫為準；Slice 2、Slice 3 的詳細計畫會在進度到達對應 slice 時才建立。

---

## 1. Operator-visible outcome

Repository把canonical protocol-driven target與既有production Flat、static Flat／Hierarchical experiment assets清楚
分開。既有definitions、component／provisioning assets、scripts、commands與tests原則上保留，其legacy／unverified狀態
只記在`5G_NWDAF_Infrastructure/README.md`的legacy asset inventory；不要求逐一改寫help、既有docs、test名稱或source
config。Canonical acceptance不把它們當成supported evidence，canonical config validation也不再因trusted local
sources的nested digest chain產生無關mismatch。

既有WebConsole則維持獨立optional feature，不列為legacy，也不因canonical experiment未使用而刪除；其source、config、
build／lifecycle targets與既有tests保留，而canonical protocol-driven profile保持disabled。

此slice是support-boundary與cleanup checkpoint，不交付可執行的新實驗。Slice 1完成後，legacy assets仍可作後續遷移、
參考或best-effort手動使用，但不保證能跑，也不投入專屬修復。Canonical path不得隱式選取legacy profile；任何準備經
同一common lifecycle啟動的profile，仍須先通過共通靜態安全檢查。不能為維持暫時可跑而保留隱式default／fallback。

## 2. 盤點基線與邊界

本次 read-only inventory 以 `5G_NWDAF_Infrastructure` parent revision
`9a7bc914f40f3ab5e4f13b19a538c0a2a3c4d754` 為基線。Parent working tree 只有下列既有 submodule pointer 差異，本次
文件工作不修改或覆蓋它們：

| Path | Parent recorded revision | Actual revision | Disposition |
| --- | --- | --- | --- |
| `ML/PyMTLF` | `747962971b63f0a53031d52a1eb7e047ae776998` | `579f7375a8d0ff78054334eab2925a4ca4cefdaf` | 新實驗 dependency；保留 actual revision，後續另列入 implementation review |
| `NFs/nwdaf` | `6aed268d6528f8be6c729cbd45b59d067e5e80dc` | `be3fa576a22c6b787c9ce578c79621b486641e6d` | 新實驗 dependency；保留 actual revision，後續另列入 implementation review |

Inventory 範圍是 parent repository 自有的 source、scripts、tests、tracked examples 與 documentation。下列內容不視為
本 slice 要簡化的 testbed hash contract：submodule 內的 protocol／authentication cryptography、dependency lockfile
checksums、Git object IDs 的演算法細節，以及未由 testbed 產生或消費的第三方 implementation。

本 slice 不修改 `NWDAF`、`PyMTLF`、NRF 或 ADRF component behavior，也不操作 real VM、container、Vagrant 或
VirtualBox runtime。若清理需要改變component contract、建立新selector／renderer／checker、弱化destructive safety，
或清除selected exact scope以外的runtime state，必須停止並更新上層計畫。

## 3. Legacy flow disposition inventory

| ID | 現有範圍 | Slice 1 disposition | 理由與邊界 |
| --- | --- | --- | --- |
| L1 | `testbed.yaml`、`testbed.static-flat.yaml`、`testbed.static-hierarchical.yaml` 與 `config/default/` legacy baseline | retain and isolate | 三份既有definitions都保留為legacy／unverified。`config/default/`可保留作committed historical generated example，但不得是authoritative source或任何canonical command的implicit fallback；Slice 2另產生明確selected `CONFIG_DIR`。 |
| L2 | `experiments/examples/*` 的 UE／CAT scenarios 與 Path A／B traffic JSON | retain as legacy | 全部既有examples保留，README以群組說明legacy／unverified；不逐檔建立support metadata，也不投入修復。任何舊example都不能作為MNIST canonical scenario fallback。只有generated cache、secret或具有具體安全風險者可另案刪除。 |
| L3 | `scripts/host/dataset*.py`、`dataset-stage.sh`、`scripts/guest/dataset-activate.sh`、`tools/datasetgen/` | retain and isolate | 這些是既有UE／PseudoDriver dataset execution資產，雖不被canonical MNIST path使用，仍可保存舊實驗意圖與best-effort操作。Canonical renderer、preflight、stage與acceptance不得引用；本計畫不測試或修復舊pipeline。 |
| L4 | `fl-control.py` 與 `fl-collection-*`／`fl-training-*` Make targets | retain as legacy entrypoints | 它們控制既有retained-descriptor training，不是新Root→Branch→Leaf campaign controller，但仍是可辨識的完整舊operator surface。保留command與tests，不保證能跑；Slice 3優先擴充或重用既有control entrypoint，只有語意確實衝突時才以reviewed in-place adaptation取代相關branch。 |
| L5 | Consumer、subscriptions、subscriber fixtures 與相關 Guest unit／scripts／targets | retain and isolate | 整套source、fixtures、Guest unit、scripts與targets保留為legacy assets。Canonical selected service inventory、runtime tools allowlist、stage／start／status／stop／reset與acceptance均不得納入；保留不表示完整5GC data path仍是prerequisite或可執行。 |
| L6 | WebConsole source、config、build／start／status／stop flow、targets與tests | retain as optional | WebConsole原本已有獨立enablement與lifecycle boundary。Canonical protocol-driven profile保持disabled，本計畫不部署、adapt或執行real acceptance，但不能只因新experiment未使用就刪除或改列legacy。 |
| L7 | AMF、SMF、UPF、UERANSIM、NSSF、UDR、UDM、AUSF、PCF、PyAnLF、gtp5g 的 config、provision／build branches、submodule declarations 與 lock rows | retain outside canonical dependency inventory | 現有submodules、lock provenance、config與build／provision source保留，因其共同構成舊testbed且刪除不會降低canonical runtime cost。Canonical capacity、build、provision、start與acceptance只依selected minimal inventory，不能因source仍存在而執行這些components。 |
| L8 | root／`config/default`／generated Compose 中的 PyAnLF、舊PyMTLF或legacy seed／dataset entries | retain sources; replace canonical output | 既有Compose sources／examples保留。Slice 2沿用同一renderer在明確`CONFIG_DIR`產生只含canonical十一個PyMTLF services的Compose；root或`config/default` Compose不得成為canonical fallback。Legacy profile只有在同一renderer可安全產生時才可選取。 |
| L9 | `core.sh`、`path.sh`、service lifecycle、reset、status、observe 與 logs 內的 full-core／RAN／UPF role branches | retain dormant branches; adapt common path | 既有role branches原則上保留，不因canonical未使用而刪除。Slice 2將common path改為selected inventory；canonical reachability不得進入full-core／RAN／UPF branch，且任何branch不得繞過provider guard、config identity、exact scope或fail-closed checks。只有實際阻擋common adaptation或形成安全旁路的段落才可逐項移除。 |
| L10 | README、`docs/`、help與tests對production Flat、static Flat／HFL、UE traffic、WebConsole、subscriptions及舊commands的內容 | README-only support classification | 只在`5G_NWDAF_Infrastructure/README.md`新增legacy asset inventory並把protocol-driven experiment列為canonical target；既有詳細docs、help文字、tests與records原則上原樣保留，不逐處加legacy標籤。若舊test只驗證已替換的implicit default而阻擋canonical suite，可隔離或逐項移除，但不為使其通過而修復legacy flow。 |
| L11 | `Vagrantfile`、`lib.sh`、network／status／provider tests 的三機與 Path A／B 固定 inventory | defer canonical/common adaptation to Slice 2 | Slice 2只要求canonical與common reachable path由selected inventory驅動；不要求repository-global hard-code為零。Dormant legacy-only固定值可保留並列入allowlist，若能低成本mechanical migration再一併處理，不為legacy恢復或複製A／B專用branch。 |

### 3.1 Legacy preservation contract

保留邊界固定如下：

- 保存production Flat、static Flat／Hierarchical definitions、scenarios、component／provisioning sources、scripts、tests、
  participant／data-owner mapping、traffic／training參數及既有operator entrypoints。Generated cache、runtime state與
  secrets不屬於此source preservation contract；committed generated examples可保留但不得成為canonical authority。
- 保留資產的支援狀態只由`5G_NWDAF_Infrastructure/README.md`的legacy asset inventory維護；不在YAML、manifest或其他
  source config新增status欄位，也不建立特殊檔名規則、opt-in flag或額外lifecycle branch。它們不得成為implicit
  default，也不得被canonical experiment commands內部選取；operator明確指定既有legacy `TESTBED`／command時仍可
  best-effort嘗試，但不構成支援承諾。
- Slice 1不建立新schema或遷移profile；它只保留資產、移除錯誤的supported claims與implicit fallback，並建立Slice 2
  handoff。
- Slice 2只在inventory-driven common pipeline可直接表達時進行bounded migration，不建立compatibility renderer、舊
  lifecycle branch或額外selector。
- 遷移後的profile必須先通過和canonical profile相同的parse、schema、reference、non-empty inventory及destructive-
  safety validation才能被`TESTBED`選取。未通過時仍保留source asset，但config/start/stop/reset必須fail closed。
- 不為legacy profiles執行dedicated unit regression、real VM、5GC、UE、training或result acceptance；遇到失敗不阻塞
  Slice 2／3，也不在本計畫內修復，除非問題同時影響maintained common pipeline。

Source刪除必須在implementation review中逐項證明至少符合一項：它是可重建且非authoritative的generated/cache產物；
它形成canonical implicit fallback或安全旁路；它具體阻擋common pipeline adaptation；或它是已有語意替代且沒有其他
consumer的重複項目。「canonical未使用」、「看似obsolete」或「舊flow不驗收」單獨都不是刪除理由。

## 4. Hash disposition inventory

判斷原則是：canonical／common path上的trusted local provenance與重複digest預設移除；只有直接服務external
supply-chain、唯一selected／active runtime identity、Docker-native identity、component-native contract，或既有optional
subsystem中有直接consumer的build-artifact reuse contract才保留。已隔離legacy flow內的hash若不增加canonical runtime
負擔、不形成fallback或安全旁路，原樣保留，不為了repository-global歸零而改寫舊toolchain。`keep`表示保留該boundary，
不表示保留目前所有重複欄位或顯示方式。

| ID | Mechanism／主要 occurrence | Producer → consumer | 原本保障 | Disposition 與理由 | 必要 replacement／限制 |
| --- | --- | --- | --- | --- | --- |
| H1 | Go archive `provisioning.lock.yaml: go.sha256` | provisioning lock → `scripts/guest/common.sh` download check | 防止外部下載損壞或被替換 | **keep**；這是明確的 external transport／supply-chain boundary，版本字串與解壓成功不能等價證明下載 artifact 身分 | 只在 download boundary 保留一份 authoritative checksum；generated receipt 不再複製 expected checksum |
| H2 | MongoDB signing-key fingerprint | provisioning lock → `scripts/guest/core.sh` staged／installed key check | 防止信任錯誤 repository signing key | **keep**；fingerprint 定義實際 apt trust anchor，沒有其他等價 identity check | authoritative value只留在 provisioning lock；manifest可記錄 resolved package／source，不複製 fingerprint |
| H3 | Docker base image digest與Torch wheel URL `#sha256` | Dockerfile pin → Docker／Python installer | 固定外部 base image與wheel bytes | **keep**；兩者都是 package transport／supply-chain contract | 不再由 testbed 另算 expected digest；沿用 native installer verification |
| H4 | canonical generated config tree digest：`config_hash.py`、Guest `active.sha256`、Host selected comparison、Compose `io.5g-nwdaf.config-hash` | selected `CONFIG_DIR` → Guest activation marker／actual tree與Host container labels → status／stop／reset guards | 偵測 wrong-config、partial activation與selected／active runtime mismatch | **keep and consolidate**；這是唯一必要的跨Guest／Host runtime identity safety boundary，移除會弱化 destructive safety | digest只涵蓋 execution-relevant generated artifacts／inventory；不得巢狀加入H7–H10 provenance hash，所有runtime consumer共用同一值 |
| H5 | Docker image ID與OCI revision observation in `ml-status.py` | Docker daemon／image labels → status與run metadata | 辨識實際執行 image，避免把 stale image 當成目前 source | **keep**；是Docker-native immutable identity／直接runtime evidence，不是testbed自建expected hash chain | 只觀測native identity並和selected image／revision規則比對，不複製成另一expected digest |
| H6 | PyMTLF `artifact_key`／`candidateDigest`與seed import result comparison | PyMTLF artifact repository／protocol → native config、runtime import、status／evidence | 定址模型artifact、驗證seed restoration與protocol publication identity | **keep at component boundary**；欄位由PyMTLF contract定義，testbed不能改用自訂ID | Testbed只傳遞或以component-native tooling驗證；legacy `fl-control.py`可繼續顯示，不代表testbed另建hash contract |
| H7 | canonical config manifest的`scenario.definitionHash` | renderer → `resolve_config_scenario`及config checker | 嘗試偵測scenario source在render後變更 | **remove from canonical／common config contract**；scenario是trusted local input，此hash讓既有generated config受mutable source牽連，且和schema、path、name／kind及generated runtime semantics重複。Legacy dataset spec內同名欄位歸H10，不在此移除 | Scenario先做schema與reference validation；execution-relevant值render進`CONFIG_DIR`，deployed identity由H4保護，run provenance記source path與repository revision |
| H8 | `topology.definitionHash`與`generated.definitionHash` | renderer → config checker | 嘗試證明selected `TESTBED`和render時相同 | **remove**；同一topology被重複hash兩次，仍不能取代generated inventory的語意正確性 | Checker由selected `TESTBED`重建machine／service／container／network／reset inventory並和manifest exact compare；H4保護active bytes |
| H9 | `generated.baselineHash`與`generated.generatorSourceHash` | renderer source／`config/default` tree → config checker | 讓baseline或renderer變動時舊config失效 | **remove**；generator source不是runtime contract，helper重構不應使有效generated config失效，且hash不證明render語意正確 | 保留generated file inventory與native/schema validation；provenance記repository revision／dirty flag，behavior由focused tests證明 |
| H10 | Legacy dataset graph：`profileHash`、scenario `definitionHash`、`generatorSourceHash`、`datasetSetId`、Parquet `sha256` | dataset resolver／generator → cache path、Guest activation與audit | content-addressed reuse、source drift與staged Parquet byte comparison | **retain inside isolated legacy flow**；它們有現存producer／consumer，移除會迫使重寫不納入驗收的PseudoDriver toolchain，且隔離後不增加canonical MNIST負擔 | Canonical manifest、renderer、preflight與dataset path不得引用H10；Slice 2另以seed、sample indices、class／shape／count、train-validation-test disjointness、path containment、read-only mount與native loader驗證MNIST，不新建identity hash |
| H11 | Host→Guest runtime-tools archive hash、remote filename與`runtime-tools.sha256` | `guest-tools-sync.sh` → remote pre-extract check／installed receipt | 嘗試偵測trusted local upload損壞並記錄tools source | **remove**；upload走Vagrant SSH transport，gzip/tar本身會拒絕損壞archive，installed receipt沒有consumer，額外SHA只重複transport並增加同步狀態 | 保留explicit file allowlist、strict tar exit、temporary staging、machine-role check、required-file／mode驗證與fail-closed install；remote temp name不使用content hash |
| H12 | `network-config.py: fragmentSha256` | network plan renderer → 無consumer | 原意應為network fragment identity | **remove**；repository trace只有producer，沒有任何validator、activation或recovery consumer，因此目前沒有實際保障 | 保留YAML parse、address/device ownership、desired／previous／stale exact set與apply result驗證 |
| H13 | Subscriber fixture hash | subscriber files → log與remote temporary filename | 為legacy subscriber upload建立可重現的temporary identity與操作log識別 | **retain inside isolated legacy flow**；它有直接producer／filename consumer，刪除只會改寫不納入驗收的舊command，對canonical path沒有收益 | Canonical selected inventory與runtime-tools allowlist不得引用subscriber upload；若未來正式維護此flow，再獨立判斷是否以run-scoped temporary name取代 |
| H14 | WebConsole helper/build identity與Host→Guest archive checksum | source revision／helper／tool versions → Guest current artifact reuse；archive → Guest pre-extract | 避免重用不同source／helper／toolchain的artifact，並在build前核對uploaded archive | **split with L6**；保留build identity，因為它直接決定既有artifact能否安全重用；移除archive checksum與content-derived remote filename，因為trusted local upload、strict tar extraction及完整build-output checks已涵蓋必要failure | WebConsole subsystem、build identity與operator contract保留；archive改用run-scoped temporary name，任何upload、extract或build失敗都fail closed；identity不混入canonical H4或experiment evidence |
| H15 | CPU smoke `sourceConfigHash` | test helper → generated smoke manifest | 記錄測試輸入config tree | **remove field only**；只有producer、沒有consumer，不能驗證或保護任何test behavior；這不要求刪除CPU smoke helper或container test | 保留測試時使用fixture名稱、實際generated config identity與image revision；是否繼續執行legacy smoke不屬於本hash disposition |
| H16 | provisioning manifest `lock.sha256`、duplicated `archiveSha256`與copied signing-key fingerprint | provisioning lock → informational Guest receipt | 記錄整份lock與expected external pins | **remove duplicated receipt fields**；direct trace沒有runtime consumer，且把authoritative external values複製進receipt形成第二份provenance chain | Receipt保留requested／resolved version、platform、package source與drift；external verification仍由H1–H2的authoritative lock完成 |
| H17 | Unused `configlib.sha256_file`與Host `sha256sum` preflight dependency | 無consumer helper；preflight → 保留的H13／H14 | helper沒有保障；Host dependency支援仍存在的local hash flows | **split**；移除無consumer的`configlib.sha256_file`，但H13與H14 build identity仍直接使用Host `sha256sum`，因此preflight dependency保留。不能在consumer尚存時宣稱dependency已orphan | H11與H14 archive checksum移除後，以repository search確認H13及H14 build identity consumers仍閉合；Guest H1的`sha256sum`屬另一環境，不受此處理影響 |

Git commit／submodule revision仍作為source與artifact provenance保留，但不包進H4 runtime config digest的巢狀local
provenance。現有submodule declarations、component source與lock dependency依L7保留；canonical selected inventory只
解析NRF、ADRF、NWDAF、PyMTLF與實際需要的persistence revision。現有
`generated.generatorRevision`不再放在H4涵蓋的execution config manifest；正式run由
Slice 3 `metadata.json`直接記錄repository revision與dirty flag。

## 5. Implementation plan

Slice 1 依下列順序實作；每一波完成focused tests後才進下一波，避免同時刪除producer與consumer而失去trace：

1. **Common-safety characterization**：先建立／保留H1–H6與provider guard、empty inventory、wrong-config、partial
   activation、unexpected runtime、exact stop／reset的deterministic tests；所有provider tests只用synthetic fixtures。
2. **Canonical／common hash simplification**：按H7–H17逐項套用remove、split或keep disposition，只移除表中明確位於
   canonical／common path或已無consumer的fields、producers、consumers、docs與tests；H10、H13等isolated legacy hash
   原樣保留。移除後以repository search確認沒有orphan field、helper、CLI參數或receipt，且legacy hash沒有回流canonical
   manifest／preflight／experiment evidence。
3. **Legacy classification與canonical isolation**：按L1–L8保留production Flat、static Flat／Hierarchical definitions及
   既有runtime assets；移除implicit default／fallback與canonical reachability，不刪除僅因新實驗未使用的source。
   Canonical selected dependency只包含NRF、ADRF、NWDAF、PyMTLF與persistence；legacy dependencies與WebConsole source
   仍留在repository，但不納入canonical inventory、capacity、build或acceptance。
4. **Common lifecycle adaptation handoff**：按L9辨識canonical／common reachability與dormant legacy branches。Slice 1只
   關閉安全旁路與fallback；inventory-driven machine／service adaptation由Slice 2完成。不得為了repository-global清理
   刪除full-core／RAN／UPF branches，也不得複製helper或先建立Slice 2 topology branch。
5. **Operator surface與transition guard**：在`5G_NWDAF_Infrastructure/README.md`建立唯一的legacy asset inventory，
   說明每組保留資產的`legacy`／`unverified`狀態、用途與無執行保證；不要求逐一改寫help、既有docs、tests或source
   config。在Slice 2 canonical definitions尚未存在時，缺少明確selection的deployment lifecycle必須fail closed，不得
   回退至`config/default`或舊production `testbed.yaml`；明確選取legacy profile仍須通過當下common static safety。
6. **Inventory closure**：維護兩張inventory的source-to-consumer mapping、保留legacy asset清單，以及canonical／common
   path與dormant legacy-only machine／Path hard-code allowlist；每個實際刪除項目附上3.1節所要求的理由。除L11與3.1節
   明確交給Slice 2或隔離保留的項目外，不得留下未分類legacy reference或hash producer／consumer。

## 6. Focused verification

- Repository search對H1–H17逐項核對：canonical／common removed項目沒有producer、consumer、field、CLI、receipt、test
  或current docs；kept項目只存在於表列boundary，H4沒有第二套canonical config identity；H10／H13只留在隔離legacy
  reachability，不被新renderer、preflight或campaign使用。
- Synthetic config／lifecycle fixtures仍能偵測wrong-config、partial activation、Host runtime identity mismatch、empty
  inventory、unexpected unit／container／volume與destructive scope mismatch。
- External provisioning tests確認Go archive checksum及MongoDB key fingerprint仍由authoritative lock驗證；Dockerfile仍
  使用digest-pinned base image與wheel checksum。
- PyMTLF native loader／artifact tooling的non-starting test確認artifact key仍走component contract；不啟動container或
  修改component source。
- WebConsole source、config、build／lifecycle targets與既有tests仍存在；H14 build identity的producer／consumer保持閉合，
  archive checksum與content-derived remote filename已移除，strict upload／tar／build failure checks仍通過。Canonical rendered
  config明確disabled，且H14沒有混入H4或canonical experiment evidence。
- Repository搜尋與README review確認production／static definitions、examples、dataset／subscriber／control tooling、
  component／provisioning source及dormant lifecycle branches均由README legacy asset inventory以清楚群組涵蓋，且
  canonical path沒有implicit default selection或安全旁路；help、既有docs、tests、YAML、manifest及source config不重複
  支援狀態metadata。三機／Path A／B hard-code在canonical／common reachable path必須為零，isolated legacy-only occurrence
  列入L11 allowlist，不要求repository-global歸零。
- Deleted-file review逐項確認刪除對象符合3.1 removal gate；不存在以「canonical未使用」或「legacy不驗收」為唯一理由的
  source、script、target、test、submodule、lock row或config刪除。
- Transitional fail-closed test確認缺少明確`TESTBED`／effective `CONFIG_DIR`時，config、start、stop與reset不會使用
  stale default或擴大scope；另確認明確legacy selection不會被冒充為canonical acceptance。
- Provider guard測試只使用synthetic fixture／mock，不在sandbox內啟動real Vagrant／VirtualBox process。本slice不要求
  real provider、VM、container或experiment evidence。

## 7. Acceptance criteria

- Slice 2 可接續使用的common substrate不依賴任何已移除hash或implicit legacy fallback；isolated legacy branches可保留，
  但不在canonical reachability內。Slice 1 checkpoint本身不宣稱新topology已實作或可執行。
- H1–H17全部和source一致；保留項目有直接contract／safety理由，移除項目的必要保障已有semantic／lifecycle替代，
  且沒有open或unclassified occurrence。
- L1–L10全部完成分類：production Flat、static Flat／Hierarchical的既有source與operator assets有明確保留清單；
  canonical selected path已和legacy runtime隔離。任何實際刪除均符合3.1 removal gate；L11及inventory-driven
  adaptation／bounded migration留給Slice 2。
- README legacy asset inventory完整涵蓋所有保留資產及其migration disposition，並明確說明`legacy`／`unverified`、
  無執行保證與無專屬維護；repository不另建machine-readable support-status contract。
- Legacy assets不作為implicit default，也不被canonical operator path選取；若明確經common lifecycle選取，未通過
  common static safety時無法進入provider或destructive lifecycle。不要求它們通過dedicated regression或real-environment
  驗證，保留既有tests也不代表必須讓它們通過。
- WebConsole optional subsystem及既有operator contract保留，canonical protocol-driven profile不啟用；本Slice不要求
  WebConsole adaptation或real-environment驗證。
- Common lifecycle safety及transitional fail-closed tests通過；不要求舊Flat／static experiment regression或real
  environment驗證通過。
- 完成mandatory initial review、user-review handoff與獨立commit proposal／approval後，才進入Slice 2 implementation。

## 8. Review handoff requirements

Slice 1 implementation交付review時必須列出：

- `5G_NWDAF_Infrastructure` 的完整changed／deleted file inventory；每個deleted file另附consumer trace及符合3.1 removal
  gate的具體理由，不能用「新實驗未使用」概括；
- H1–H17每一項的production path、test evidence與remaining occurrence；
- L1–L10的retain／isolate／remove evidence、canonical reachability trace、`5G_NWDAF_Infrastructure/README.md`中的
  preserved legacy asset inventory、L6 WebConsole retention evidence，以及L11 canonical／common與legacy-only
  machine／Path hard-code分類及migration handoff；
- preserved common safety tests、transitional fail-closed result與未執行的real-environment evidence；
- pre-existing `ML/PyMTLF`、`NFs/nwdaf` pointer差異的處置，並與本slice實作diff分開說明。

在使用者確認review前，所有implementation changes保持unstaged／uncommitted，本文件狀態保持open。
