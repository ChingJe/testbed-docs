# Slice 2 Inventory-driven Four-VM Protocol Topology Detailed Plan

日期：2026-09-09

狀態：Completed；implementation、required real-environment evidence、使用者review與commit approval已完成

上層計畫：

- [Multi-host Branch Replacement Experiment Plan](../multi-host-branch-replacement-experiment-plan.md)
- [5G NWDAF Infrastructure Development Policy](../../../development_policy.md)

本文件只細化即將執行的 Slice 2。跨-slice 的 experiment scope、Branch replacement flow、structured evidence 與
completion 規則仍以上層主計畫為準；Slice 3 的詳細計畫會在 Slice 2 完成 review 與 commit checkpoint 後才建立。

---

## 1. Operator-visible outcome

Operator 使用既有 `TESTBED` + scenario → `config-create` → effective `CONFIG_DIR` → validate／VM／service／ML／training
control／status／logs／stop／reset pipeline，即可建立下列 canonical environment：

- 四台 VM：`core`、`path-a`、`path-b`、`path-c`；
- 一個 Root、三個 active regional Branches、Area A 一個 replacement candidate，以及六個 Leaves；
- 十一個 Guest Go NWDAF processes，和十一個 Host PyMTLF containers 一對一對應；
- 只有 NRF、ADRF 與 MongoDB persistence 的 minimal dependency graph；
- 可明確切換 MNIST 或 CIFAR-10 scenario，並以較小 workload 完成至少兩個 accepted hierarchical rounds。

Canonical `TESTBED` 會成為 Make 的預設 definition；其 `config.directory` 是後續 commands 的 effective `CONFIG_DIR`。
因此建立 config 時仍須明確選擇 MNIST 或 CIFAR-10 scenario，但後續 `vm-up`、`experiment-start`、status、logs、stop 與
reset 不需要反覆輸入一長串 `CONFIG_DIR`。切換 dataset 時必須先停止並依需要 reset 前一個 run，再以另一個 scenario
重新產生同一 canonical config directory；config identity guard 必須拒絕拿新 config 操作仍在執行的舊 runtime。

Slice 2 只證明 topology、dataset、native config、protocol、ADRF 與 lifecycle wiring 能正常工作。Branch fail-stop、
natural degraded rounds、replacement recovery 與 restored rounds 屬於 Slice 3，不在本 Slice 注入故障。

## 2. 盤點基線與實作邊界

### 2.1 Repository 與 revision 基線

本次 read-only inventory 的 revision 與 working-tree 狀態如下：

| Repository | Branch／revision | Working tree | Slice 2 disposition |
| --- | --- | --- | --- |
| `5G_NWDAF_Infrastructure` | `feat/hierarchical-fl-protocol-extension` / `f6d7134` | Parent 僅有 `ML/PyMTLF`、`NFs/nwdaf` gitlink 差異 | 唯一 implementation repository；保留既有差異並在 implementation review 中把需要的 component revision update 明確納入 |
| `ML/PyMTLF` | `feat/hierarchical-fl-protocol-extension` / `579f737` | Clean | Current native contract；不修改 component behavior |
| `NFs/nwdaf` | `feat/hierarchical-fl-protocol-extension` / `be3fa57` | Clean | Current native contract；不修改 component behavior |
| `NFs/nrf` | `feat/r18-nwdaf-discovery` / `0dd4024` | Clean | Runtime dependency；不修改 |
| `NFs/adrf` | `feat/r18-federated-learning` / `905f059` | Clean | Runtime dependency；不修改 |
| `nwdaf-resources` | `feat/hierarchical-fl-protocol-extension` / `e2b21e1` | Clean | Branch-replacement topology與real-process harness的read-only semantic reference |
| `nwdaf-docs` | `main` / `f88941b` | Clean | Component design／plan的read-only source |
| `testbed-docs` | `main` / `d256278` | Clean、相對 remote ahead 8 | 本詳細計畫與上層進度的文件 owner |

Parent 目前記錄的 `ML/PyMTLF` gitlink 是 `7479629`，實際 checked-out revision 是 `579f737`；`NFs/nwdaf` 記錄的是
`6aed268`，實際是 `be3fa57`。Slice 2 需要 current protocol、image workload與experiment recording contract，因此
implementation 應將這兩個 gitlinks 及 `components.lock.yaml` 的對應 rows 一起更新到實際 revision。這不是清理
submodule dirty state；它是本次可重建 deployment 的必要 source identity update。其他 submodule pointers 不變。

### 2.2 已確認可重用能力

Current component source 已直接確認：

- PyMTLF 原生支援 `mnist` 與 `cifar10` image-classification workload，使用各自 seed model、input shape、interoperability
  ID 與 `.npz` `images`／`labels` loader；
- MNIST contract 是 `uint8 [N,1,28,28]`、10 classes；CIFAR-10 是 `uint8 [N,3,32,32]`、10 classes；
- Root／Branch／Leaf role 由 protocol topology assignment 決定，Branch 使用 `FL_SERVER_AND_CLIENT` capability；
- Root candidate selection 已執行 `selection_method: priority`，completion policy 已支援 `accept_failures` 與
  `minimum_completion_rate`；
- FedProx `proximalMu`、`sampleWeighted` aggregation、Branch `reportAfter: round` 與 Leaf `reportAfter: epoch` 都已有
  native validation與runtime consumer；
- Root private training request API 是 `POST /internal/v1/federated-learning/training-requests`，status 使用 response
  `Location` 對應的 GET resource；
- PyMTLF 已可在 node-local `experiment-records/<mlCorreId>/observations.jsonl` 保存 Root evaluation、round cohort、
  failure與replacement events。

因此 Slice 2 不新增 protocol field、不修改 FedProx、selection、aggregation 或 component state machine，也不把
`nwdaf-resources` runner搬進 testbed。Testbed只把已存在的 native contract透過既有 renderer、Compose、systemd 與
lifecycle pipeline部署出來。

### 2.3 已確認的 testbed gaps

| Gap | Current evidence | Slice 2 disposition |
| --- | --- | --- |
| Machine inventory 固定三機 | `Vagrantfile`、`lib.sh`、status／logs／reset與tests仍有 `core path-a path-b` 固定集合 | Canonical／common reachable path改由selected `TESTBED`及generated manifest inventory驅動 |
| Path provision只認 A／B | `common.sh`與`path.sh`拒絕 `path-c`，且Path build固定包含gtp5g、UPF與UERANSIM | Provision contract接受validated machine key與selected build inventory；canonical Paths只建NWDAF |
| Runtime schema固定舊roles／UEs | `configlib.py`要求既定role counts、non-empty UE inventory及舊deployment kinds | 加入 `protocol-hierarchical` discriminator與十一-node exact contract；legacy branches保留隔離 |
| Renderer以legacy `config/default`為baseline | Canonical config仍會複製full-5GC generated example | Canonical branch不複製legacy tree；由current component templates與single renderer產生minimal files |
| Scenario固定UE traffic | schema 2 checker要求Path A／B traffic、monitoring與PseudoDriver training fields | 同一 scenario schema加入明確 image-classification variant；不新增scenario selector或第二個renderer |
| Dataset pipeline是PseudoDriver專用 | `dataset.py`／`datasetlib.py`只處理traffic Parquet | 在同一 dataset command surface加入image dataset acquisition／partition branch；legacy branch原樣隔離 |
| Lifecycle仍啟動舊資料鏈 | `services-start`仍會stage Guest dataset、gtp5g preflight與subscriber apply | Canonical manifest明確宣告無Guest dataset／subscriber／subscription domain，common lifecycle依inventory跳過 |
| 原始影像資料不存在 | Workspace沒有MNIST或CIFAR-10 raw archives | Slice 2加入受控external acquisition與deterministic preparation；缺檔不得延遲到training才失敗 |
| Seed import固定traffic seed | Compose entrypoint與manifest固定 `/seed_models/initial` | 由scenario dataset選取current PyMTLF image seed、event、interoperability與component-native artifact identity |
| Reset只清舊experiment scope | NRF cleanup與state inventory不是十一個NWDAF exact IDs | Manifest列出exact registrations、ADRF state與十一個volumes；reset只按selected-and-active scope處理 |

### 2.4 Host capacity observation

文件盤點時 Host 有 64,140 MiB RAM、24 online CPUs、約 146 GiB workspace filesystem free space；當時
`MemAvailable` 約 22,410 MiB，2 GiB swap 已用完。這只是 planning observation，不是 real acceptance evidence。
Implementation後仍須以selected manifest重新執行 capacity gate；任何 VM、Docker build或其他 workload造成的當下
資源變化都必須以執行時結果為準。

## 3. Authoritative input 與 schema extension

### 3.1 單一 selection pipeline

Canonical inputs固定為：

```text
testbed.protocol-hierarchical.yaml
+ one selected image-classification scenario.yaml
              │
              ▼
       existing config-create
              │
              ▼
config/local/protocol-hierarchical/
  native configs + topology + Compose + manifest
              │
              ▼
existing validate / VM / service / ML / control / status / logs / stop / reset
```

約束如下：

- 新增一份 complete `testbed.protocol-hierarchical.yaml`，但不新增 selector、top-level config directory、parallel renderer
  或 checker。
- `Makefile` 的 `TESTBED` default改為該 canonical definition，`NAME` default與其
  `config.directory: config/local/protocol-hierarchical`一致。
- `FROM`仍須明確選擇MNIST或CIFAR-10 scenario；dataset kind不由environment variable、container name或舊config猜測。
- `CONFIG_DIR` explicit override仍可用於診斷，但省略時只能從selected definition解析；不得回退至`config/default`。
- Existing legacy definitions仍可explicit selection；canonical default不得指向它們。

### 3.2 Canonical TESTBED contract

沿用 `schemaVersion: 1` complete definition，新增 `analytics.topology: protocol-hierarchical`。Canonical definition只保存
deployment與protocol topology，不保存dataset kind或run result：

- `machines`：四台VM的resources與management／SBI anchors；
- `networks`：只保留management與SBI private networks；Vagrant預設NAT interface只供package／time source；
- `placement`：每台Guest的exact services及十一個Host containers；
- `coreServices`：MongoDB、NRF、ADRF；
- `analytics`：十一個logical NWDAF identities、placement、SBI endpoint、role、PyMTLF mapping與Root recursive topology；
- `mlRuntime`：十一個container resource limits、ports、volumes與Host addresses；
- `operations`：clock tolerance、journal bounds及lifecycle timeouts；
- `hostSafety`：capacity與storage thresholds。

Recursive topology內以logical NWDAF unit name引用candidate，renderer再解析成native `nfInstanceId`。同一identity不得同時
在node definition與topology重複手寫；policy、priority、enabled與`reportAfter`則只在recursive topology出現。

### 3.3 Canonical scenario contract

保留同一 `schemaVersion: 2` loader，加入由 `workload.profile: image-classification`明確辨識的variant。兩個tracked
scenarios除dataset與model family外使用相同bounded settings：

| Field | MNIST | CIFAR-10 |
| --- | --- | --- |
| `workload.dataset` | `mnist` | `cifar10` |
| `workload.event` | `X_IMAGE_CLASSIFICATION` | `X_IMAGE_CLASSIFICATION` |
| `workload.modelInteroperability` | `pymtlf-image-classification-mnist` | `pymtlf-image-classification-cifar10` |
| `workload.modelFamilyId` | `image-classification-mnist` | `image-classification-cifar10` |
| `partition.seed` | 42 | 42 |
| `partition.samplesPerLeaf` | 100 | 100 |
| `partition.validationSamples` | 200 | 200 |
| `partition.heldOutSamples` | 200 | 200 |
| `training.acceptedRounds` | 2 | 2 |
| `training.batchSize` | 16 | 16 |
| `training.learningRate` | 0.001 | 0.001 |
| `training.device` | `cpu` | `cpu` |

Leaf local update count不在scenario與topology重複保存；它只由topology的Leaf
`reportAfter: {count: 1, unit: epoch}`決定。Root run長度則由scenario的`acceptedRounds: 2`生成PyMTLF server
`round_count`。Timeouts必須是bounded positive values，並允許兩個accepted rounds加上preparation、ADRF及cleanup
時間；不能以縮短timeout代替較小sample／epoch設定。

Legacy UE scenarios仍走同一loader的既有explicit variant；canonical checker不得要求其`trafficProfiles`、monitoring或
PseudoDriver fields，legacy checker也不得把image fields解讀成UE flow。

## 4. Exact physical 與 logical inventory

### 4.1 VM 與 network inventory

| Machine | vCPU | RAM | Disk ceiling | Management anchor | SBI anchor | Canonical Guest services |
| --- | ---: | ---: | ---: | --- | --- | --- |
| `core` | 2 | 3072 MiB | 24 GiB | `192.168.56.10` | `192.168.57.2` | MongoDB、NRF、ADRF、`nwdaf-root` |
| `path-a` | 2 | 2048 MiB | 20 GiB | `192.168.56.11` | `192.168.57.3` | Area A primary／replacement Branch及兩個Leaves |
| `path-b` | 2 | 2048 MiB | 20 GiB | `192.168.56.12` | `192.168.57.4` | Area B Branch及兩個Leaves |
| `path-c` | 2 | 2048 MiB | 20 GiB | `192.168.56.13` | `192.168.57.5` | Area C Branch及兩個Leaves |

Canonical private networks固定為 `management: 192.168.56.0/24` 與 `sbi: 192.168.57.0/24`。不產生N2、N3、N4、N6、
UPF、gNB或UE network aliases。Host PyMTLF bind／advertised address沿用 `192.168.57.1`。所有machine、alias與endpoint都
必須通過CIDR membership、duplicate address、port、owner及Host reachability validation。

### 4.2 NWDAF inventory

所有Go NWDAF的public SBI port為8080，ANLF／MTLF internal ports分別為8090／8091；不同process以獨立SBI alias隔離：

| Unit | Machine | Role／capability | NF Instance ID | SBI address | PyMTLF service |
| --- | --- | --- | --- | --- | --- |
| `nwdaf-root` | `core` | Root / `FL_SERVER` | `10000000-0000-4000-8000-000000000001` | `192.168.57.30` | `pymtlf-root` |
| `nwdaf-branch-a-primary` | `path-a` | Branch / `FL_SERVER_AND_CLIENT` | `10000000-0000-4000-8000-000000000101` | `192.168.57.41` | `pymtlf-branch-a-primary` |
| `nwdaf-branch-a-replacement` | `path-a` | Branch / `FL_SERVER_AND_CLIENT` | `10000000-0000-4000-8000-000000000111` | `192.168.57.42` | `pymtlf-branch-a-replacement` |
| `nwdaf-leaf-a1` | `path-a` | Leaf / `FL_CLIENT` | `10000000-0000-4000-8000-000000001101` | `192.168.57.43` | `pymtlf-leaf-a1` |
| `nwdaf-leaf-a2` | `path-a` | Leaf / `FL_CLIENT` | `10000000-0000-4000-8000-000000001102` | `192.168.57.44` | `pymtlf-leaf-a2` |
| `nwdaf-branch-b` | `path-b` | Branch / `FL_SERVER_AND_CLIENT` | `10000000-0000-4000-8000-000000000102` | `192.168.57.51` | `pymtlf-branch-b` |
| `nwdaf-leaf-b1` | `path-b` | Leaf / `FL_CLIENT` | `10000000-0000-4000-8000-000000001201` | `192.168.57.52` | `pymtlf-leaf-b1` |
| `nwdaf-leaf-b2` | `path-b` | Leaf / `FL_CLIENT` | `10000000-0000-4000-8000-000000001202` | `192.168.57.53` | `pymtlf-leaf-b2` |
| `nwdaf-branch-c` | `path-c` | Branch / `FL_SERVER_AND_CLIENT` | `10000000-0000-4000-8000-000000000103` | `192.168.57.61` | `pymtlf-branch-c` |
| `nwdaf-leaf-c1` | `path-c` | Leaf / `FL_CLIENT` | `10000000-0000-4000-8000-000000001301` | `192.168.57.62` | `pymtlf-leaf-c1` |
| `nwdaf-leaf-c2` | `path-c` | Leaf / `FL_CLIENT` | `10000000-0000-4000-8000-000000001302` | `192.168.57.63` | `pymtlf-leaf-c2` |

Leaf profile使用PLMN `466/92`與各自TAI `001101`、`001102`、`001201`、`001202`、`001301`、`001302`。它們只是
controlled image workload的NF discovery scope，不代表Slice 2啟動UPF或UE。

Area A replacement在Slice 2是「hierarchy尚未採用」而不是「process未啟動」：它的Go NWDAF與PyMTLF都必須健康並
完成NRF registration，但Root初始選擇必須因priority而採用primary。真正的process不可用與natural replacement
timing留給Slice 3。

### 4.3 Host container inventory

所有container內部listener使用9092；Host published port、volume與resource limits各自獨立：

| Service | Host port | CPU limit | Memory limit | Durable volume |
| --- | ---: | ---: | ---: | --- |
| `pymtlf-root` | 9292 | 1.0 | 1024 MiB | `pymtlf-root-data` |
| `pymtlf-branch-a-primary` | 9293 | 0.75 | 768 MiB | `pymtlf-branch-a-primary-data` |
| `pymtlf-branch-a-replacement` | 9294 | 0.75 | 768 MiB | `pymtlf-branch-a-replacement-data` |
| `pymtlf-branch-b` | 9295 | 0.75 | 768 MiB | `pymtlf-branch-b-data` |
| `pymtlf-branch-c` | 9296 | 0.75 | 768 MiB | `pymtlf-branch-c-data` |
| `pymtlf-leaf-a1` | 9092 | 1.0 | 768 MiB | `pymtlf-leaf-a1-data` |
| `pymtlf-leaf-a2` | 9093 | 1.0 | 768 MiB | `pymtlf-leaf-a2-data` |
| `pymtlf-leaf-b1` | 9094 | 1.0 | 768 MiB | `pymtlf-leaf-b1-data` |
| `pymtlf-leaf-b2` | 9095 | 1.0 | 768 MiB | `pymtlf-leaf-b2-data` |
| `pymtlf-leaf-c1` | 9096 | 1.0 | 768 MiB | `pymtlf-leaf-c1-data` |
| `pymtlf-leaf-c2` | 9097 | 1.0 | 768 MiB | `pymtlf-leaf-c2-data` |

Selected totals是8 vCPU Guest ceilings、約9 GiB Guest RAM、10 vCPU container limits與8.5 GiB container RAM，再加2 GiB
container build overhead。Canonical policy固定CPU，不要求GPU。Capacity gate必須同時檢查total capacity、當下
`MemAvailable`、online CPUs、workspace／VirtualBox／Docker storage及port conflicts；swap依既有`warn` policy顯示，
不能因swap已用完而隱藏警告。

## 5. Protocol topology 與 native config generation

### 5.1 Root topology

Renderer必須輸出PyMTLF current `StaticTopologyFile`可直接載入的native topology。Root policy固定：

```yaml
selection_method: priority
min_available_nodes: 2
fraction_train: 1.0
min_train_nodes: 2
accept_failures: true
min_completion_rate: 0.66
```

三個Branch groups如下：

- Area A candidates：primary `priority: 100`、replacement `priority: 50`，兩者皆`enabled: true`；
- Area B：唯一Branch `priority: 100`；
- Area C：唯一Branch `priority: 100`；
- 每個Branch `reportAfter: {count: 1, unit: round}`；
- 每區兩個Leaves，priority分別100／90，`reportAfter: {count: 1, unit: epoch}`；
- 每個Leaf group policy使用priority、`min_available_nodes: 2`、`min_train_nodes: 2`、
  `accept_failures: false`與`min_completion_rate: 1.0`。

Root與每個Branch group的strategy都固定為`fedProx`、`sampleWeighted`、`proximalMu: 0.01`。Renderer只做snake_case
TESTBED representation到component-native wire/config naming的mapping，不在testbed controller或container environment
再保存另一份policy default。Reference harness目前使用Leaf `report_after.count: 2`；本testbed profile改成1是已確認的
bounded workload調整，用來縮短四個real acceptance runs，不改變欄位owner、unit或protocol semantics。

### 5.2 Go NWDAF configs

Single renderer以current `NFs/nwdaf/config` role-appropriate template作結構輸入，產生十一份獨立native config：

- exact `nwdafName`、UUIDv4、SBI與internal endpoints；
- Root `FL_SERVER`、Branch `FL_SERVER_AND_CLIENT`、Leaf `FL_CLIENT` capability；
- `X_IMAGE_CLASSIFICATION` event、dataset-specific interoperability vendor contract與Leaf tracking area；
- NRF URI、HTTP/H2C、OAuth disabled；
- `anlfBackend.enabled: false`；
- one-to-one Host PyMTLF endpoint與bounded request timeout。

Go config不得攜帶dataset filesystem path、sample indices、local epoch counter或test controller instruction。

### 5.3 PyMTLF configs與seed

Renderer以current PyMTLF的`fl-server-hierarchy.yaml`、`fl-server-client.yaml`與`fl-client.yaml`為role templates，產生十一份
independent configs：

- Root擁有training trigger、server、topology、seed catalog、validation recording與publication state；
- Branch同時擁有server／client engines，但沒有local image shard；
- Leaf使用scenario-selected dataset與read-only `/data/train.npz`；
- 每個node有獨立artifact、model-state、publication、workspace與experiment-record directories，全部位於自己的volume；
- artifact origin allowlist由recursive topology精確推導：Root允許Branches，Branch允許Root及其Leaves，Leaf允許其
  candidate parents；需要下載Root global model的nodes另允許ADRF origin；
- Root per-round validation只讀 `/data/validation.npz`；final held-out file不進入任何training config。

Root container entrypoint依scenario選擇tracked PyMTLF image seed source、event、model ID、interoperability與
component-native artifact key。這個artifact key是PyMTLF既有model artifact contract，不是testbed另算的config hash；
entrypoint必須以current importer得到的actual key核對config期待值，錯誤時不啟動Root。

### 5.4 NRF、ADRF、MongoDB與generated artifacts

Canonical renderer不複製legacy `config/default` tree。它以selected `TESTBED`值產生minimal NRF config，並以current ADRF
native template產生ADRF config；MongoDB command與storage path由manifest exact inventory決定。

Generated `CONFIG_DIR`至少包含：

```text
manifest.yaml
compose.yaml
nrfcfg.yaml
adrfcfg.yaml
network/{core,path-a,path-b,path-c}.yaml
topology/protocol-hierarchical.yaml
nwdafcfg-<logical-node>.yaml            # 11
pymtlf-<logical-node>.yaml              # 11
```

不得產生AMF、SMF、UPF、UERANSIM、subscriber、consumer、PyAnLF或WebConsole config。Manifest保存exact machines、Guest
services、NWDAFs、Host containers、ports、volumes、dataset paths、capacity、start order、reset scope、coordinator、
seed restoration與optional services disabled state。Canonical config identity只使用Slice 1保留的single generated-tree
runtime digest；不重新加入definition、generator、dataset shard或nested provenance hashes。

## 6. MNIST 與 CIFAR-10 dataset preparation

### 6.1 External source boundary

Raw datasets目前不在workspace。Canonical `dataset-generate` branch需從固定HTTPS來源取得：

- MNIST原始IDX gzip archives；
- CIFAR-10 binary archive，避免載入外部pickle。

下載只寫入ignored `.cache/image-datasets/`。固定URL與archive名稱由dataset acquisition code內的reviewed allowlist維護，
不成為scenario欄位、另一份operator config source或自動fallback清單；來源不可用時直接失敗並進入decision gate。

此boundary不新增checksum、digest或其他hash-like identity chain。Slice 2的production requirement是取得符合既定dataset
語意的資料並完成短流程，不要求用content-derived value證明每次下載的bytes完全相同；因此以HTTPS、固定source allowlist、
archive安全解析，以及expected source counts、dtype、shape、class與label range驗證直接保護實際consumer。Raw extraction、
generated `.npz`、split manifest與config同樣不得建立hash-like provenance。

下載器必須有bounded timeout、temporary file、atomic rename、archive size ceiling及安全member allowlist。HTTPS、archive
parsing、expected source counts／shapes或labels任一不符時刪除temporary file並fail closed，不保留partial cache。

### 6.2 Deterministic split contract

同一 `dataset.py` command surface根據scenario variant執行image branch。每次generate都從verified raw source及scenario
seed重建selected output到temporary directory，通過validation後atomic replace exact generated dataset directory，不靠
generated-content hash判斷reuse。

兩種dataset都使用相同語意：

1. 依class分組後以seed 42各自permutation；
2. 從official training split配置六個互斥Leaf shards，每個100 samples且每class 10 samples；
3. 從official test split以獨立seed 43配置Root validation與final held-out，各200 samples且每class 20 samples；
4. train、validation、held-out source indices完全互斥；validation與held-out使用不同檔案及不同consumer；
5. 每個`.npz`只含`images`與`labels`，使用PyMTLF要求的dtype、rank、shape與class range；
6. split manifest保存dataset、source split、seed、node-to-source-index mapping、counts、shape與class histogram，使結果可從
   verified raw source重建，不保存derived hash。

Generated output位於ignored `.generated/image-datasets/<scenario-name>/`。`manifest.yaml`保存該expected path；Host lifecycle
自動將它傳給Compose，operator不需另設`DATASET_ROOT`。六個Leaf shard和Root validation以read-only bind mount加入selected
containers；held-out檔保留在Host，Slice 2不讓training runtime讀取。

### 6.3 Native validation

Config validation必須實際使用current PyMTLF `Settings` loader、`StaticTopologyFile`與`ImageDatasetLoader`驗證兩個scenarios的
全部generated PyMTLF configs、topology、六個shards與Root validation。Seed model必須以component-native importer／bundle
loader確認dataset、event、interoperability、shape與model contract一致。Testbed自己的YAML／NumPy parse只能作前置診斷，
不能取代native acceptance。

## 7. Inventory-driven infrastructure 與 lifecycle

### 7.1 Machine與provider boundary

`Vagrantfile`、`lib.sh`及其provider guard必須先從explicit selected `TESTBED`解析non-empty、unique、safe machine names，
再建立唯一的selected machine inventory。所有metadata、machine-readable status、VBoxHeadless UUID與process comparisons都
使用該集合；任何missing、extra、duplicate、changing或unparseable record仍fail closed。

Real `vagrant`／`VBoxManage` command無論由Make、script或test間接啟動，都只能走approved sandbox-outside execution。
Provider guard tests只使用synthetic process／metadata fixtures，不以real provider command測試blocking。

原有三台legacy VM的disk與once-only provisioning內容不符合minimal four-VM contract，不能以`vagrant up`原地視為完成遷移。
本計畫review期間已先由Host OS process inventory觀測，再核對Vagrant metadata與VirtualBox provider state；當時只有
`core`、`path-a`、`path-b`三台registered且均為`poweroff`，沒有running或unexpected／orphan VM。經使用者明確批准後，
三台VM已在approved host context完成destroy且未備份Guest disk，事後狀態均為`not_created`，VirtualBox registered／running
inventory皆為空。這只關閉clean-start前置條件，不構成Slice 2 implementation或real acceptance evidence；後續仍須從
new canonical inventory重建三台並新增`path-c`。

### 7.2 Guest provisioning

Common provisioning接受selected machine key、安裝shared runtime tools及chrony，不再列舉A／B。Role-specific build由selected
Guest service inventory決定：

- `core`只build NRF、ADRF與NWDAF；
- 三個Path只build NWDAF；
- canonical path不build gtp5g、UPF、UERANSIM或其他5GC binaries；
- legacy profile原有build branch可保留，但不能由canonical inventory到達。

Component mapping必須是reviewed allowlist；YAML unit name不能直接轉成任意filesystem path或shell command。Provision manifest
保存實際OS、tool與component revisions，startup比對actual binary／source identity，不以build成功推定revision正確。

### 7.3 Clock synchronization

所有Guest啟用chrony，透過NAT reachability取得time source。Real experiment start前逐機驗證chrony synchronized state，並以
同一Host sampling window比較四個Guest UTC clock；任兩個Guest或Guest-to-Host偏差超過
`operations.clockSkewToleranceMs: 1000`即拒絕啟動training。NTP source unavailable、clock step未完成或SSH measurement
ambiguous都fail closed。這個check只保障event timeline；不把timestamp當成round completion barrier。

### 7.4 Render、activate與start

Canonical start sequence：

1. resolve selected `TESTBED`與effective `CONFIG_DIR`，exact compare manifest；
2. validate dataset availability、Host capacity、ports、provider runtime與clock；
3. sync shared Guest tools，stage四機config並執行all-or-rollback activation；
4. reconcile四機management／SBI aliases；
5. 依manifest order啟動MongoDB → NRF → ADRF →十一個Go NWDAFs並等待required readiness／registration；
6. 啟動十一個Host PyMTLF containers並等待native health；
7. 執行cross-boundary reachability與one-to-one backend checks後才宣告experiment active。

Canonical path不執行PseudoDriver dataset stage、gtp5g preflight、subscriber apply、Consumer subscriptions或WebConsole start。
Aggregate start在任一階段失敗時只停止本次已啟動的selected runtimes；若cleanup失敗，保留evidence並明確列出仍active
targets，不碰unexpected runtime。

### 7.5 Training control、status與logs

Existing `fl-control.py`依deployment kind加入protocol-hierarchical branch，沿用`fl-training-start`／`fl-training-status` operator
surface：

- start向Root PyMTLF private API送出canonical UUIDv4 `RUN_ID`及scenario `modelFamilyId`；
- status只追蹤response resource對應的request／plan identity，輸出state、current round、completed rounds與failure；
- canonical path不建立collection resources，也不接受legacy `fl-collection-*`作為prerequisite；
- control前核對selected config、Root container identity與scenario model family。

`services-status`、`experiment-status`與`logs`使用manifest machine／service inventory，支援`path-c`與十一個containers；
不維護第二份VM allowlist。Status必須分開呈現provider state、active config identity、Guest units、container health、NRF
registrations、ADRF health與training request，不以其中一項健康推定整體健康。

### 7.6 Stop、restart與reset

- Stop依manifest reverse order停止selected training／NWDAF／dependency runtimes，保留VM、volumes、generated datasets與logs。
- Restart使用同一selected config；任何Guest或container仍帶不同config identity時拒絕執行。
- Reset前要求selected containers及Guest services全部停止，並列出plan；apply只清十一個PyMTLF volumes、exact十一個NWDAF
  NRF registrations、ADRF registration／model records／model files及Mongo experiment state。
- Unexpected container、volume、unit、Vagrant metadata或provider process只報告並拒絕normal reset；事故cleanup另作fresh
  inventory並取得批准。
- Reset保留verified raw dataset cache與generated read-only splits；下一次start會依selected scenario重新validate／rebuild
  splits。Root volume清空後由component-native importer恢復該dataset的seed。

## 8. Legacy profile bounded migration

Slice 2對`testbed.yaml`、`testbed.static-flat.yaml`與`testbed.static-hierarchical.yaml`只做下列bounded work：

1. 讓selected machine解析、provider guard、stage／activate、status／logs、stop／reset common helpers可從各definition取得三機
   inventory，不再依global `MACHINES`常數；
2. 執行parse、schema、reference、non-empty inventory、network ownership、exact destructive scope與synthetic provider
   safety checks；
3. 能用同一renderer直接保持的legacy branch就保留；若必須新增compatibility schema、parallel renderer、舊service或
   profile-specific repair才可啟動，則保留source但在README inventory記錄migration gap與不可啟動；
4. 不執行legacy VM、5GC、UE、PseudoDriver或Flat／static Hierarchical training regression。

Common helper中的canonical／legacy discriminator必須是deployment contract，而不是散落的filename comparison。Dormant
legacy-only Path A／B、UE、UPF或Consumer code可以留在reviewed branch；repository-global hard-code為零不是驗收條件，
但canonical或common reachable path不得再依賴它們。

## 9. Implementation work units

本Slice在同一work unit內依序完成，不另外建立子-slice文件：

### 9.1 Contract與inventory foundation

- 加入canonical complete TESTBED與兩個image scenarios；
- 更新current component gitlinks／locks；
- 擴充`configlib.py`的deployment、machine、service、NWDAF、capacity與reset inventory；
- 將Vagrant及common provider／Guest helpers改為selected inventory。

### 9.2 Minimal renderer與dataset flow

- 在existing renderer加入protocol-hierarchical branch及minimal native configs；
- 在existing dataset command surface加入external source、deterministic split與native validation；
- 產生十一-container Compose、read-only mounts、dataset-specific seed與exact manifest。

### 9.3 Lifecycle與operator surface

- 改造provision、stage／activate、network、start／rollback、status／logs、training control、stop／restart／reset；
- 加入clock與runtime capacity preflight；
- 隔離canonical flow不需要的legacy dataset／5GC／subscription branches。

### 9.4 Focused verification與legacy disposition

- 把durable assertions加入既有domain tests，必要時新增以功能命名的image dataset test；不得出現`slice2`、phase或
  implementation checkpoint命名的meta-test；
- 對兩個canonical scenarios與三個legacy definitions執行各自required static checks；
- 更新README legacy asset inventory的migration disposition，不宣稱legacy profiles已能real run。

### 9.5 Approved real-environment acceptance

- 經approved host-context flow完成exact VM rebuild與四機startup；
- 驗證十一組NWDAF↔PyMTLF、NRF、ADRF、Mongo、network、clock與cleanup；
- MNIST、CIFAR-10各跑一個至少兩個accepted rounds的short normal flow；
- 完成mandatory initial review與user-review handoff，未取得required real evidence時保持
  `Implementation Complete / Verification Incomplete`。

## 10. Verification matrix

| Requirement | Static／synthetic evidence | Real-environment evidence |
| --- | --- | --- |
| Single authoritative pipeline | default/effective selection、no fallback、scenario discriminator tests | Operator commands在未輸入`CONFIG_DIR`時解析同一canonical directory |
| Four-machine inventory | valid／empty／duplicate／extra machine fixtures；Vagrant config parser review | provider／OS exact `core`、`path-a`、`path-b`、`path-c` comparison |
| Provider safety | synthetic Vagrant metadata、UUID與VBoxHeadless process fixtures | 每個real provider action前approved host-context preflight |
| Minimal Guest placement | generated service and build allowlist exact tests | OS process inventory只有MongoDB、NRF、ADRF、十一個NWDAFs |
| Eleven one-to-one pairs | unit、UUID、SBI、internal endpoint、Host port、volume、backend mapping uniqueness | 十一個Go／PyMTLF health及雙向endpoint reachability |
| Protocol topology | current `StaticTopologyFile` loader、policy／priority／reportAfter／strategy exact checks | Root initial cohort選Area A primary，不採用低priority replacement |
| Native configs | current NWDAF／PyMTLF native loader或等價non-starting validation | NRF registrations、ADRF store／retrieve、Root trigger與two-round completion |
| MNIST data | external archive、balanced partition、disjointness、native loader／seed model tests | Six mounted shards、Root validation與至少2 accepted rounds |
| CIFAR-10 data | external archive、balanced partition、disjointness、native loader／seed model tests | Six mounted shards、Root validation與至少2 accepted rounds |
| Clock alignment | parser、threshold與ambiguous sample tests | 四Guest synchronized且observed skew ≤1000 ms |
| Capacity gate | manifest arithmetic、port conflicts、insufficient RAM／CPU／storage fixtures | Actual pre-start capacity result及runtime headroom observation |
| Activation／rollback | partial activation、wrong identity、start failure tests | controlled activation/start failure或等價approved host check |
| Status／logs | dynamic machine/service filters、path-c、unexpected runtime tests | provider、Guest、container、NRF、ADRF與training status一致 |
| Stop／reset | selected-and-active exact scope、unexpected unit/container/volume fixtures | all selected processes stopped；exact state clean；dataset retained；seed可恢復 |
| Legacy bounded migration | 三個definitions的common static safety checks與README disposition | 不要求legacy real evidence |

Repository tests不能取代real VM、container、NRF、ADRF、network、clock與training evidence。MNIST成功不能替代CIFAR-10，
CIFAR-10成功也不能替代MNIST。

## 11. Normative conformance map

Implementation review必須逐項列出production path、deterministic test、verification result與remaining gap：

| ID | Slice 2 item | Expected implementation owner |
| --- | --- | --- |
| S2-01 | Canonical Make default、explicit scenario與definition-owned effective `CONFIG_DIR` | `Makefile`、`testbed.protocol-hierarchical.yaml`、config resolution tests |
| S2-02 | Canonical／common machine inventory不依賴三機常數 | `Vagrantfile`、`configlib.py`、`lib.sh`、lifecycle scripts |
| S2-03 | Provider metadata、process與state exact comparison仍fail closed | provider guard與synthetic tests |
| S2-04 | Four-VM minimal placement與selective Guest builds | TESTBED、Guest provisioning、manifest tests |
| S2-05 | Eleven NWDAF identities／endpoints／roles與eleven PyMTLF ports／volumes一對一 | renderer、checker、Compose tests |
| S2-06 | Root candidate pools、priority、policy、FedProx、sampleWeighted與reportAfter只有一個authoritative topology source | TESTBED recursive topology、renderer、native loader |
| S2-07 | Area A replacement process健康但初始hierarchy不採用 | topology validation、NRF registration與normal-run cohort evidence |
| S2-08 | MNIST／CIFAR-10由scenario選取且不由environment推測 | scenario loader、renderer、control tests |
| S2-09 | External raw acquisition使用固定HTTPS source allowlist與semantic validation，不新增hash-like identity chain | dataset acquisition code與focused tests |
| S2-10 | Six balanced train shards、Root validation與held-out可由source indices／seed重建且無generated hash chain | dataset generator、split manifest、native loader tests |
| S2-11 | Dataset-specific seed、event、interoperability與model contract一致 | renderer、entrypoint、component-native importer |
| S2-12 | Canonical start不進入UPF／UE／subscriber／Consumer／PyAnLF／WebConsole path | lifecycle reachability tests與real process inventory |
| S2-13 | Chrony與≤1000 ms clock-skew gate在training前完成 | provision、preflight與real result |
| S2-14 | Capacity包含VM、containers、build overhead、storage及CPU policy | manifest、preflight與rejection tests |
| S2-15 | Training control重用existing entrypoint並核對runtime identity | `fl-control.py`、Make targets、API fixtures |
| S2-16 | Status／logs／stop／reset由selected manifest inventory驅動 | lifecycle scripts與tests |
| S2-17 | Reset清exact registrations／ADRF／volumes，拒絕unexpected runtime | reset manifest、synthetic與real cleanup evidence |
| S2-18 | Legacy profiles只有bounded static migration，不建立parallel compatibility flow | config branches、README inventory與static safety results |
| S2-19 | MNIST與CIFAR-10各至少兩個accepted normal rounds | Root status與node-local JSONL evidence |

## 12. Failure 與 recovery behavior

- Canonical TESTBED或scenario missing／invalid／ambiguous：render前fail closed。
- Raw archive下載、HTTPS、archive parser、count、shape、class或split失敗：不發布dataset directory，不開始runtime。
- Config output已存在：預設拒絕覆寫；explicit force只替換exact canonical config directory。若舊runtime仍active，後續identity
  checks必須拒絕start／stop／reset混用，operator需回到舊config或進行approved incident handling。
- Capacity、port或clock gate失敗：不開始Guest service或Host container。
- Partial config activation：rollback已成功machines到原identity；任何rollback失敗都列出exact machine並停止流程。
- Mongo／NRF／ADRF／NWDAF／PyMTLF任一start或readiness失敗：停止本次已啟動的selected runtimes，保留bounded logs與state。
- Training request未達兩個accepted rounds、terminal非`COMPLETE`或Root JSONL不一致：該dataset run失敗；不得以container健康
  或一個round代替。
- Stop／reset遇unexpected runtime：normal path拒絕操作，不擴大target；若需要清理，先重新盤點並取得批准。
- Real provider或component evidence缺失：保留open verification state，不把implementation tests描述成Slice完成。

## 13. Slice acceptance

Slice 2只有同時符合下列條件才可標為completed：

- S2-01至S2-19都有production path、test與review result，沒有未分類canonical／common hard-code或destructive target；
- four-VM selected inventory與actual provider／OS state exact match；
- minimal dependency graph及十一個NWDAF↔PyMTLF pairs通過health、registration、reachability與identity checks；
- MNIST與CIFAR-10各完成一個short normal run，每個run至少有兩個accepted hierarchical rounds及相對應Root
  `ROOT_ROUND_OUTCOME`／`MODEL_EVALUATION` records；
- Area A primary因priority出現在initial accepted cohort，replacement process雖健康但未被初始採用；
- stop、restart、reset、seed restoration及unexpected runtime refusal皆有required evidence；
- 三個legacy definitions各有migration disposition，沒有把static-only checks描述成可執行保證；
- 完成mandatory initial review，保持changes unstaged／uncommitted並交付affected repositories、完整diff摘要、verification、
  gaps與plan conformance給使用者review；review通過後再另提commit proposal。

如果implementation完成但real four-VM或任一dataset training evidence尚缺，狀態只能是
`Implementation Complete / Verification Incomplete`。

### 13.1 Current review checkpoint

2026-09-10的approved host-context verification已完成下列evidence：

- exact provider inventory為`core`、`path-a`、`path-b`、`path-c`，四台VM皆由canonical definition重建並保持running；
- minimal Guest placement、12筆NRF registrations、十一個NWDAF↔PyMTLF pairs、雙向backend reachability與clock-skew gate通過；
- CIFAR-10 request `fc2fc0aa-dffc-4578-ab45-3b713c7dbc9a`完成兩個accepted rounds，Root records的Area A候選為
  primary `10000000-0000-4000-8000-000000000101`，未採用replacement
  `10000000-0000-4000-8000-000000000111`；
- 同一CIFAR-10 config完成stop→restart，十一個containers、12筆registrations與雙向backend checks重新ready；
- MNIST request `d821786b-ff0b-4d84-863b-1eb353cf4f27`完成兩個accepted rounds，具有相同priority-selection結果；
- 兩個dataset各有一筆`ROOT_INITIAL`、兩筆accepted `ROOT_ROUND_OUTCOME`與兩筆對應`ROOT_GLOBAL`
  `MODEL_EVALUATION`；這些數值只證明integration flow，不作模型品質比較；
- exact stop與guarded reset／verify已清空十一個PyMTLF volumes、selected NRF／ADRF records與model storage；raw cache、
  generated datasets、containers、images與VM保留；
- initial review發現的partial activation attribution、unexpected project resources、stale NRF URI cleanup與protocol status
  routing均已完成behavioral remediation與targeted follow-up review。

目前generated config選取MNIST，selected Guest／Host experiment processes已停止，selected state已驗證為空；完成review後，
四台VM已經由guarded lifecycle關閉並重新確認為`poweroff`。唯一environment warning為free swap低於建議值；runtime RAM、CPU與storage gates均通過。Legacy profiles仍維持
README所述的unverified disposition，不納入real regression。

## 14. 明確延後至 Slice 3

- 停止Area A primary Go NWDAF與PyMTLF；
- 由Root依production path自然準備replacement並觀測實際degraded count；
- 8 accepted rounds、2 normal後fault及至少1 restored round；
- Branch failure／replacement latency與controller events；
- final held-out evaluation、`events.jsonl`／`run.json`與flow-acceptance record；CSV／plot留給後續離線分析；
- quiet long-running monitor與bounded failure-log collection。

Slice 2可保存node-local JSONL作integration evidence，但不得提前建立fault controller、process kill path或Slice 3
analysis pipeline。

## 15. Decision gates

遇到下列任一情況必須停止implementation並更新計畫請使用者決策：

- current NWDAF／PyMTLF native loader反證上述contract，必須修改component behavior或protocol schema；
- official MNIST／CIFAR-10 source無法合法、穩定或可驗證地取得，必須改為人工dataset input或新增第三方library；
- four-VM／eleven-container capacity gate不通過，必須減少participants、改變isolation、增加Host或使用GPU；
- minimal runtime仍需要AMF、SMF、UPF、UDM、UDR、PyAnLF或其他被排除dependency；
- canonical rendering必須新增第二個config source、selector、renderer、checker或current-selection state；
- safe migration必須弱化provider、config identity、partial activation、unexpected runtime或exact reset guard；
- existing VM inventory含unexpected／orphan machine，無法只destroy已確認的`core`、`path-a`、`path-b`；
- 任一dataset無法完成兩個accepted rounds，且需要降低acceptance而不只是調整bounded timeout／sample／resource設定。

非阻塞finding分類為Slice 3 handoff、legacy gap、optional hardening或future work，不在本Slice順手擴大component或
visualization scope。
