# Slice 3 Dual-dataset Branch Replacement Acceptance Detailed Plan

日期：2026-09-11

狀態：Ready for User Review

上層計畫：

- [Multi-host Branch Replacement Experiment Plan](../multi-host-branch-replacement-experiment-plan.md)
- [5G NWDAF Infrastructure Development Policy](../../../development_policy.md)

本文件細化 Slice 3 的 GPU execution、fault controller、natural replacement timing、structured evidence、離線分析及兩個
real flow runs。跨-slice 的四機 topology、protocol ownership、legacy disposition、provider safety、review 與 completion
規則仍以上層主計畫為準。

---

## 1. Operator-visible outcome

Operator 先用既有 `config-create` 明確選取 MNIST 或 CIFAR-10 的 Branch replacement scenario，並以 `DEVICE=gpu`
產生同一 canonical `CONFIG_DIR`。四台 VM 已啟動且 config 已 stage／activate 後，以一個 domain-specific experiment
runner 執行完整 flow：

1. 驗證 selected／active topology、GPU、dataset、clock、Guest／Host process 與 evidence directory；
2. 啟動 canonical Guest services與十一個 Host PyMTLF containers；
3. 觸發 protocol-driven training；
4. 完成 2 個 normal accepted rounds；
5. fail-stop Area A primary Branch 的 exact Go NWDAF 與 PyMTLF container；
6. Root在背景自然進行replacement preparation，runner觀測實際accepted degraded round數；
7. 由 Root 依 candidate priority 選出 replacement；
8. replacement 首次成功貢獻後，在固定8 accepted rounds內至少完成1個restored round；
9. 執行獨立 held-out evaluation，將必要evidence整合成`events.jsonl`與`run.json`，再停止本次 processes。

Runner 的 terminal output 只顯示里程碑與低頻 compact heartbeat。必要structured events與run結果直接寫入ignored
`runs/`；journald／Docker logs只在需要診斷時保存bounded範圍，不長時間串流進terminal。MNIST與CIFAR-10各執行一次；兩個run
都必須完成8 accepted rounds，在2 normal後注入fault，保存實際degraded count並至少完成1 restored round；不能用
單輪聚合、replacement process 成功啟動或兩個
dataset 合併結果代替。

預定operator sequence維持明確scenario切換，不保存額外current-selection state：

```bash
make config-create \
  FROM=experiments/protocol-hierarchical/branch-replacement/mnist/scenario.yaml \
  DEVICE=gpu FORCE=true
make vm-up
make fl-branch-replacement-run RUN_ID=<uuid-v4>
```

CIFAR-10 run只在前一run已stop、evidence已保存且exact reset完成後，將`FROM`換成
`experiments/protocol-hierarchical/branch-replacement/cifar10/scenario.yaml`重新render，再使用相同run target與新的
UUID。Canonical `TESTBED`與effective `CONFIG_DIR`維持預設，因此後續commands不需重打長路徑。

## 2. 盤點基線與直接證據

### 2.1 Repository 與 revision 基線

本次 read-only inventory 的 current revision 如下；implementation 開始前仍須重新檢查 working tree，不能把此表當成
未來 run 的 actual revision evidence。

| Repository | Branch／revision | Slice 3 disposition |
| --- | --- | --- |
| `5G_NWDAF_Infrastructure` | `feat/hierarchical-fl-protocol-extension` / `8495f4a` | 唯一 implementation repository |
| `ML/PyMTLF` | `feat/hierarchical-fl-protocol-extension` / `579f737` | GPU training、evaluation、JSONL 與 replacement runtime；預設 read-only |
| `NFs/nwdaf` | `feat/hierarchical-fl-protocol-extension` / `be3fa57` | Guest protocol transport 與 exact process target；預設 read-only |
| `NFs/nrf` | `feat/r18-nwdaf-discovery` / `0dd4024` | priority candidate fresh exact-ID discovery；預設 read-only |
| `NFs/adrf` | `feat/r18-federated-learning` / `905f059` | global model distribution與temporary records；預設 read-only |
| `nwdaf-resources` | `feat/hierarchical-fl-protocol-extension` / `e2b21e1` | 已跑通的replacement／evidence語意參考；不搬移其runtime owner |
| `nwdaf-docs` | `main` / `f88941b` | component contract與event schema的read-only source |
| `testbed-docs` | `main` / `93b6b07` | 本計畫與後續review／record owner |

Slice 2 checkpoint 後，四台 selected VMs 已 power off、十一個 containers 已停止、experiment state 已 reset。Slice 3
從同一四機 definition 與乾淨 runtime scope重新啟動，不 destroy／rebuild VM；若實作或preflight顯示 box、disk或active
identity 已不一致，才依既有 exact-scope lifecycle 回報處理，不自動清除。

### 2.2 已確認可重用的 topology 與 component behavior

- Canonical `TESTBED` 已描述四台 VM、十一個 NWDAF↔PyMTLF one-to-one mappings、Area A primary／replacement、Area
  B／C surviving regions、NRF、ADRF、MongoDB、ports、volumes 與 reset scope。
- Area A candidate pool 已使用 `selection_method: priority`；primary priority `100`，replacement priority `50`。Root
  擁有 failure eviction、fresh exact-ID resolve 與 replacement selection。
- Root completion policy 已允許 3 個 regions 中 2 個成功時接受 degraded round；每個 Branch 的兩個 Leaves仍要求
  完整成功。
- PyMTLF 已輸出 `MODEL_EVALUATION`、`ROOT_ROUND_OUTCOME`、`BRANCH_FAILURE_DETECTED` 與
  `BRANCH_REPLACEMENT_READY` node-local JSONL；controller 只需另記錄實際 stop time。
- `BRANCH_REPLACEMENT_READY` 表示新的 Branch subscription 與 downstream preparation 已完成，不表示它已對某輪
  成功貢獻；首次貢獻只能由後續 accepted `ROOT_ROUND_OUTCOME.successfulNfInstanceIds`判定。
- Root在一個round完成後才處理failed Branch，並以背景future啟動replacement preparation；下一輪只要剩餘active
  Branch數符合policy就會立即進行，不等待replacement ready。Replacement若在一輪進行中ready，只會加入下一個尚未
  dispatch的cohort。
- Current generated settings將preparation與round deadline都設為300秒。PyMTLF FL Server在preparation request中傳送
  `timeAvReq: PT300S`與`mLTrainRepInfo.maxResTime: 300`，round PATCH也傳送
  `mLTrainRepInfo.maxResTime: 300`，並在本地依相同configured deadline等待callback；PyMTLF FL Client可透過既有
  delay notification在bounded policy內請求extension。Go NWDAF負責傳送request與callback，不擁有FL countdown。
- PyMTLF 已提供 `tools/evaluate_image_model.py`，可用 final Root artifact、獨立 held-out `.npz` 與 image client config
  產生 machine-readable accuracy result。

### 2.3 GPU 環境盤點

2026-09-11 的 Host 直接觀測：

| Item | Observation |
| --- | --- |
| GPU | one `NVIDIA GeForce RTX 3080` |
| GPU UUID | `GPU-05edb56c-f554-413d-4cf3-b92e8e85a42b` |
| Driver | `535.183.01` |
| Memory | 10,240 MiB total；盤點時 9,138 MiB free |
| Compute mode | `Default` |
| CDI | `nvidia.com/gpu=0`、UUID selector、`nvidia.com/gpu=all` 可見 |
| Docker runtime | `nvidia` runtime 已註冊 |
| PyMTLF image | `5g-nwdaf-infrastructure/pymtlf:local`，size約5.42 GB，OCI revision `579f7375…` |
| Disposable CUDA probe | `torch 2.5.1+cu121`、`cuda_available=True`、one RTX 3080 visible |

PyMTLF source已直接確認：Linux x86_64 image安裝CUDA 12.1 PyTorch wheel；device resolver接受`cpu`、`cuda`與
`cuda:N`並在CUDA不存在時fail closed；Leaf local model／tensor與Root validation evaluation都會移到configured
device。MNIST與CIFAR-10 seed model分別約78 KiB與81 KiB，兩者皆是小型CNN；每個Leaf目前只有100個training
samples、batch size 16、每輪local epoch 1。

這些資料證明目前image與runtime可看見GPU，但不等於七個process並行時的容量evidence。正式run前仍須啟動完整
selected GPU inventory並確認七個CUDA participants皆ready、沒有OOM／device fallback，且實際GPU memory狀態符合
本文件的admission contract。

### 2.4 現有 testbed gaps 與處置

| Gap | Direct evidence | Slice 3 disposition |
| --- | --- | --- |
| Protocol scenario固定CPU與2 rounds | 兩份current image scenarios有`training.device: cpu`及`acceptedRounds: 2` | 保留normal smoke scenarios；另建兩份replacement scenarios，rounds固定8，device ownership移回`TESTBED` |
| Renderer hard-code protocol Root／Leaf為CPU | `render_protocol_pymtlf()`直接寫入`cpu` | 由resolved per-service device產生Root validation與Leaf training config |
| Canonical TESTBED十一個services均為CPU | `mlRuntime.services.*.device: cpu` | Root與六Leaves設為accelerator-capable `cuda:0`；四Branches保持CPU |
| `DEVICE=gpu`目前無法改protocol profile | override只轉換source中非CPU services | 以TESTBED標記七個accelerator participants，`DEVICE=gpu`保留CUDA，`DEVICE=cpu`提供整體fallback |
| Scenario與TESTBED重複device truth | scenario與service definition都保存device | replacement與normal image scenario都移除`training.device`；scenario只擁有run behavior |
| GPU capacity只計participant數與單一free-memory floor | current preflight不估七個CUDA contexts的實際占用 | 設8,192 MiB pre-start free floor，並增加seven-participant real startup／CUDA visibility evidence |
| 無multi-host replacement runner | current `fl-control.py`只支援training start／status | 新增一個domain-specific runner，複用既有config、identity、lifecycle與FL control，不建立第二套renderer |
| 人為控制replacement preparation會改變自然恢復timing | Root已在背景準備replacement，且request／callback有bounded deadline | 不pause、blackhole或代理request；保留direct production path並觀測實際degraded round數 |
| 無testbed final held-out command | dataset已有held-out，component已有evaluator | Runner從exact terminal artifact identity取出Root artifact，以同image執行GPU held-out evaluation |
| 現有status偏向log-derived milestones | protocol mode不解析legacylog，但尚無完整evidence collector | Runner增量讀Root JSONL；一般logs只在failure、timeout或特定protocol問題時按需保存bounded範圍 |

## 3. 固定 Slice 3 contract

### 3.1 Run 數量與 training workload

新增`experiments/protocol-hierarchical/branch-replacement/mnist/scenario.yaml`與
`experiments/protocol-hierarchical/branch-replacement/cifar10/scenario.yaml`。兩者沿用Slice 2已驗證的dataset split與
模型contract：seed `42`、每Leaf 100 training samples、Root 200 validation samples、200 held-out samples、batch size
16、learning rate `0.001`、Leaf local epoch 1。唯一必要的run長度變更是`acceptedRounds: 8`。

每份scenario另外保存run behavior，而不重複topology identity或device：

```yaml
fault:
  mode: branch-replacement
  branchGroup: area-a
  normalAcceptedRounds: 2
  restoredAcceptedRounds: 1
observation:
  pollIntervalMilliseconds: 250
  heartbeatSeconds: 30
```

`branchGroup`只指定這次在哪個logical group注入fault；controller必須從selected topology解析該組目前最高priority的
active candidate作為stop target，不能在scenario手寫primary／replacement NF Instance ID。Replacement candidate仍由
Root依priority選擇，controller不得從scenario推導或傳送selection結果。

`acceptedRounds`必須為8，`normalAcceptedRounds`必須為2，`restoredAcceptedRounds`必須為1；scenario不保存
`degradedAcceptedRounds`，該數量由primary stop、Root outcomes與replacement first contribution推導。欄位缺少、
不合法或round budget無法容納2 normal加1 restored時，config validation應fail closed。Internal polling可以250 ms以
避免GPU round太快而錯過in-flight barrier；terminal只每30秒輸出heartbeat，兩者不是同一頻率。

### 3.2 GPU ownership 與混合配置

GPU assignment屬於physical/runtime placement，因此authoritative owner是selected `TESTBED`的
`mlRuntime.services.*.device`。Scenario不得保存device。Slice 3固定使用：

| PyMTLF role | Count | Resolved device with `DEVICE=gpu` | 理由 |
| --- | ---: | --- | --- |
| Root | 1 | `cuda:0` | 執行initial與每個accepted global model validation；terminal後runner另以同GPU執行held-out evaluation |
| Branches | 4 | `cpu` | 只做protocol orchestration與parameter aggregation，沒有local image shard；不建立無用CUDA context |
| Leaves | 6 | `cuda:0` | 執行local image training |
| **GPU participants** | **7** | one shared RTX 3080 | 由manifest與actual runtime exact compare |

`DEVICE=gpu`只保留TESTBED中accelerator-capable services為`cuda:0`；`DEVICE=cpu`可把同七個services coordinated
override為CPU，供診斷或未來explicit fallback，但不能拿CPU run充當本Slice acceptance。Make global default暫不改成GPU，
避免改變legacy／normal operator default；本Slice的config-create與run metadata必須明確記錄`DEVICE=gpu`，後續commands
從generated manifest取得resolved mapping，不需重打device。

`hostSafety.minimumGpuMemoryMiB`由0改為8,192。此值是start前的共享GPU admission floor，不表示每個participant各需要
8 GiB，也不宣稱等於實際用量。Preflight必須同時確認one selected GPU、CDI、Docker runtime、CUDA image probe與七個
configured participants；startup後再逐container確認configured `cuda:0`沒有CPU fallback，並保存一次actual
`nvidia-smi` snapshot。若七個process實際training出現OOM或顯著資源競爭，屬capacity blocker，不能把某些Leaves改回CPU
後仍宣稱GPU acceptance。

### 3.3 Natural replacement timing與deadline

本Slice不控制replacement preparation的開始、停留或放行時間。Primary failure被Root接受後，Root沿用目前background
replacement future，透過原本的Go NWDAF↔PyMTLF endpoint立即準備下一順位candidate；runner只觀測
`BRANCH_REPLACEMENT_READY`與first contribution，不插入HTTP gate、proxy、額外port、sleep、pause或network
blackhole。

目前selected TESTBED的`preparationTimeoutSeconds`與`roundTimeoutSeconds`皆為300。Renderer產生相同的PyMTLF
server deadlines；preparation request以`timeAvReq`與`mLTrainRepInfo.maxResTime`告知Client，round PATCH以
`mLTrainRepInfo.maxResTime`告知Client，而Server另以本地monotonic deadline等待callback。這些既有production
deadlines保持不變，並在run metadata記錄effective values。若Client發送delay notification，既有bounded extension policy
照常處理；controller不得代替Client要求extension。

Go NWDAF的同步HTTP timeout和上述FL callback deadline是不同邊界：目前generated
`mtlfBackend.requestTimeout`為120秒，而peer ML Model Training request另有30秒HTTP timeout；它們只限制request
建立或PATCH的同步傳輸。Request成功建立後，training結果仍經非同步callback，適用300秒FL deadline。

若replacement在下一個round dispatch前ready，該round即可使用replacement；若在round進行中才ready，因cohort已凍結，
只能從再下一輪加入。兩者都是有效production outcome，實際degraded count由structured events推導。

### 3.4 Fault ownership

Controller只停止Area A primary的兩個exact runtime owners：

- Guest `nwdaf-branch-a-primary` systemd unit；
- Host `pymtlf-branch-a-primary` Compose service／container。

Target必須由selected manifest、`branchGroup: area-a`與current priority ordering解析，並在stop前核對active identity。
Guest stop必須經既有approved host-context provider／SSH guard；任何real Vagrant／VirtualBox command都不能在sandbox內
執行。Controller只有在兩個process都confirmed stopped後，才寫入`BRANCH_PROCESS_STOPPED`，其`recordedAt`必須是
兩者皆停止後的UTC RFC3339 timestamp。任一stop ambiguous、target mismatch或postcondition失敗時，不得操作其他
candidate或擴大stop scope，run標為failed並保存evidence。

## 4. End-to-end execution flow

### 4.1 準備與admission

1. 檢查affected repositories的actual revisions、dirty flags與component image identity。
2. 以選定replacement scenario執行既有`config-create DEVICE=gpu`；不直接修改generated config。
3. 執行common `config-validate`、dataset generate／validate及component-native config／topology loader。
4. 驗證resolved device map為Root + six Leaves CUDA、four Branches CPU，Compose runtime／CDI設定與native config一致。
5. 驗證GPU total／free memory、driver、CDI、Docker NVIDIA runtime、PyTorch CUDA probe與selected ports ownership。
6. 在approved host context啟動或核對四台VM，完成stage／activate、clock skew與selected／active identity checks。
7. 建立exclusive run directory；同一selected config只能有一個active replacement runner，既有run directory不得覆寫。

### 4.2 Start 與normal phase

1. 透過existing lifecycle啟動Guest services與十一個PyMTLF containers；所有services維持selected direct endpoints。
2. 執行backend health、NRF registrations、ADRF health、eleven one-to-one mappings、seven CUDA visibility及unexpected
   runtime checks。
3. 初始化`run.json`並把initial inventory／GPU snapshot寫入`events.jsonl`後，透過既有private training request建立run。
4. 從training status取得`planId`／`mlCorreId`，並開始以byte cursor增量讀Root node-local JSONL；解析資料不直接印出。
5. 只有accepted `ROOT_ROUND_OUTCOME`可增加normal counter；rejected attempt不算round，也不得有`ROOT_GLOBAL`
   evaluation。
6. 等待exactly 2個normal accepted rounds及下一round進入dispatch／waiting。若第3個accepted outcome在stop前已完成，
   代表controller錯過barrier，run必須失敗，不可重新標記phase。

### 4.3 Fault 與degraded phase

1. Stop並確認Area A primary Go NWDAF與PyMTLF；寫入controller event。
2. 等待Root accepted outcome同時包含primary於failed set、Area B／C於successful set，並確認
   `BRANCH_FAILURE_DETECTED`。
3. Root依production path立即在背景準備replacement；runner不阻塞或放行request。
4. 從第2步確認primary失敗且Area B／C成功的accepted outcome起，到replacement首次成功貢獻之前，將只由surviving
   regions完成的accepted outcomes分類為degraded，保存每輪selected／successful／failed identities。除了必要的
   failure-detection outcome外，不預先設定目標數量。
5. Rejected attempts另行保存，但不計入8 accepted rounds或任一phase count。

### 4.4 Replacement 與restored phase

1. 等待`BRANCH_REPLACEMENT_READY`，核對failed identity為primary、replacement identity是Root依priority選出的Area A
   next candidate。
2. Ready event只代表preparation完成；等待replacement第一次出現在後續accepted
   `successfulNfInstanceIds`，另記first-contribution timestamp與round。
3. First contribution所在round計為第一個restored round；accepted outcome必須同時有Area A replacement、Area B與C，
   且不得再含primary。
4. 至少完成1個restored accepted round，總accepted rounds必須為8；Root training resource最後必須`COMPLETE`。
5. 若replacement加入已dispatch的舊cohort、priority選錯、8 accepted rounds內沒有restored round、ready後無成功貢獻或
   old primary重新出現，run失敗。

### 4.5 Final evaluation、collection 與shutdown

1. 用terminal status的Root NF identity、round與component-nativeartifact identity在Root volume解析exactly one final artifact。
2. 將final artifact複製到run evidence，使用相同PyMTLF image、GPU runtime、Leaf image workload config及該dataset的獨立
   held-out `.npz`執行`evaluate_image_model.py`；disposable evaluation container使用同一GPU但不和七個training
   participants並行，且必須由runner命名、記錄並`--rm`清除。
3. 在停止或reset前，將Root與controller的必要structured records、直接需要的participant records、runtime／GPU snapshots
   及held-out evaluation依時間寫入`events.jsonl`；不逐一複製所有node JSONL。
4. 以已收集events更新`run.json`的metadata、phase timeline、latency、held-out result與terminal state；CSV與plot留給
   實驗後的離線分析，不屬於runner輸出。
5. 透過existing exact-scope lifecycle停止仍在執行的selected Guest／Host processes；failed primary已停止是合法差異，但
   其他missing／unexpected runtime仍須明確報告。
6. 將cleanup outcome寫入events並finalize `run.json`；two-file schema／consistency checker通過後才把run標為successful。
   四台VM、datasets、images與run evidence預設保留；兩個dataset run之間使用approved exact reset清除NRF／ADRF／volume
   experiment state並驗證seed restoration，不destroy VMs。

## 5. Observation 與 evidence contract

### 5.1 Authoritative observations

| Question | Authoritative source | Interpretation |
| --- | --- | --- |
| Round是否accepted及cohort | Root `ROOT_ROUND_OUTCOME` JSONL | `accepted=true`才計入phase；selected／successful／failed IDs直接使用record |
| Per-round loss／accuracy | Root `MODEL_EVALUATION` JSONL | `ROOT_INITIAL`為起點；每個accepted round應有exactly one `ROOT_GLOBAL` |
| Failure detection | Root `BRANCH_FAILURE_DETECTED` JSONL | 必須對應exact stopped primary |
| Replacement ready | Root `BRANCH_REPLACEMENT_READY` JSONL | preparation ready，不等於成功貢獻 |
| Primary實際停止時間 | Controller `BRANCH_PROCESS_STOPPED` JSONL | 兩個process confirmed stopped後才寫入 |
| First replacement contribution | 第一筆含replacement ID的accepted `ROOT_ROUND_OUTCOME` | 不由ready event或一般log推測 |
| Training terminal artifact | Training status resource + component-native artifact identity | exact定位Root final model；不建立testbed-owned digest |
| Final held-out accuracy | `evaluate_image_model.py` JSON output | 和per-round validation分開 |
| Candidate priority與identity | Selected topology + NRF exact-ID profile + Root events | controller不建立替代selection truth |
| GPU execution | Generated native config、Compose runtime、container CUDA probe、Host `nvidia-smi` | 需同時證明selected intent與actual device可用 |

Resource location、`notifCorreId`、Leaf old-route retirement與ADRF temporary record lifecycle若current JSONL沒有完整欄位，
可從exact component API response或bounded owner log擷取supplemental evidence。這些log只能回答其特定protocol lifecycle
問題，不能取代round cohort或metric JSONL。

### 5.2 Run directory

每個run使用canonical UUIDv4並保存：

```text
runs/protocol-hierarchical/<dataset>/<run-id>/
├── events.jsonl
├── run.json
├── final-root-model.tar.gz
└── diagnostics/                 # optional；只保存bounded diagnostic evidence
```

`runs/`已由repository ignore；run artifacts不是implementation source，不commit到implementation repository。成功run經
review後只將必要摘要與可重建evidence索引整理到`testbed-docs/records/`，不複製所有raw logs進Git。

`diagnostics/`不是acceptance必備輸出。只有failure、timeout或回答特定protocol lifecycle問題需要一般log時才建立，並按
Guest、container或runtime status分目錄保存exact owner與bounded time window；成功且不需補充診斷的run可以沒有此目錄。

### 5.3 `events.jsonl`與`run.json` minimum

`events.jsonl`是append-only chronological evidence stream。每筆record至少保存`recordedAt`、`source`、適用時的
`nfInstanceId`、`eventType`及原始`payload`。Runner只整合acceptance直接需要的Root／controller records、必要participant
records、runtime／GPU snapshots與held-out evaluation，不複製每個node的完整JSONL，也不把一般文字log轉成round或metric
authority。

`run.json`至少保存：

- run UUID、dataset、scenario definition、start／finish timestamps、exit status；
- actual repository branch／revision／dirty flag；
- selected TESTBED、generated config identity與actual active identity；
- VM inventory、Guest units、Host containers、volumes與ports；
- PyMTLF image ID及OCI revision（Docker-native identity，只觀測、不建立expected hash chain）；
- GPU name／UUID、driver、total／free memory、CDI selector、PyTorch／CUDA version；
- resolved per-service device map與GPU participant count；
- seed、sample counts、batch size、learning rate、local epoch、accepted-round、normal／restored minima及observed
  degraded count；
- primary stop target、replacement group、effective preparation／round deadlines及delay-extension policy；
- round／phase timeline、derived replacement latencies、held-out result與terminal outcome；
- final Root artifact path、cleanup outcome及任何remaining gap。

Runner在run開始與重要milestone以atomic replace更新`run.json`，cleanup後才寫入final outcome；因此中斷時仍保留目前狀態，
但只有finalized且通過cross-file consistency check的檔案才能構成successful acceptance。

### 5.4 Two-file consistency與offline derivation rules

- `events.jsonl`保留可重建的source records，`run.json`保存最終interpretation；不得在兩者複製完整event payload形成兩份
  raw truth。
- 每個Root round attempt都能從events重建round、accepted、phase、selected／successful／failed identities及timestamp。
- Learning curve只使用`ROOT_INITIAL`與accepted `ROOT_GLOBAL` evaluations；每個accepted outcome必須exactly one global
  point，rejected attempt不得有point。
- Phase以event／cohort語意分類，不以固定wall-clock區段切割。
- Latency至少包含stop→failure detected、stop→replacement ready及stop→first contribution。
- `run.json`保存phase counts、identity checks、GPU checks、terminal state、held-out result、evidence file list與failures。
- Parser遇到missing／duplicate／out-of-order identity、invalid timestamp、NaN／infinite metric或無法完整解析JSONL時fail closed，
  不跳過壞record後宣稱成功。
- CSV與plot不是Slice acceptance output。後續可從兩份machine-readable檔案離線產生；圖表若產生，只能呈現validation
  loss／accuracy對accepted round與normal／degraded／restored phase，不平滑或跨dataset疊圖推論。

### 5.5 Quiet monitoring

- 正常執行只對Root JSONL及training status做incremental read；byte cursor與last record identity留在runner memory／run state，
  不反覆讀整份檔案。
- Milestone發生時立即輸出一行；無state change時每30秒輸出一行
  `accepted/phase/replacement-state/degraded-count/elapsed`摘要。
- Docker logs與journald不預設保存。Failure、timeout或特定protocol問題發生時，先將exact owner最近bounded tail與時間窗寫入
  `diagnostics/`；仍不足才逐步擴大，不能一次dump十一個nodes全部歷史。
- Human terminal transcript不是acceptance source；若runner stdout／stderr對診斷有必要，才保存於`diagnostics/`。

## 6. Failure、retry 與 recovery

| Failure | Required behavior |
| --- | --- |
| Config、scenario、device map或port invalid | 在VM／process mutation前fail closed |
| GPU／CDI／runtime／memory不足 | 不啟動training；保留preflight result，禁止CPU fallback充當acceptance |
| VMs未啟動或selected／active identity mismatch | 不自動destroy／recreate；回報exact mismatch |
| Missed 2-normal in-flight barrier | run failed；停止並reset後重跑，不把後續round重分類 |
| Primary只停止一個owner | 不記成功stop、不擴大target；保存partial stop狀態供exact recovery |
| Degraded round rejected | 保存attempt但不計8 accepted rounds；在overall timeout內等待後續accepted outcome |
| Replacement preparation timeout／error | 不手動指定candidate或重建resource；run failed |
| Evidence copy／parse／evaluation失敗 | training success不能升格為run success；先保存可取得raw evidence |
| Runner被中斷 | 收集best-effort bounded evidence、exact停止本次selected processes；無法確認scope時停止自動cleanup |
| Stop或reset發現unexpected runtime | 正常destructive path拒絕；需要fresh inventory與另外的使用者批准 |

Retry以整個dataset run為單位。不得從中間round續跑、編輯JSONL、重用failed run的Root artifact，或把兩次partial run拼成
一份8-round evidence。Failed run directory保留unique ID與failure summary；重新run使用新UUID。

若中斷發生在fault注入前，可在exact stop與reset後用相同generated config、不同run UUID重新啟動完整flow。Primary已
fail-stop後不支援只restart該Branch、從中間round繼續或hand back；只能先保存evidence，再依selected exact scope停止、
reset並從round 1重跑。

## 7. Implementation impact map

### 7.1 `5G_NWDAF_Infrastructure` production changes

預期修改下列既有owners；實作時若需要額外top-level directory、service、daemon、external dependency或component change，
屬decision gate：

| Owner | Planned change |
| --- | --- |
| `testbed.protocol-hierarchical.yaml` | Root／six Leaves標示CUDA；Branches保持CPU；GPU free-memory floor |
| normal MNIST／CIFAR scenarios | 移除重複的`training.device`，維持2-round normal smoke語意 |
| new Branch-replacement scenarios | dataset-specific 8-round workload、Area A fault group、2 normal／1 restored minimum、poll／heartbeat behavior |
| `scripts/host/configlib.py` | 驗證scenario fault contract、device ownership與derived runtime inventory |
| `scripts/host/config-render.py` | 由per-service device產生Root／Leaf config；保留replacement direct endpoint與configured deadlines |
| `scripts/host/config-check.py` | exact混合device、scenario phase、deadline、native config與manifest checks |
| `scripts/host/ml-compose-check.py` | 驗證GPU runtime／CDI與existing exact ports，不建立replacement proxy path |
| existing preflight／status helpers | 保存七個GPU participants與actual CUDA evidence；不另建第二套capacity path |
| one domain-specific experiment runner | incremental observation、exact stop、natural phase classification、evidence collection、analysis、held-out evaluation與cleanup |
| `Makefile`／operator docs | 提供`fl-branch-replacement-run` entrypoint與簡短操作說明；不以Slice名稱命名production command |

Runner與其support code必須以durable domain semantics命名；implementation artifact、test、function、fixture、log與runtime
label不得以`slice3`、plan title或review work item作identity。

### 7.2 Tests

- Existing config／renderer suite加入normal CPU fallback及replacement GPU mixed-device cases；不另建只檢查literal token的
  meta-test。
- Existing lifecycle／capacity suites覆蓋GPU missing、memory insufficient、port collision、wrong config、unexpected runtime
  及partial cleanup。
- 新runner是distinct owner與state machine，可建立一份domain-named behavioral test module；tests直接驅動fake JSONL、
  process owners與clock，驗證contract-defined transitions，而非只檢查檔案或helper存在。
- Runner behavioral tests至少涵蓋exact stop、failure detection、replacement於下一round前或round中ready、自然degraded
  count、insufficient restored rounds、timeout與runner shutdown。
- Evidence parser tests涵蓋8-round成功、variable degraded count、rejected attempt不計數、ready不等於contribution、
  missing evaluation、duplicate
  outcome、invalid timestamp與partial evidence。
- Provider guard只能使用synthetic fixtures；不得為測試sandbox blocking而執行real Vagrant／VBoxManage。

## 8. Existing-flow disposition

| Baseline stage | Slice 3 disposition |
| --- | --- |
| TESTBED／scenario selection | Adapted：同一selector與schema family；新增replacement scenarios，device truth只在TESTBED |
| Config generation | Adapted：同一renderer產生混合GPU與8-round fault scenario，不建平行renderer或proxy path |
| Validation | Adapted：common validation全部保留，再加入mixed device、phase與deadline checks |
| Dataset acquisition／partition | Reused without semantic change：沿用Slice 2 cache、deterministic split與native checks |
| VM／network／stage／activate | Reused without semantic change：同四VM、同provider guard與active identity |
| Guest／Host startup | Reused without semantic change：eleven one-to-one runtimes與direct endpoints不變 |
| Training trigger／status | Reused and composed：沿用private API與existing FL control contract，runner只編排 |
| Fault injection | New in this flow：exact stop兩個primary owners，controller不修改production code或priority |
| Observation | Adapted：node JSONL為authority，incremental quiet monitor與bounded diagnostic logs |
| Final evaluation | Adapted：複用component evaluator與held-out dataset，使用exact terminal artifact |
| Stop／reset／recovery | Adapted：先保存evidence，再走selected-and-active exact scope |
| Legacy／normal scenarios | Reused／isolated：normal 2-round profiles保留；legacy profiles不驗收GPU或replacement |

## 9. Verification matrix

| Requirement | Static／synthetic evidence | Required real-environment evidence |
| --- | --- | --- |
| Single source與scenario ownership | schema／render tests確認scenario無device、TESTBED擁有service device | `run.json`顯示selected scenario與resolved config一致 |
| Mixed GPU mapping | exact native／Compose／manifest checks | Root + six Leaves CUDA visible；four Branches CPU；無fallback／OOM |
| GPU capacity | missing CDI/runtime/memory fixtures | `events.jsonl`保存RTX 3080 snapshot、8,192 MiB admission與seven participants ready |
| Natural replacement timing | runner state-machine與deadline contract tests | direct endpoint、observed degraded count、configured deadline與ready timing |
| Exact primary stop | fake process owner／partial-stop tests | Area A primary Go+PyMTLF confirmed stopped，其他regions仍active |
| Priority selection | topology／manifest candidate ordering tests | Root events與NRF profile證明primary 100失效後選replacement 50 |
| 8-round phase integrity | JSONL parser/state-machine fixtures | MNIST與CIFAR-10各一個完整8-accepted-round run，2 normal後fault且至少1 restored |
| JSONL／evaluation consistency | duplicate／missing／rejected fixtures | 每個accepted outcome exactly one Root global evaluation |
| Replacement lifecycle | ready-versus-contribution fixtures | failure detected、ready及first contribution形成至少1個restored round |
| Final held-out path | evaluator command／artifact resolver tests | exactRoot final artifact在GPU上完成獨立held-out evaluation |
| Quiet monitoring | incremental cursor與bounded-log tests | terminal transcript只含milestones／heartbeat；一般logs只在需要時進入optional `diagnostics/` |
| Evidence completeness | `events.jsonl`／`run.json` schema與cross-file consistency tests | 兩份核心檔案與final Root artifact齊全；diagnostics按需保存 |
| Cleanup | partial／unexpected runtime fixtures | process stop、ADRF／NRF／volumes reset與seed restoration |

Static、mock與single-container CUDA probe都不能取代兩個real multi-host GPU runs。MNIST成功也不能替代CIFAR-10；任一run
缺8 accepted rounds、2-normal fault timing、至少1 restored、exact stop、priority replacement、held-out evaluation或
cleanup evidence時，Slice保持verification incomplete。

## 10. Normative conformance map

| ID | Normative item | Planned production owner／evidence |
| --- | --- | --- |
| S3-01 | 只用既有`TESTBED` + scenario + `CONFIG_DIR` pipeline，不新增selector／parallel renderer | configlib／renderer／checker |
| S3-02 | Scenario只擁有dataset、training、fault與observation behavior，不保存device或participant identity | four image scenarios與schema tests |
| S3-03 | `DEVICE=gpu`解析為Root與six Leaves CUDA、four Branches CPU | TESTBED、manifest、native config、Compose、real status |
| S3-04 | GPU acceptance使用one RTX 3080、8,192 MiB pre-start floor與seven-participant actual evidence | preflight與`events.jsonl`／`run.json` |
| S3-05 | Normal 2-round scenarios保留；replacement scenarios固定8 accepted rounds | scenario validation與normal／replacement render tests |
| S3-06 | MNIST與CIFAR-10各只有一個正式replacement flow run | run coverage checker與record wording |
| S3-07 | Replacement沿用direct production endpoint與configured deadlines，無gate／proxy／extra port／detached state | config review、runner tests與actual process／port inventory |
| S3-08 | Controller不控制replacement preparation timing；degraded count由stop、Root outcomes與first contribution推導 | `events.jsonl`與`run.json` |
| S3-09 | Fault target由Area A group與selected priority解析，exact stop primary Go+PyMTLF | manifest resolver、stop postcondition、controller JSONL |
| S3-10 | Controller不指定replacement、不修改priority、不偽造component event | call-path review、Root／NRF evidence |
| S3-11 | 每run固定8 accepted rounds，在2 normal後fault、記錄natural degraded count並至少完成1 restored | two-file phase checker與`run.json` |
| S3-12 | Ready與first contribution分開判定；replacement只加入尚未dispatch cohort | lifecycle events與successful IDs |
| S3-13 | Rejected attempt不計phase且不得有Root global evaluation | parser tests與actual consistency check |
| S3-14 | Same `mlCorreId`、new per-edge resource／`notifCorreId`、same Area A Leaves，無retained result | structured／bounded protocol evidence |
| S3-15 | Final artifact依component-native terminal identity定位，held-out與validation分離 | artifact resolver、evaluator JSON、dataset manifest |
| S3-16 | Root JSONL是round／metric authority，必要records整合到`events.jsonl`；一般log只回答bounded diagnostics | collector、two-file checker與review |
| S3-17 | Monitor增量讀取且terminal只輸出milestones／30-second heartbeat | runner tests與actual transcript |
| S3-18 | Evidence在stop／reset前收集，failed run亦保留unique directory | failure-path tests與run inventory |
| S3-19 | Stop／reset維持selected-and-active exact scope與provider host-context safety | lifecycle tests與approved-host evidence |
| S3-20 | 不新增testbed-owned hash／checksum／digest chain；只觀測既有Docker與component-native identity | semantic diff review與`run.json` schema |
| S3-21 | 所有implementation／test artifacts使用durable domain identity，不含Slice／review tracking identity | artifact review與test quality review |
| S3-22 | 兩個run只證明flow acceptance，不宣稱模型品質、dataset優劣或controlled comparison | `run.json`／record language review |

## 11. 明確非目標

- CPU acceptance替代GPU run，或CPU／GPU performance comparison。
- GPU utilization、energy、communication bytes、network cost或long-term telemetry instrumentation。
- 修改PyMTLF device resolver、CNN architecture、FedProx、aggregation或Branch `reportAfter`。
- 增加samples、epochs、seeds或runs來研究accuracy／convergence；除非目前minimum flow無法觀測fault barrier並經使用者決策。
- Paired no-failure baseline、統計推論或MNIST／CIFAR-10模型品質比較。
- Retained-result recovery、old Branch handback、Leaf replacement、多Branch failure或Root restart recovery。
- 新增persistent controller service、message broker、database、metrics backend、Grafana／Prometheus或`5g-viz`整合。
- 執行或修復legacy Flat／static Hierarchical experiment。
- 重新計算Docker image、dataset、config或artifact hash作為testbed-owned proof。

## 12. Decision gates

出現下列任一情況必須停止並請使用者決策，不得自行弱化acceptance：

- 七個CUDA participants無法在RTX 3080 10 GiB上共同start或training，必須改participants、device、samples或hardware；
- 250 ms internal observation仍無法可靠在2 normal後的in-flight round注入fault，必須增加workload或修改component hook；
- exact primary stop需要繞過provider guard、直接使用未納入inventory的provider path或擴大destructive scope；
- current component JSONL／status缺少重建8-round natural-recovery timeline、priority selection或evaluation consistency的必要資料；
- 必須修改NWDAF、PyMTLF、NRF或ADRF contract／production recovery semantics；
- MNIST或CIFAR-10任一run無法完成8 accepted rounds、2-normal fault timing、至少1 restored、held-out evaluation、
  evidence保存或cleanup；
- 需要新service、persistent state、top-level config source／directory或parallel lifecycle；
- 實際runtime存在不在selected exact scope內的VM、container、volume或state，而cleanup需要擴大target。

不阻塞本Slice的問題分類為future-phase handoff、legacy cleanup、optional hardening、integration verification gap或
unconfirmed risk，不順手加入implementation diff。

## 13. Implementation、review 與 completion sequence

### Work unit A — GPU與scenario contract

1. 更新TESTBED的seven-participant CUDA eligibility與GPU floor。
2. 從normal scenarios移除device duplication；新增兩份8-round replacement scenarios。
3. 更新renderer、manifest、native config、Compose及validators的mixed-device／phase／deadline semantics。
4. 完成focused config、capacity、normal regression及container GPU checks。

### Work unit B — Controller與evidence pipeline

1. 實作domain-specific foreground runner、exact target resolver與natural replacement phase classifier。
2. 複用existing FL control、provider guard、Compose及lifecycle owners。
3. 實作incremental JSONL phase state machine、two-file evidence writer、按需bounded diagnostics、final artifact與held-out evaluation。
4. 完成process failure、variable degraded phase、deadline、evidence consistency與cleanup tests。

### Work unit C — Real dual-dataset acceptance

1. 在approved host context完成MNIST GPU replacement run。
2. Review evidence、stop、exact reset並驗證seed restoration。
3. 重新render CIFAR-10 selected config，完成CIFAR-10 GPU replacement run。
4. 執行mandatory initial review、finding admission／remediation、final fresh-read conformance與required full verification。
5. 保持changes unstaged／uncommitted，交付affected repositories、完整diff、verification、remaining gaps及conformance供
   使用者review。
6. User review與commit approval分開；commit後仍需另行取得push approval。兩個run evidence通過review後，才建立
   flow-acceptance record並提議完成主計畫。

### Slice acceptance

- Implementation artifacts、tests與operator docs完成S3-01至S3-22，且無未處理admitted finding。
- MNIST與CIFAR-10各有一個GPU real run，皆完成8 accepted rounds、2 normal後fault、記錄natural degraded count並至少
  完成1 restored round。
- Root依priority選出replacement；controller只停止primary，不控制replacement preparation timing。
- 七個CUDA participants有actual evidence，四個Branches維持CPU，無silent fallback或OOM。
- 每個accepted outcome與Root evaluation一致，replacement ready／first contribution明確分開。
- Final held-out evaluation、`events.jsonl`、`run.json`與final Root artifact完整；必要的bounded logs保存於optional
  `diagnostics/`，CSV／plot不屬於acceptance。
- Evidence先保存，再完成process stop、ADRF／NRF／volume reset與seed restoration；四台VM可在兩run之間保留。
- Required real evidence、mandatory review及使用者確認完成後，才把Slice標為Completed並建立records；只有code或synthetic
  tests完成時，狀態維持`Implementation Complete / Verification Incomplete`。
