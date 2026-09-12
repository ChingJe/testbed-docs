# Slice 3 Dual-dataset Branch Replacement Acceptance Detailed Plan

日期：2026-09-11

最近更新：2026-09-13

狀態：Completed；required evidence、mandatory review、user review 與 verified record 均已完成

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
5. 以`SIGKILL` fail-stop Area A primary Branch 的 exact Go NWDAF 與 PyMTLF container，並確認沒有自動重啟；
6. Root在背景自然進行replacement preparation，runner觀測實際accepted degraded round數；
7. 由 Root 依 candidate priority 選出 replacement；
8. replacement 首次成功貢獻後，在固定8 accepted rounds內至少完成1個restored round；
9. Root到達`COMPLETE`且`FINAL_MODEL_SAVED`通過初步核對後，先保存training-complete checkpoint；預設接著從持久
   procedure record收集final model、執行獨立held-out evaluation並將必要evidence整合成`events.jsonl`與`run.json`，
   再停止本次processes。若後處理失敗，可用同一`RUN_NAME`單獨重試，不重新訓練。

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
make fl-branch-replacement-run RUN_NAME=123
```

正常情況仍只需上述一個run target。若training已完成，但final-model collection、evidence finalization或held-out
evaluation失敗，修正後使用同一domain runner的collection入口重試：

```bash
make fl-branch-replacement-collect RUN_NAME=123
```

Collection入口必須讀取既有run checkpoint與Root持久experiment-record volume，不得再次提交training request，也不得
要求Root process或container仍在執行。它不是新的deployment selector、renderer或config source。

`RUN_NAME`是operator定義的安全本地名稱，不是PyMTLF protocol identity。新training建立exclusive run directory後，runner
自行產生canonical lowercase UUIDv4 `requestId`，在送出request前連同`runName`原子寫入`run.json`，再將該UUID原樣傳給
PyMTLF private training API。PyMTLF建立hierarchical procedure後回傳的`planId`／`mlCorreId`仍是第三個獨立identity。
三者的binding固定保存於同一checkpoint：

```text
runName   = 123                                      # operator-facing directory identity
requestId = 8f20b31e-806a-4b38-b670-c92f23438c61   # runner-generated PyMTLF request identity
planId    = <PyMTLF-created procedure identity>
```

`RUN_NAME`必須符合`[A-Za-z0-9][A-Za-z0-9._-]{0,63}`，因此`123`與`mnist-test-1`合法，路徑分隔符、空白、`.`及`..`不合法。
同一dataset下的新training遇到既有同名run directory必須fail closed，不覆寫也不自動改名；只有collection-only入口可重用
既有名稱，且checkpoint狀態必須是`collection-pending`或`collection-failed`。既有通用`fl-training-*`／`fl-collection-*`
commands的`RUN_ID`與PyMTLF UUIDv4 contract維持不變；本Slice只在domain-specific runner前提供人類可讀的name mapping。

CIFAR-10 run只在前一run已stop、evidence已保存且exact reset完成後，將`FROM`換成
`experiments/protocol-hierarchical/branch-replacement/cifar10/scenario.yaml`重新render，再使用相同run target與新的
`RUN_NAME`。Runner會為它產生新的UUIDv4 `requestId`。Canonical `TESTBED`與effective `CONFIG_DIR`維持預設，因此後續
commands不需重打長路徑。

## 2. 盤點基線與直接證據

### 2.1 Repository 與 revision 基線

本次 read-only inventory 的 current revision 如下；implementation 開始前仍須重新檢查 working tree，不能把此表當成
未來 run 的 actual revision evidence。

| Repository | Branch／revision | Slice 3 disposition |
| --- | --- | --- |
| `5G_NWDAF_Infrastructure` | `feat/hierarchical-fl-protocol-extension` / `8495f4a` | 唯一 implementation repository |
| `ML/PyMTLF` | `feat/hierarchical-fl-protocol-extension` / `bdbd2a9` | GPU training、evaluation、JSONL、Root final-model persistence與replacement runtime；預設 read-only |
| `NFs/nwdaf` | `feat/hierarchical-fl-protocol-extension` / `be3fa57` | Guest protocol transport 與 exact process target；預設 read-only |
| `NFs/nrf` | `feat/r18-nwdaf-discovery` / `0dd4024` | priority candidate fresh exact-ID discovery；預設 read-only |
| `NFs/adrf` | `feat/r18-federated-learning` / `905f059` | global model distribution與temporary records；預設 read-only |
| `nwdaf-resources` | `feat/hierarchical-fl-protocol-extension` / `24f8b63` | 已跑通的replacement／persistent final-model evidence語意參考；不搬移其runtime owner |
| `nwdaf-docs` | `main` / `3107680` | component contract、final-model handoff與event schema的read-only source |
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
- PyMTLF Root在成功terminal前把最後accepted `ROUND_GLOBAL`原樣保存為
  `experiment-records/<mlCorreId>/final-model.tar.gz`，並輸出`FINAL_MODEL_SAVED`。保存失敗或同procedure內容衝突時
  不進入`COMPLETE`；FL workspace之後仍依原本lifecycle清理。
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
| PyMTLF image（先前runs） | `5g-nwdaf-infrastructure/pymtlf:local`，size約5.42 GB，OCI revision `579f7375…` |
| Disposable CUDA probe | `torch 2.5.1+cu121`、`cuda_available=True`、one RTX 3080 visible |

PyMTLF source已直接確認：Linux x86_64 image安裝CUDA 12.1 PyTorch wheel；device resolver接受`cpu`、`cuda`與
`cuda:N`並在CUDA不存在時fail closed；Leaf local model／tensor與Root validation evaluation都會移到configured
device。MNIST與CIFAR-10 seed model分別約78 KiB與81 KiB，兩者皆是小型CNN；每個Leaf目前只有100個training
samples、batch size 16、每輪local epoch 1。

上述image observation只描述先前runs，不再是後續驗證target。所有後續synthetic、container及real runs都必須從
PyMTLF `bdbd2a9`重建image，並在run metadata確認OCI revision不再是`579f7375…`；若實際image仍是舊revision，必須在
啟動training前fail closed。

這些資料證明目前image與runtime可看見GPU，但不等於七個process並行時的容量evidence。正式run前仍須啟動完整
selected GPU inventory並確認七個CUDA participants皆ready、沒有OOM／device fallback，且實際GPU memory狀態符合
本文件的admission contract。

2026-09-12第一次real MNIST GPU run已證明上述minimum workload不足以形成可操作的fault window。前三個accepted
outcome時間為`09:57:07.412319Z`、`09:57:07.573837Z`與`09:57:07.745261Z`，相鄰間隔約0.162秒與0.171秒；250 ms
polling一次即取得第三個accepted outcome，因此runner依既定contract fail closed，未注入fault。這不是GPU capacity、
replacement或component recovery failure，而是本Slice decision gate明列的workload timing contradiction。使用者已
確認先將兩個replacement scenarios提高為每Leaf 8,000 samples與local epochs 4；normal scenarios維持原本100／1。

2026-09-12第二次real MNIST GPU run使用8,000／4，第一個accepted outcome從initial evaluation到完成約5.9秒，但六個
Leaves隨後全部以exit 137、`OOMKilled=true`退出；第二輪因此在約295秒後由三個Branches全部回報failure，Root拒絕該
round並結束run。直接原因是六個Leaf containers各自的768 MiB Host RAM上限，不是GPU VRAM不足：run開始時GPU仍有
9,988 MiB free。該failed run已保存evidence並完成selected exact reset。

使用者依decision gate確認保留每Leaf 8,000 samples，將local epochs提高至32以擴大fault window，並把六個Leaf
container上限提高至2,048 MiB。當下Host有64,140 MiB total、25,137 MiB available，四台VM仍在運行；調整後十一個
containers的memory limits合計16,384 MiB，扣除6,144 MiB Host reserve後仍有2,609 MiB runtime餘裕。另計2,048 MiB
build overhead時只剩561 MiB，因此正式run期間不得並行build或加入其他大型workload。

2026-09-12第三次real MNIST GPU run（run ID `c4f97fb2-8287-4dc9-8e43-503ea8dba195`）確認2,048 MiB Leaf memory已
排除OOM，並完成2個normal與2個accepted degraded rounds，但replacement準備失敗。Runner使用`systemctl stop`與
`docker compose stop --timeout 30`，讓舊Branch收到`SIGTERM`並執行graceful cleanup；舊Branch先刪除Area A Leaf
subscriptions，Leaves因而retire同一hierarchy-wide `plan_id`，replacement以相同`mlCorreId`建立new edge時被
`plan_id is retired in this process`拒絕。`nwdaf-resources`既有real-process harness的fault path則對PyMTLF與Go NWDAF
直接呼叫process `kill()`，在Linux上送出`SIGKILL`；其正常teardown另走`SIGTERM`。因此本Slice的fault contract必須
恢復為hard fail-stop，不能複用graceful stop lifecycle。舊harness為固定degraded round數使用的preparation proxy仍不
納入本Slice；只有fault signal語意需要對齊。

同一run也證明scenario宣告的`localEpochs: 32`尚未進入Leaf training：generated topology仍從TESTBED複製
`report_after.count: 1`，而PyMTLF Leaf以該node instruction覆蓋server round metadata，所以實際local work仍是1 epoch。
本Slice將scenario的`training.localEpochs`固定為Leaf epoch唯一人工來源：TESTBED仍擁有Leaf hierarchy與candidate
identity，但Leaf candidate不得再定義`reportAfter`；renderer依Leaf role產生`unit: epoch`，並以scenario值產生
`count`，形成每個generated Leaf的完整native `report_after`。Validator與run evidence再核對actual value為32。這不修改
PyMTLF protocol或training behavior，只移除互相矛盾的人工truth。

2026-09-12第四次real MNIST GPU run（run ID `860f27a3-6cde-4671-a6df-a550a45bc748`）已確認修正後的
8,000 samples／32 local epochs可完成2個normal與2個accepted degraded rounds，且Guest與Host原process最後皆由
`SIGKILL`終止並未自動重啟；但先對仍active的Guest systemd instance執行`mask --runtime`會觸發unit reload。Guest
journal在mask當下先出現`Current command vanished from the unit file`與MTLF backend generation reset，舊Go NWDAF因此在
真正收到`SIGKILL`前刪除Area A Leaf subscriptions，Leaves再次retire同一`plan_id`，replacement preparation失敗。後續
Root的`protocol participants cannot change during an active operation`是清理該失敗candidate時的次生錯誤，不是最初
retirement來源。Fault path因此固定改為先對exact Guest unit cgroup送出`SIGSTOP`，在application無法執行cleanup時安裝
runtime mask，再送出`SIGKILL`；mask負責阻止`Restart=on-failure`，normal teardown仍負責解除mask。

2026-09-12第五次real MNIST GPU run（run ID `d0bb1110-ea13-49bb-b00e-8c27deb33f6d`）確認Guest
freeze→mask→`SIGKILL`已排除mask本身觸發cleanup的問題，並完成2個normal與6個accepted degraded rounds；但Guest與
Host採平行stop仍留下跨owner race。Host PyMTLF在`12:38:25.335Z`先被`SIGKILL`，Guest Go NWDAF在guarded SSH
preflight期間仍可執行，並於`12:38:56.301Z`／`12:38:56.303Z`刪除兩個Area A Leaf subscriptions；controller直到
`12:39:06.966Z`才確認兩個owner皆停止。Root於`12:43:25.452Z`偵測primary failure後立即開始replacement，約18毫秒內
建立replacement resource，但兩個Leaves都以`plan_id is retired in this process`拒絕，因此後續增加round數也不會
恢復。Fault path必須改為兩階段：先完成exact target preflight，接著在Host backend仍存在時凍結Guest，再凍結Host
container；只有兩個application owners都不能執行cleanup後，才安裝Guest runtime mask並對兩者送出`SIGKILL`。

2026-09-12第六次real MNIST GPU run（run ID `edec6824-53a0-4a25-bf99-074a3eb2af95`）驗證兩階段fault修正
有效：run完成8個accepted rounds，phase為2 normal、2 degraded、4 restored；failure detection、replacement ready與first
contribution相對confirmed stop分別為256.489、257.143與374.445秒。Replacement從round 4起和Area B／C共同成功貢獻，
沒有再出現Leaf plan retirement。但run在final artifact copy失敗，因此不能作為完整acceptance evidence。直接原因是當時runner
把terminal `candidateDigest`當成persistent `storage.artifact_root`的key；當時protocol Root的terminal aggregate只保留在
`federated_learning.workspace_root/<planId>/<rootNfInstanceId>/<roundInd>/ROUND_GLOBAL/<candidateDigest>.tar.gz`。這形成
後續component final-model persistence follow-up的直接輸入；它不再是目前runner應採用的artifact source。

2026-09-12第七次real MNIST GPU run（run ID `08460924-e09c-4bb1-bdcc-38ebb5c1ae56`）再次完成2 normal、2 degraded、
4 restored，且已成功在Root shutdown前複製74,631-byte final artifact；但supplemental protocol-resource collector使用
`docker logs --since <run-start> --tail 500`，Root初始primary edge被後續health／context log擠出tail，因而誤判exact resource
evidence不完整。Run-start到collection本身已是bounded時間窗，collector應讀取完整window並只把包含exact `plan_id`的少量
resource lines寫入`diagnostics/`，不能再用line tail破壞早期evidence。使用該failed run已保存artifact做獨立診斷時，也確認
held-out disposable container雖為`--read-only`，Python bundle loader仍需要temporary directory；修正為保留read-only rootfs，
只掛載64 MiB、`noexec`／`nosuid`的ephemeral `/tmp` tmpfs。修正後同一artifact在GPU上完成200筆MNIST held-out evaluation，
accuracy為0.925；這只是收尾path診斷，不能把failed run補寫成acceptance結果。

第七次run後，component workstream已以PyMTLF `bdbd2a9`及nwdaf-resources `24f8b63`完成final-model persistence
follow-up：Root procedure record現在擁有持久`final-model.tar.gz`與`FINAL_MODEL_SAVED`，local normal及Branch
replacement real-process flow均已通過。這取代本Slice原本從terminal-retained FL workspace直接複製的策略；正式
multi-host testbed仍需使用新PyMTLF image重新驗證。Current testbed runner尚會拒絕新的record type並查找舊workspace
path，因此在重跑前必須完成下表所列adaptation。

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
| Scenario local epoch未進入Leaf node instruction | scenario宣告32，但generated `report_after.count`仍由TESTBED固定為1，PyMTLF Leaf採用後者 | Scenario的`training.localEpochs`是Leaf epoch唯一人工來源；TESTBED Leaf candidate移除`reportAfter`；renderer依role與scenario合成完整native instruction，validator／run evidence核對32 |
| Fault runner讓舊Branch取得cleanup機會 | 第三次run的graceful stop直接傳遞`SIGTERM`；第四次run證明對active unit先mask會觸發Guest backend generation reset；第五次run證明平行stop會讓先死亡的Host backend被仍可執行的Guest觀測，三者都在fault完成前刪除Leaf subscriptions | 先在Host仍存在時以`SIGSTOP`凍結exact Guest unit cgroup，再凍結exact Host container；確認兩個application owners都不可執行後，才安裝Guest runtime mask並對兩者送出`SIGKILL`。一般shutdown仍使用graceful lifecycle並解除mask |
| 人為控制replacement preparation會改變自然恢復timing | Root已在背景準備replacement，且request／callback有bounded deadline | 不pause、blackhole或代理request；保留direct production path並觀測實際degraded round數 |
| Testbed final handoff仍依賴暫存workspace | Current runner拒絕`FINAL_MODEL_SAVED`並從`workspace_root`查找`ROUND_GLOBAL`；PyMTLF `bdbd2a9`已改由Root procedure record持久保存final model | Runner接受並驗證exactly one `FINAL_MODEL_SAVED`，從configured Root `experiment_recording.directory/<planId>/final-model.tar.gz`原樣收集，以round／digest／size和terminal交叉核對，再以同image與bounded ephemeral `/tmp` tmpfs執行GPU held-out evaluation |
| Training與collection目前不可獨立重試 | Current foreground runner在同一控制流完成training、copy、evaluation與reset；collection defect可能觸發cleanup並失去可重試來源 | 在`COMPLETE`與`FINAL_MODEL_SAVED`後先保存training-complete checkpoint；預設繼續collection，但失敗時保留run directory與Root experiment-record volume。提供同一runner的collection-only入口，以相同`RUN_NAME`解析既有`requestId`／`planId`並重試，不重新提交training |
| Host PyMTLF image仍是舊revision | 最近real run的OCI revision為`579f7375…`，不含final-model persistence | 後續任何測試前從nested PyMTLF `bdbd2a9`重建image；preflight與run metadata exact確認source／OCI revision，禁止沿用舊image充當驗證 |
| 現有status偏向log-derived milestones | protocol mode不解析legacylog，但尚無完整evidence collector；第七次run證明在run-start bounded window上再套固定line tail會遺失早期primary edge | Runner增量讀Root JSONL；一般logs只在failure、timeout或特定protocol問題時按需保存bounded時間窗。Protocol resource supplemental evidence讀取完整run window後只保存exact plan lines，不使用會截斷必要event的固定tail |

## 3. 固定 Slice 3 contract

### 3.1 Run 數量與 training workload

新增`experiments/protocol-hierarchical/branch-replacement/mnist/scenario.yaml`與
`experiments/protocol-hierarchical/branch-replacement/cifar10/scenario.yaml`。兩者沿用Slice 2已驗證的dataset split與
模型contract：seed `42`、每Leaf 8,000 training samples、Root 200 validation samples、200 held-out samples、batch size
16、learning rate `0.001`、Leaf local epochs 32及`acceptedRounds: 8`。8,000乘32是兩次real-run feedback後核准的
固定flow timing workload；它不建立模型品質、收斂或效能目標。Normal scenarios仍維持每Leaf 100 samples、local epoch 1
與2 accepted rounds。

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
不合法、replacement workload不是每Leaf 8,000 samples／local epochs 32，或round budget無法容納2 normal加1 restored時，
config validation應fail closed。Internal polling維持250 ms；實際fault window由固定training workload提供，不能靠更密集
polling假設跨Host／Guest exact stop能即時完成。Terminal只每30秒輸出heartbeat，兩者不是同一頻率。

Leaf epoch使用下列單一來源與生成規則：

- 每個image scenario的`training.localEpochs`都是必填正整數；normal scenarios明確寫`1`，replacement scenarios寫`32`，
  不再依賴renderer內部default。
- TESTBED的Branch candidates保留`reportAfter`，且必須是正整數`count`與`unit: round`；本Slice不改變Branch聚合週期。
- TESTBED的Leaf candidates必須省略`reportAfter`；若仍提供，schema validation直接拒絕，不能讓它覆蓋scenario。
- Renderer對每個Leaf以scenario的`training.localEpochs`產生`report_after.count`，並依Leaf role產生
  `report_after.unit: epoch`。Generated server `client_training.epochs`若存在，也必須由同一scenario值產生。
- Generated topology／native config checker必須確認所有Leaves的完整`report_after`和selected scenario一致，並確認
  Branch `report_after`仍和TESTBED一致。

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

Target必須由selected manifest、`branchGroup: area-a`與current priority ordering解析，並在fail-stop前核對active
identity與原process identity。Fault path必須對兩個原process送出`SIGKILL`，不得以`systemctl stop`、`docker stop`、
`docker compose stop`或其他會先送`SIGTERM`的graceful lifecycle operation作為故障注入。Normal completion、runner
failure teardown與reset仍使用既有graceful／exact-scope lifecycle；hard kill只屬於已確認target的experiment fault。

Guest fail-stop必須經既有approved host-context provider／SSH guard；任何real Vagrant／VirtualBox command都不能在
sandbox內執行。Guest systemd unit目前有`Restart=on-failure`，因此controller先完成兩個exact targets的preflight，
在Host backend仍存在時以`SIGSTOP`凍結exact Guest unit cgroup，再以`SIGSTOP`凍結Host container。只有兩個application
owners都不能執行cleanup後，才安裝Guest runtime mask並依序對Guest與Host送出`SIGKILL`；不得讓任一owner先死亡而另一
owner仍可執行，也不得對仍可執行application code的active unit先mask。Implementation必須驗證原MainPID已消失、unit
沒有新的active MainPID。Host container必須驗證exact container process先進入stopped state，之後由`SIGKILL`退出且
restart policy沒有建立新process。Controller只有在兩個original process都confirmed
dead且沒有自動重啟後，才寫入`BRANCH_PROCESS_STOPPED`；其`recordedAt`必須是兩者皆符合postcondition後的UTC RFC3339
timestamp。任一kill ambiguous、target mismatch、意外重啟或postcondition失敗時，不得操作其他candidate或擴大scope，
run標為failed並保存evidence。

## 4. End-to-end execution flow

### 4.1 準備與admission

1. 檢查affected repositories的actual revisions、dirty flags與component image identity；PyMTLF source必須是`bdbd2a9`或其
   經review的後繼，image必須由該source重建且OCI revision一致。
2. 以選定replacement scenario執行既有`config-create DEVICE=gpu`；不直接修改generated config。
3. 執行common `config-validate`、dataset generate／validate及component-native config／topology loader。
4. 驗證resolved device map為Root + six Leaves CUDA、four Branches CPU，Compose runtime／CDI設定與native config一致。
5. 驗證GPU total／free memory、driver、CDI、Docker NVIDIA runtime、PyTorch CUDA probe與selected ports ownership。
6. 在approved host context啟動或核對四台VM，完成stage／activate、clock skew與selected／active identity checks。
7. 驗證operator `RUN_NAME`並建立exclusive run directory；新training自行產生UUIDv4 `requestId`，在提交前將name／request
   binding原子寫入checkpoint。同一selected config只能有一個active replacement runner，既有run directory不得覆寫。

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

1. 對Area A primary Go NWDAF與PyMTLF先完成exact preflight，依序凍結Guest與Host，確認兩個application owners都
   不可執行後再送出exact `SIGKILL`；確認original processes死亡且未自動重啟後寫入controller event。舊Branch不得先
   執行graceful subscription cleanup，也不得因其中一個owner先死亡而讓另一個owner執行dependency-loss cleanup。
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

1. Root到達`COMPLETE`前必須已產生exactly one `FINAL_MODEL_SAVED`；核對其`roundInd`、`artifactDigest`、
   `artifactFile`及`sizeBytes`和terminal status一致，保存完整phase summary、terminal status、plan identity、selected／active
   config與image identity為training-complete checkpoint，並將run狀態設為`collection-pending`。
2. Collection stage從manifest解析Root experiment-record volume，以read-only方式讀取configured
   `experiment_recording.directory/<planId>/final-model.tar.gz`並原樣複製到run evidence；不再查找`workspace_root`，也不依賴
   Root process或container仍在執行。來源檔案的round／component-native identity／size必須和checkpoint中的
   `FINAL_MODEL_SAVED`及terminal status一致。
3. 使用相同PyMTLF image、GPU runtime、Leaf image workload config及該dataset的獨立held-out `.npz`執行
   `evaluate_image_model.py`；disposable evaluation container使用同一GPU但不和七個training participants並行，且必須由
   runner命名、記錄並`--rm`清除。
4. 在停止或reset前，將Root與controller的必要structured records、直接需要的participant records、runtime／GPU snapshots
   及held-out evaluation依時間寫入`events.jsonl`；不逐一複製所有node JSONL。
5. 以已收集events更新`run.json`的metadata、phase timeline、latency、held-out result與terminal state；CSV與plot留給
   實驗後的離線分析，不屬於runner輸出。
6. 透過existing exact-scope lifecycle停止仍在執行的selected Guest／Host processes；failed primary已停止是合法差異，但
   其他missing／unexpected runtime仍須明確報告。
7. 將cleanup outcome寫入events並finalize `run.json`；two-file schema／consistency checker通過後才把run標為successful。
   四台VM、datasets、images與run evidence預設保留；兩個dataset run之間使用approved exact reset清除NRF／ADRF／volume
   experiment state並驗證seed restoration，不destroy VMs。

Default run入口會依序呼叫training與collection stages，但兩段不能共享只能存在於training call stack的必要state。
Collection stage必須可從run directory與持久volume重新建立全部輸入。若第2至5步失敗，runner可停止仍在執行的selected
processes，但必須將run標為`collection-failed`、保存diagnostics，且不得reset Root experiment-record volume或刪除
training-complete checkpoint。修正後以同一`RUN_NAME`執行collection-only入口，從checkpoint沿用原`requestId`／`planId`；
成功後才進入第6至7步。重試寫入必須
idempotent並採atomic replace，不得重複source events或混入另一個training run。

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
| Training terminal artifact | Root `FINAL_MODEL_SAVED` + procedure-local `final-model.tar.gz` + training status resource | exact定位並核對Root final model；不建立testbed-owned digest |
| Final held-out accuracy | `evaluate_image_model.py` JSON output | 和per-round validation分開 |
| Candidate priority與identity | Selected topology + NRF exact-ID profile + Root events | controller不建立替代selection truth |
| GPU execution | Generated native config、Compose runtime、container CUDA probe、Host `nvidia-smi` | 需同時證明selected intent與actual device可用 |

Resource location、`notifCorreId`、Leaf old-route retirement與ADRF temporary record lifecycle若current JSONL沒有完整欄位，
可從exact component API response或bounded owner log擷取supplemental evidence。這些log只能回答其特定protocol lifecycle
問題，不能取代round cohort或metric JSONL。

### 5.2 Run directory

每個run使用operator-defined safe name作目錄，並在checkpoint保存runner-generated UUIDv4 `requestId`：

```text
runs/protocol-hierarchical/<dataset>/<run-name>/
├── events.jsonl
├── run.json
├── final-root-model.tar.gz      # Root procedure-local final-model.tar.gz的原樣收集副本
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

- operator `runName`、runner-generated UUIDv4 `requestId`、PyMTLF `planId`／`mlCorreId` binding，以及dataset、scenario
  definition、start／finish timestamps與exit status；
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
- final Root artifact path、`FINAL_MODEL_SAVED` identity／size核對、cleanup outcome及任何remaining gap。

Runner在run開始與重要milestone以atomic replace更新`run.json`，cleanup後才寫入final outcome；因此中斷時仍保留目前狀態，
但只有finalized且通過cross-file consistency check的檔案才能構成successful acceptance。

`run.json`的training／collection狀態至少區分`running`、`collection-pending`、`collection-failed`與`successful`。
`collection-pending`或`collection-failed`必須保存足以重建collection輸入的plan、terminal、final-model event、Root volume、
dataset、image及selected／active config identity；它們是retry checkpoint，不是successful acceptance。

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
| Primary只有一個owner完成hard kill，或任一owner自動重啟 | 不記成功stop、不擴大target；保存partial stop／restart狀態供exact recovery |
| Degraded round rejected | 保存attempt但不計8 accepted rounds；在overall timeout內等待後續accepted outcome |
| Replacement preparation timeout／error | 不手動指定candidate或重建resource；run failed |
| Persisted final-model record在training完成前缺少或不一致 | training不能建立有效checkpoint或升格為`COMPLETE` evidence；保存可取得raw evidence，不回退至暫存workspace |
| Final-model copy、evidence finalization或held-out evaluation失敗 | 將run標為`collection-failed`，保存training checkpoint與Root experiment-record volume；修正後以同一`RUN_NAME`解析既有component identities並單獨重試collection，不重新訓練 |
| Runner被中斷 | 收集best-effort bounded evidence、exact停止本次selected processes；無法確認scope時停止自動cleanup |
| Stop或reset發現unexpected runtime | 正常destructive path拒絕；需要fresh inventory與另外的使用者批准 |

Training或protocol flow失敗時，retry以整個dataset run為單位。不得從中間round續跑、編輯JSONL，或把兩次partial run
拼成一份8-round evidence；重新training使用新的`RUN_NAME`，runner隨之產生新的`requestId`。只有training已完成且
checkpoint有效時，collection／held-out的retry才沿用同一`RUN_NAME`、原`requestId`與同一Root procedure record，且不得
再次送出training request。

若中斷發生在fault注入前，可在exact stop與reset後用相同generated config、不同`RUN_NAME`重新啟動完整flow。Primary已
fail-stop後不支援只restart該Branch、從中間round繼續或hand back；只能先保存evidence，再依selected exact scope停止、
reset並從round 1重跑。

## 7. Implementation impact map

### 7.1 `5G_NWDAF_Infrastructure` production changes

預期修改下列既有owners；實作時若需要額外top-level directory、service、daemon、external dependency或component change，
屬decision gate：

| Owner | Planned change |
| --- | --- |
| `testbed.protocol-hierarchical.yaml` | Root／six Leaves標示CUDA；Branches保持CPU；GPU free-memory floor；Branch candidates保留`reportAfter`，Leaf candidates移除該欄位 |
| normal MNIST／CIFAR scenarios | 移除重複的`training.device`，明確加入`training.localEpochs: 1`並維持2-round normal smoke語意 |
| new Branch-replacement scenarios | dataset-specific 8-round workload、Area A fault group、2 normal／1 restored minimum、poll／heartbeat behavior |
| `scripts/host/configlib.py` | 驗證scenario fault contract、所有image scenario必填的positive `training.localEpochs`、Branch `reportAfter`與Leaf禁止`reportAfter`的role-specific schema、device ownership及derived runtime inventory |
| `scripts/host/config-render.py` | 由per-service device產生Root／Leaf config，並依Leaf role與scenario local epoch合成完整generated Leaf `report_after`；保留replacement direct endpoint、configured deadlines與既有persistent experiment-record directory |
| `scripts/host/config-check.py` | exact混合device、scenario phase、Leaf native `report_after`、Branch `report_after`、deadline、native config與manifest checks |
| `scripts/host/ml-compose-check.py` | 驗證GPU runtime／CDI與existing exact ports，不建立replacement proxy path |
| existing preflight／status helpers | 保存七個GPU participants與actual CUDA evidence；不另建第二套capacity path |
| one domain-specific experiment runner | operator `RUN_NAME`到runner-generated UUIDv4 `requestId`的持久binding、incremental observation（包含`FINAL_MODEL_SAVED`）、exact hard fail-stop、restart postcondition、natural phase classification、training-complete checkpoint、可獨立重試的persistent-volume collection、held-out evaluation與cleanup |
| `Makefile`／operator docs | 提供以`RUN_NAME`操作、預設串接兩stage的`fl-branch-replacement-run`，以及以同名checkpoint重試的`fl-branch-replacement-collect`入口與簡短說明；既有generic `RUN_ID` contract不變，不以Slice名稱命名production command |

Runner與其support code必須以durable domain semantics命名；implementation artifact、test、function、fixture、log與runtime
label不得以`slice3`、plan title或review work item作identity。

### 7.2 Tests

- Existing config／renderer suite加入normal CPU fallback及replacement GPU mixed-device cases；不另建只檢查literal token的
  meta-test。
- Existing schema／renderer suite驗證Branch `reportAfter`仍為positive round instruction、TESTBED Leaf `reportAfter`被拒絕、
  normal／replacement scenarios分別以唯一`training.localEpochs`產生Leaf native count 1／32與`unit: epoch`。
- Existing lifecycle／capacity suites覆蓋GPU missing、memory insufficient、port collision、wrong config、unexpected runtime
  及partial cleanup。
- 新runner是distinct owner與state machine，可建立一份domain-named behavioral test module；tests直接驅動fake JSONL、
  process owners與clock，驗證contract-defined transitions，而非只檢查檔案或helper存在。
- Runner behavioral tests至少涵蓋兩個exact owners皆收到`SIGKILL`、graceful stop不得充當fault、systemd／container
  restart postcondition、partial kill、failure detection、replacement於下一round前或round中ready、自然degraded count、
  insufficient restored rounds、timeout與runner shutdown。
- Evidence parser tests涵蓋8-round成功、variable degraded count、rejected attempt不計數、ready不等於contribution、
  `FINAL_MODEL_SAVED` round／digest／size和terminal一致、missing／duplicate final-model record、missing evaluation、duplicate
  outcome、invalid timestamp與partial evidence。
- Training／collection boundary tests涵蓋checkpoint先於collection、collection failure不reset experiment-record volume、
  collection-only不提交training、stopped Root container仍可read-only讀取volume、same-run retry idempotence及成功後才reset。
- Run identity tests涵蓋safe `RUN_NAME`、path traversal拒絕、新training同名拒絕、UUIDv4 `requestId`在request前持久保存，
  以及collection-only從checkpoint沿用既有`requestId`／`planId`而不產生新identity。
- Provider guard只能使用synthetic fixtures；不得為測試sandbox blocking而執行real Vagrant／VBoxManage。

## 8. Existing-flow disposition

| Baseline stage | Slice 3 disposition |
| --- | --- |
| TESTBED／scenario selection | Adapted：同一selector與schema family；新增replacement scenarios，device truth只在TESTBED；Leaf epoch只在scenario的必填`training.localEpochs`。`RUN_NAME`只定位run checkpoint，不是deployment selector |
| Config generation | Adapted：同一renderer依Leaf role與scenario epoch產生完整native `report_after`及混合GPU、8-round fault config，不建平行renderer或proxy path |
| Validation | Adapted：common validation全部保留，再加入mixed device、role-specific `reportAfter` ownership、Leaf local epoch、phase與deadline checks |
| Dataset acquisition／partition | Reused without semantic change：沿用Slice 2 cache、deterministic split與native checks |
| VM／network／stage／activate | Reused without semantic change：同四VM、同provider guard與active identity |
| Guest／Host startup | Reused without semantic change：eleven one-to-one runtimes與direct endpoints不變 |
| Training trigger／status | Reused and composed：沿用private API與existing FL control contract，runner只編排 |
| Fault injection | New in this flow：以`SIGKILL` hard fail-stop兩個exact primary owners、阻止自動重啟且不執行graceful subscription cleanup；controller不修改production code或priority |
| Observation | Adapted：node JSONL為authority，incremental quiet monitor與bounded diagnostic logs |
| Final evaluation | Adapted：複用component evaluator與held-out dataset，從持久Root procedure record使用exact terminal artifact；可和training分開重試 |
| Stop／reset／recovery | Adapted：先保存training checkpoint與evidence；collection成功前保留Root experiment-record volume，成功後才走selected-and-active exact reset scope |
| Legacy／normal scenarios | Reused／isolated：normal 2-round profiles保留；legacy profiles不驗收GPU或replacement |

## 9. Verification matrix

| Requirement | Static／synthetic evidence | Required real-environment evidence |
| --- | --- | --- |
| Single source與scenario ownership | schema／render tests確認scenario無device且必填positive `training.localEpochs`、TESTBED擁有service device／topology／Branch `reportAfter`但Leaf不得定義`reportAfter`，renderer產生完整Leaf native instruction | `run.json`顯示selected scenario、generated Leaf count 32／unit epoch與actual local work一致 |
| Mixed GPU mapping | exact native／Compose／manifest checks | Root + six Leaves CUDA visible；four Branches CPU；無fallback／OOM |
| GPU capacity | missing CDI/runtime/memory fixtures | `events.jsonl`保存RTX 3080 snapshot、8,192 MiB admission與seven participants ready |
| Natural replacement timing | runner state-machine與deadline contract tests | direct endpoint、observed degraded count、configured deadline與ready timing |
| Exact primary fail-stop | fake process owner／freeze ordering／signal／restart／partial-stop tests | Area A primary Go+PyMTLF先依序凍結且都無cleanup機會，再因`SIGKILL`死亡且未重啟，其他regions仍active |
| Priority selection | topology／manifest candidate ordering tests | Root events與NRF profile證明primary 100失效後選replacement 50 |
| 8-round phase integrity | JSONL parser/state-machine fixtures | MNIST與CIFAR-10各一個完整8-accepted-round run，2 normal後fault且至少1 restored |
| JSONL／evaluation consistency | duplicate／missing／rejected fixtures | 每個accepted outcome exactly one Root global evaluation |
| Replacement lifecycle | ready-versus-contribution fixtures | failure detected、ready及first contribution形成至少1個restored round |
| Final held-out path | training／collection checkpoint、persistent-volume collector、`FINAL_MODEL_SAVED`／terminal consistency及evaluator command tests | Root persistent final artifact在GPU上完成獨立held-out evaluation，workspace cleanup及Root process停止後仍可讀取；collection failure可用同一`RUN_NAME`沿用checkpoint identity重試且不重訓 |
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
| S3-02 | Scenario只擁有dataset、training、fault與observation behavior，不保存device或participant identity；所有image scenarios明確提供Leaf epoch唯一人工來源`training.localEpochs` | four image scenarios與schema tests |
| S3-03 | `DEVICE=gpu`解析為Root與six Leaves CUDA、four Branches CPU | TESTBED、manifest、native config、Compose、real status |
| S3-04 | GPU acceptance使用one RTX 3080、8,192 MiB pre-start floor與seven-participant actual evidence | preflight與`events.jsonl`／`run.json` |
| S3-05 | Branch `reportAfter`只由TESTBED提供；TESTBED Leaf candidate不得定義`reportAfter`。Normal scenarios明確提供2 accepted rounds／`localEpochs: 1`，replacement scenarios固定8 accepted rounds／`localEpochs: 32`；renderer依Leaf role與scenario產生完整native `report_after` | role-specific schema rejection、normal／replacement render與native-config tests |
| S3-06 | MNIST與CIFAR-10各只有一個正式replacement flow run | run coverage checker與record wording |
| S3-07 | Replacement沿用direct production endpoint與configured deadlines，無gate／proxy／extra port／detached state | config review、runner tests與actual process／port inventory |
| S3-08 | Controller不控制replacement preparation timing；degraded count由stop、Root outcomes與first contribution推導 | `events.jsonl`與`run.json` |
| S3-09 | Fault target由Area A group與selected priority解析；先依序凍結exact Guest與Host，兩者都不能執行cleanup後才以`SIGKILL` hard fail-stop primary Go+PyMTLF，且不得自動重啟 | manifest resolver、cross-owner freeze ordering、signal／restart postcondition、controller JSONL |
| S3-10 | Controller不指定replacement、不修改priority、不偽造component event | call-path review、Root／NRF evidence |
| S3-11 | 每run固定8 accepted rounds，在2 normal後fault、記錄natural degraded count並至少完成1 restored | two-file phase checker與`run.json` |
| S3-12 | Ready與first contribution分開判定；replacement只加入尚未dispatch cohort | lifecycle events與successful IDs |
| S3-13 | Rejected attempt不計phase且不得有Root global evaluation | parser tests與actual consistency check |
| S3-14 | Same `mlCorreId`、new per-edge resource／`notifCorreId`、same Area A Leaves，無retained result | structured／bounded protocol evidence |
| S3-15 | Final artifact由Root procedure record持久保存；runner在`COMPLETE`與`FINAL_MODEL_SAVED`後建立training checkpoint，再以persistent-volume collector核對terminal component-native identity／round／size。Collection／held-out可用同一`RUN_NAME`解析原`requestId`／`planId`後獨立重試且不重訓，不依賴暫存workspace或running Root container，held-out與validation分離 | training／collection state machine、persistent-volume collector、retry tests、event／terminal consistency、evaluator JSON、dataset manifest |
| S3-16 | Root JSONL是round／metric authority，必要records整合到`events.jsonl`；一般log只回答bounded diagnostics | collector、two-file checker與review |
| S3-17 | Monitor增量讀取且terminal只輸出milestones／30-second heartbeat | runner tests與actual transcript |
| S3-18 | Training checkpoint與可取得evidence在stop／reset前保存；collection失敗保留same-run retry source與unique directory | failure-path、retry tests與run inventory |
| S3-19 | Stop／reset維持selected-and-active exact scope與provider host-context safety；Root experiment-record volume只在collection成功後reset | lifecycle tests與approved-host evidence |
| S3-20 | 不新增testbed-owned hash／checksum／digest chain；只觀測既有Docker與component-native identity | semantic diff review與`run.json` schema |
| S3-21 | 所有implementation／test artifacts使用durable domain identity，不含Slice／review tracking identity | artifact review與test quality review |
| S3-22 | 兩個run只證明flow acceptance，不宣稱模型品質、dataset優劣或controlled comparison | `run.json`／record language review |
| S3-23 | 後續所有testbed tests與real runs使用包含final-model persistence的目前PyMTLF source及重建image，不接受舊OCI revision | repository／image preflight、container checks與run metadata |
| S3-24 | Domain-specific runner接受operator-defined safe `RUN_NAME`，新training產生並持久綁定UUIDv4 `requestId`，collection-only沿用checkpoint identity；不改變PyMTLF與generic control的UUID contract | identity validator、exclusive-directory、request submission與collection retry tests |

## 11. 明確非目標

- CPU acceptance替代GPU run，或CPU／GPU performance comparison。
- GPU utilization、energy、communication bytes、network cost或long-term telemetry instrumentation。
- 修改PyMTLF device resolver、CNN architecture、FedProx、aggregation或Branch `reportAfter`。
- 超出已核准每Leaf 8,000 samples／local epochs 32的workload、增加seeds或runs來研究accuracy／convergence。
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
2. 從normal scenarios移除device duplication、明確加入`training.localEpochs: 1`，並維持兩份8-round replacement
   scenarios的`training.localEpochs: 32`。
3. TESTBED保留Branch `reportAfter`並移除Leaf `reportAfter`；更新renderer與validators，使scenario local epoch成為
   generated Leaf native `report_after`的唯一人工來源。
4. 更新manifest、native config、Compose及validators的mixed-device／phase／deadline semantics。
5. 完成focused config、role-specific schema rejection、normal／replacement render、capacity、normal regression及
   container GPU checks。

### Work unit B — Controller與evidence pipeline

1. 實作domain-specific foreground runner、exact target resolver與natural replacement phase classifier。
2. 複用existing FL control、provider guard、Compose及lifecycle owners。
3. 實作incremental JSONL phase state machine（接受並驗證`FINAL_MODEL_SAVED`）、training-complete checkpoint、two-file
   evidence writer、按需bounded diagnostics，以及不依賴running Root container的persistent-volume collector與held-out
   evaluation；移除暫存workspace lookup。
4. 以operator `RUN_NAME`建立exclusive run checkpoint並在request前持久保存runner-generated UUIDv4 `requestId`；預設run
   入口串接training與collection，另提供same-`RUN_NAME` collection-only retry。後處理失敗時保留Root record volume且
   不重新提交training、不產生新request identity，成功後才允許exact reset。
5. 完成process failure、variable degraded phase、deadline、checkpoint／retry、evidence consistency與cleanup tests。

### Work unit C — Real dual-dataset acceptance

1. 在approved host context完成MNIST GPU replacement run。
2. Review evidence、stop、exact reset並驗證seed restoration。
3. 重新render CIFAR-10 selected config，完成CIFAR-10 GPU replacement run。
4. 執行mandatory initial review、finding admission／remediation、final fresh-read conformance與required full verification。
5. 保持changes unstaged／uncommitted，交付affected repositories、完整diff、verification、remaining gaps及conformance供
   使用者review。
6. User review與commit approval分開；commit後仍需另行取得push approval。兩個run evidence通過review後，才建立
   flow-acceptance record並提議完成主計畫。

### 2026-09-13 implementation與verification evidence

Slice 3 implementation已在`5G_NWDAF_Infrastructure`完成，component repositories保持read-only。兩個正式run結果如下；
held-out accuracy只證明獨立evaluation path可執行，不作模型品質或dataset比較：

| Dataset | `RUN_NAME` | `requestId` | `planId`／`mlCorreId` | Phase counts | Final artifact | Held-out path | Cleanup |
| --- | --- | --- | --- | --- | --- | --- | --- |
| MNIST | `mnist-replacement-20260913-b` | `2aad3ece-cd8e-4dcf-b10f-cbb661fe7ec8` | `768e68dd-e375-4e92-9e2d-3f2519b28fc2` | 2 normal、2 degraded、4 restored | 74,634 bytes | 185／200，accuracy 0.925 | process stop、Guest restart policy restoration與exact reset verified |
| CIFAR-10 | `cifar10-replacement-20260913-a` | `3a090f6f-bb27-4f89-8569-d71c8279f5a5` | `a6520a53-edc1-47cf-abda-0404012c6137` | 2 normal、2 degraded、4 restored | 76,695 bytes | 100／200，accuracy 0.5 | process stop、Guest restart policy restoration與exact reset verified |

兩個run皆使用PyMTLF image revision `bdbd2a9591e47d59b690f0031dfebc77f757bb91`、one RTX 3080與七個
CUDA participants；四個Branches維持CPU，未發生OOM或silent fallback。兩者從confirmed stop到failure detected、
replacement ready與first contribution的時間分別約為MNIST 256.134／257.002／373.649秒，以及CIFAR-10
256.226／257.106／373.606秒。每次replacement皆從下一個尚未dispatch的cohort開始貢獻，因此自然觀測到兩個
degraded rounds。

Real verification期間關閉三個current-slice findings：

1. collection已停止runtime後，外層failure path可能再次stop；加入checkpoint-aware single-stop behavior與直接測試。
2. startup lifecycle合法重建image時，runner錯把pre-build與post-build image ID不同視為stale runtime；調整為啟動前核對
   source revision，啟動後再核對running containers、post-build tag與同一revision。
3. Root volume collector產生的run-local artifact為`0600`，使UID 10001 evaluator無法讀取；保持artifact bytes不變，將
   本地副本設為read-only可讀，並同時覆蓋首次collection與same-run retry。

MNIST在第三項修正前已完成training並停在`collection-failed`；修正後以同一`RUN_NAME`、原`requestId`與原`planId`
執行`fl-branch-replacement-collect`成功，沒有重新提交training，也只在two-file checker通過後reset。CIFAR-10隨後以
default training→collection path一次完成，證明修正不只適用retry。

Final verification結果：

- `make test`通過全部repository synthetic checks，並明確確認未啟動provider或service process。
- MNIST與CIFAR-10的`config-create DEVICE=gpu`、dataset generation、`experiment-validate`與real multi-host run全部通過。
- Canonical eleven-container GPU boundary已由兩個real runs直接驗證。保留但不維護的legacy Flat／static HFL
  `make test-containers`因舊seed-artifact digest和目前PyMTLF seed不一致而失敗；依上層計畫的approved deferral，這不是
  canonical regression，也未加入compatibility workaround。
- 兩個run結束後，containers與experiment state完成exact cleanup；四台VM最後由guarded lifecycle關閉，provider status皆為
  `poweroff`。Datasets、images、VM disks與ignored run evidence保留。

Mandatory initial review及每個remediation的targeted follow-up review均未留下admitted current-slice finding。使用者於
2026-09-13確認review結果；正式摘要已保存於
[Protocol-driven Branch Replacement Validation Record](../../../records/hierarchical-federated-learning/protocol-driven-branch-replacement-validation-2026-09-13.md)，
因此本Slice標為`Completed`。Implementation與documentation commits仍等待獨立commit proposal批准。

### Slice acceptance

- Implementation artifacts、tests與operator docs完成S3-01至S3-24，且無未處理admitted finding。
- MNIST與CIFAR-10各有一個GPU real run，皆完成8 accepted rounds、2 normal後fault、記錄natural degraded count並至少
  完成1 restored round。
- Root依priority選出replacement；controller只以`SIGKILL` hard fail-stop primary並確認未重啟，不控制replacement preparation timing。
- TESTBED Branch `reportAfter`維持不變、Leaf candidates不含`reportAfter`；replacement runs的generated Leaf
  `report_after`為`count: 32`／`unit: epoch`且actual local work為32，normal scenarios明確產生count 1。
- 七個CUDA participants有actual evidence，四個Branches維持CPU，無silent fallback或OOM。
- Focused、container與兩個real runs均使用包含final-model persistence的目前PyMTLF revision及由該source重建的image；
  run metadata不得仍顯示舊`579f7375…` OCI revision。
- 每個accepted outcome與Root evaluation一致，replacement ready／first contribution明確分開。
- Final held-out evaluation、`events.jsonl`、`run.json`與從Root procedure record收集的final Root artifact完整；
  `FINAL_MODEL_SAVED`和terminal status一致，workspace cleanup後artifact仍可用；必要的bounded logs保存於optional
  `diagnostics/`，CSV／plot不屬於acceptance。
- Operator可以`RUN_NAME=123`啟動新training；runner自行產生並持久保存UUIDv4 `requestId`，不改變PyMTLF contract。
- Default run雖串接training與collection，training-complete checkpoint仍允許collection／held-out failure在不重訓下以
  同一`RUN_NAME`沿用原`requestId`／`planId`重試；collection成功前Root experiment-record volume不得reset，且
  collection-only不得提交training request或產生新identity。
- Evidence先保存，再完成process stop、ADRF／NRF／volume reset與seed restoration；四台VM可在兩run之間保留。
- Required real evidence、mandatory review及使用者確認均已完成，並已建立verified record；本Slice標為`Completed`。
