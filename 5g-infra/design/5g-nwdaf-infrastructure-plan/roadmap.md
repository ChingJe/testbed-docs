# 實作 Roadmap、Gate 與驗證

[返回主計畫](../5g-nwdaf-infrastructure-plan.md)

## 11. 實作階段與 Gate

### Phase 0 — 決策與文件

Deliverables：

- 本計畫；
- 舊 site/network inventory；
- repository 名稱、owner、visibility、license、deployment placement 與 virtualization
  provider 待決清單。

Exit gate：使用者確認計畫、第一版 component scope 與預計建立的 repositories／branches。

### Phase 0.5 — 舊 VM 保存與本機清理

此階段位於新 repository bootstrap 與新 VM 建立之前，只清理 provider VM／disk，不刪除
舊 `5G_Infrastructure` source repository。

2026-08-06 的清點與 removal report 見
[legacy-vm-inventory-and-removal-proposal-2026-08-06.md](../../reports/legacy-vm-inventory-and-removal-proposal-2026-08-06.md)。
使用者明確放棄 VM backup 後，五台舊 VM、orphan saved-state 與其殘留 process／directory
已移除；舊 source working tree 保留，NVMe available space 增加到約 178.24 GiB。

Deliverables：

- 唯讀列出所有 Vagrant registered machines、provider VM、UUID、project directory、
  storage path、virtual disk、snapshot、power state 與實際占用；
- 將主要舊 `5GC`、`UPF-EES`、`UPF-EES2`、`gNB`、`gNB2` 和其他被發現的 legacy VM
  分成「必須備份後移除」、「確認不需備份即可移除」與「不在本次範圍」；
- 保存仍只存在 guest 的 config、script、database、log 或 evidence；若 VirtualBox driver
  尚不可用，選擇 raw VM directory copy、修復後 export 或其他可驗證的外部備份方式；
- 產生 backup manifest、hash、儲存位置與 recovery 說明；
- 產生 exact VM name／UUID／disk removal list，另行取得使用者最終確認；
- 以原 Vagrant project 為優先逐台 destroy，必要時才使用 provider-specific
  unregister/delete；每台完成後檢查 Vagrant／provider state 和 reclaimed space。

Exit gate：使用者確認的舊 local VM 已移除、backup 可定位且有 recovery 說明、舊 source
repository 仍完整保留；若新 VM 使用同一 filesystem，free space 達到 preflight 設定的
安全 threshold。此計畫文件本身不構成刪除命令授權。

### Phase 1 — Repository Bootstrap

Deliverables：

- 建立本機 `5G_NWDAF_Infrastructure` repository 與初始 `main`；remote、owner 與 visibility
  在公開準備時再設定，不作為本機實作前置；
- README、LICENSE、精簡目錄骨架、單一 multi-machine `Vagrantfile`、
  `testbed.yaml`／local example、ignore policy 與 non-mutating preflight；
- 第一版不先建立空的 `tests/`、`fixtures/`、頂層 `patches/`、`profiles/`、`sites/`、
  `provisioning/`、`deployment/` 或 `experiments/`；
- CI 先驗證文件、submodule metadata 與 script static checks。

Exit gate：fresh clone 不需要實驗室私有設定即可讀取文件並通過 metadata preflight。

### Phase 2 — Component Pinning

Deliverables：

- 加入第 6 節 required submodules；
- source manifest 與 license inventory；
- 固定 upstream／team exact commits；
- optional webconsole；
- PyAnLF 固定至少包含 `9e64417` 的 configurable CUDA inference；
- PyMTLF 固定至少包含 `e9c5b08` 的 configurable CUDA training；
- 確認匿名 HTTPS clone readiness。

Exit gate：`git submodule status --recursive` clean，所有 required component 可取得、
revision 可追溯且 license 無未解阻塞。

### Phase 3 — Component Config 與必要工具移植

Deliverables：

- 匯入／整理 `config/default/` 下可直接執行的完整 core、path-a、path-b 與單一 consumer
  native config set，附來源與 topology manifest；
- 定義 `testbed.yaml` topology schema、`testbed.local.yaml` 允許的 host-only override 與
  active `configDir`，包含 Host ML published endpoints／device placement，並驗證它們和
  component config 的 network／placement 一致；
- 實作 optional `config-render.py`，只寫入 gitignored `config/generated/<name>/`；實作同時
  驗證 default／generated／local set 的 `config-check.py`；
- 盤點 `nwdaf-resources` 中與公開 VM 環境直接相關的 preflight、subscriber/group
  provisioning 與 config preparation；只移植或重寫必要小工具；
- 以既有 `nwdaf_uecomm_consumer` 為來源，整理 infrastructure-owned consumer CLI、
  callback reachability、transient subscription state 與 provenance；
- 移植時記錄原 repository、source commit 與 license，並明確指定新 owner；不複製
  整個 repository，也不讓新環境依賴它的 workspace layout；
- `nwdaf-resources` 保留既有 isolated／single-host regression，是否清理重複工具屬於
  後續獨立變更。

Exit gate：新 repository 不 clone `nwdaf-resources`、不執行 renderer，也能以 default 完成
non-mutating preflight 與準備三 VM；renderer 輸出和手工 local set 都通過相同 consistency
check，任何移植工具的 provenance 與 ownership 清楚。

2026-08-09 implementation record：`testbed.yaml` 已將 Core／Path A／Path B 固定為
4096／3072／3072 MiB、三顆 40 GiB dynamic logical disk，並將五個 Python backend placement
移到 Host containers。Default native config 與 renderer 已分離 container bind address
（`0.0.0.0`）和 VM-visible advertised endpoints；checker 會驗證 placement、port、device、
callback/artifact URL、PseudoDriver profile／guest active path 與 dataset contract。Preflight 以
10 GiB VM allocation 加 6 GiB Host reserve 作 RAM hard gate，swap 不足改為 warning；Compose
lifecycle 仍屬 Phase 6.5，這筆紀錄不宣稱 container 或 VM 已啟動。

### Phase 4 — 三 VM Skeleton 與 Network

Deliverables：

- provider 選型與 prerequisite check；
- root multi-machine `Vagrantfile` 內的 `core`、`path-a`、`path-b` VM；
- management、SBI、N2、N3-A/B、N4、N6-A/B plane；
- Host SBI address 到三台 VM 的雙向 reachability，以及五個 ML published port 的 conflict／
  firewall smoke；
- `testbed.local.yaml` override 與 public isolated default；
- `.vagrant/` metadata、provider VM disk 與 Vagrant box cache 的位置／free-space 回報；
- route、forwarding、NAT、MTU、port reachability smoke tests。

Exit gate：三 VM 可由 fresh testbed definition 重建；每個 plane 只具有設計允許的 reachability，
且不需要實驗室實體 bridge。

### Phase 5 — Core Guest Setup 與 Build

Deliverables：

- reproducible guest-local build/artifact manifest；
- `common.sh`／`core.sh` 與 Core 的 NF、MongoDB、ADRF、NWDAF-C；
- optional webconsole setup；
- start order、health、status 與 bounded cleanup。

Exit gate：required core NF、ADRF 與 NWDAF-C process 在不要求 ML backend ready 的層級健康，
NRF registration identity 符合本次 manifest，restart 不產生非預期 duplicate state；完整
analytics/model health 留到 Phase 6.5。

### Phase 6 — Path A／B Guest Setup 與 Build

Deliverables：

- `path-a`／`path-b` 由同一 pinned source 各自完成的 guest-local UERANSIM build；
- guest-specific gtp5g build；
- `common.sh`／`path.sh A|B` 與 role-local source/config placement；
- UPF-A/B、gNB-A/B、six UEs；
- NWDAF-A/B；
- topology-derived PseudoDriver Path A/B dataset、semantic manifest、guest atomic staging、
  systemd memory accounting 與 bounded direct replay smoke；
- TAI-aware UPF selection 與 subscriber/group test data provisioning。
- 三台 VM clock-skew check、`observe.sh` compact status 與可過濾的跨 VM journald live
  follow。

Exit gate：

- six UEs 都建立 current-run AMF／SMF registration；
- TAI `000001` UE 只取得 Path A pool，TAI `000002` UE 只取得 Path B pool；
- two UPFs 形成正確 PFCP association，兩條 N3/N6 path 可被獨立觀察；
- A/B replay 使用預期 dataset identity，warm-start 前至少保有 512 MiB `MemAvailable`，且
  scan/aggregation/Phase 2 不造成 OOM、強制 kill 或不可接受 reclaim；
- A／B／C NWDAF role 與 scope discovery 正確。

### Phase 6.5 — Host ML Containers 與 GPU Runtime

Deliverables：

- parent 更新 PyAnLF／PyMTLF gitlinks 至已推送的 GPU-capable commits；
- 共用 Python 3.12、PyTorch 2.5.1＋CUDA 12.1 layers 的 PyAnLF／PyMTLF images；
- 一個 `compose.yaml` 定義五個獨立 service，不把多個 process 塞進同一 container；
- PyAnLF-A/B 預設 CPU、PyMTLF-A/B 使用 `cuda:0`、PyMTLF-C 使用 CPU 的 explicit device
  placement；
- Docker／driver／CDI preflight；toolkit installation 與 NVIDIA runtime registration 是另行
  授權的 Host prerequisite，不由一般 bring-up script 自動修改系統；完成後以 reload 套用
  runtime、驗證 CDI inventory、disposable GPU probe 與 Compose resolution，不 restart shared
  daemon，也不改變 `default-runtime`；
- Host published endpoints、VM-to-Host reachability、read-only config mount、health、source／
  lock／image identity 與 bounded log rotation；
- `ml-start`／`ml-status`／`ml-stop`，並讓 `observe`／`logs` 同時涵蓋 guest 與 container；
- RTX 3080 實機 smoke：在 full-core 路徑直接執行一次 bounded A/B concurrent training，記錄
  peak VRAM／OOM 行為；只有 concurrent 失敗時才拆成單 client 診斷。另驗證 PyAnLF CPU
  inference，以及 PyMTLF-C CPU FedAvg／model serialization。

Exit gate：五個 endpoint 可由所屬 Go NWDAF 到達，status 能證明 effective device 與 image
identity；A/B GPU path 不是 silent CPU fallback；同時 training 若超過 10 GiB VRAM，必須有
明確 sequential scheduling／failure policy。完成前不移除既有 guest ML setup，通過後才以
獨立 commit 清除過時 provisioning、systemd units 與 guest disk assumptions。

### Phase 7 — Full Scenario E2E

Deliverables：

- 在新 repository 內實作最小 VM-aware full-core analytics／FL E2E；可參考
  `nwdaf-resources` assertion，但不把該 repo 變成 runtime dependency；
- 由單一 Core consumer 透過 NRF 找出兩個不同的 Events Subscription providers，再對
  NWDAF-A/B 建立同 group、不同 TAI 的 `UE_COMMUNICATION` subscriptions，保存各 path
  subscription start，並以兩側都成功作為共同 action barrier；
- stable A/B baseline、A-only degradation、automatic FL trigger；
- 實作 `full-core-cat-transition` 主 example：900 秒 warm-start 只填滿 PyAnLF input，保留
  舊 testbed 的 15 分鐘 live CAT1→CAT2 與 optional CAT2→CAT3 時間語意；
- 實作獨立 `fl-closure-smoke`：以 3000 秒 historical burst 與較少的 reference／hit policy 快速
  準備 training data，但仍走正常 WAPE trigger、ADRF retrieval 與 A／B participant path；
- two-round FedAvg、validation、ADRF publication、reprovision 與 monitor cutover；
- 證明 Go NWDAF 在三台 VM、五個 ML backend 在 Host containers 時，NRF discovery、ADRF、
  callback 與 model transfer 不依賴 Docker container IP；
- 驗證成功／失敗條件；per-run record 與 archive format 留待後續設計。

Exit gate：主 example 的既有 Phase 7 business assertions 全部通過，summary 同時包含
VM／container／process identity、source revision、config hash、binary/image hash、effective device、sample
count、model identity、ADRF transaction 與 monitor route；不得把 PseudoDriver 宣稱為
真實 application benchmark。Smoke 成功只能證明 bounded FL closure，不可替代主 example
的 production timing、CAT transition 或 business acceptance。

### Phase 8 — Public Release Readiness

第一版公開介面以「固定且可修改的 reference deployment」為目標，不宣稱為任意拓撲框架：

- 正式 reference topology 固定為 Core、Path A、Path B 三台 VM、雙 TAI、五個 Host ML
  containers 與單一 consumer；
- VirtualBox 是第一版唯一正式支援的 provider；使用者文件、example 與 validation 都以
  VirtualBox 為準；
- `466/92`、既有 subnet、TAI、S-NSSAI、DNN、subscriber identity 與資源值是 committed
  建議設定，不是 infrastructure 不可變常數；使用者可從 example 建立完整 config 後修改；
- CPU 或 GPU 是使用者在 config 中明確選擇的 runtime policy。兩者都是正式操作路徑，不能以
  Host 自動偵測結果靜默切換；effective config/status 必須記錄實際 device；
- `full-core-cat-transition` 與 `fl-closure-smoke` 只作為已驗證的 example configs。它們使用
  同一套 lifecycle，不各自擁有 runner 或特殊啟動指令；
- WebConsole 保留為 optional component，加入獨立 build／start／status／stop 與 bounded
  smoke；第一版只驗證 frontend、登入、MongoDB subscriber read path，不包含 billing、TLS
  或 certificate；
- reset 永遠維持獨立的 confirmation-gated destructive operation，不隱含在 experiment start。

#### Phase 8.1 — 公開 command surface

Make targets 分成一般實驗、進階工具與 repository tests。底層 Host／Guest scripts、systemd
units 與 MongoDB helpers 是 implementation details，不列為使用者指令。舊 targets 在新入口
驗證完成前可保留為 compatibility aliases，但不出現在預設 help；確認沒有文件或 automation
依賴後才分批移除。

一般使用者由 `make help` 看到：

| 指令 | 是否改變狀態 | 功能 |
| --- | --- | --- |
| `make help` | 否 | 顯示一般實驗流程與常用指令。 |
| `make experiment-validate CONFIG_DIR=...` | 否 | 檢查 VirtualBox、Docker、submodules／locks、RAM、磁碟、ports、CPU/GPU prerequisites、config contract、Compose wiring 與既有 dataset；不建立 VM、不啟動服務，也不偷偷產生缺少的 dataset。 |
| `make vm-up` | 是 | 建立或啟動三台 VM；首次建立會 provision/build，但不啟動實驗 processes。 |
| `make vm-status` | 否 | 顯示三台 VM 為 running、poweroff 或不存在。 |
| `make vm-halt` | 是 | Graceful shutdown 三台 VM，不刪 VM 或 virtual disks。 |
| `make experiment-start CONFIG_DIR=...` | 是 | 重新驗證必要 gates，產生／驗證／載入 dataset，啟動 Guest services、config 指定的 CPU/GPU ML containers、optional WebConsole、consumer 與兩筆 subscriptions；失敗時只反向停止本次已啟動項目。 |
| `make experiment-status` | 否 | 集中顯示 VM、23 個 Guest services、五個 ML containers、effective device、WebConsole、consumer、subscriptions、config identity 與基本 Host resources。 |
| `make experiment-stop` | 是 | 精確刪除兩筆 subscriptions，停止 consumer、optional WebConsole、ML 與 Guest services；保留 VM、dataset、MongoDB、ADRF/model state、containers、images 與 volumes。 |
| `make logs` | 否 | 依來源、VM、service、時間與 follow mode 查看 Guest、ML、consumer 與 WebConsole logs。 |

一般流程為：

```sh
make experiment-validate CONFIG_DIR=config/local/my-experiment
make vm-up
make experiment-start CONFIG_DIR=config/local/my-experiment
make experiment-status
make logs
make experiment-stop
make vm-halt
```

`experiment-start` 是便利 orchestration，不改變 VM、Guest services、ML containers、WebConsole
與 subscriptions 仍可分開控制的 ownership。它不能自動開機、reset state 或關閉 VM。

#### Phase 8.2 — Config 與 dataset 工具

由 `make help-advanced` 顯示：

| 指令 | 是否改變狀態 | 功能 |
| --- | --- | --- |
| `make config-create NAME=<name> FROM=<example>` | 本機檔案 | 從 committed example 建立可編輯的完整 `config/local/<name>/`；`NAME` 是新 config 名稱，`FROM` 是 `full-core-cat-transition` 或 `fl-closure-smoke` 等起點，省略時使用主要 reference example。 |
| `make config-validate CONFIG_DIR=...` | 否 | 對照指定 config、`testbed.yaml`、manifest-selected scenario、config-specific subscriber fixtures、PyMTLF seed metadata 與所有 NF／ML／UERANSIM configs，驗證 PLMN、TAI、S-NSSAI、DNN、endpoints、callbacks、network aliases、FL policy、timeouts 與 source hashes。只有自訂 topology 時才另傳 `TESTBED=...`。 |
| `make dataset-generate CONFIG_DIR=...` | 本機檔案 | 依 UE pools、traffic profiles、sampling、warm-start、degradation、monitor 與 model shape 產生 content-addressed Path A/B Parquet artifacts。 |
| `make dataset-validate CONFIG_DIR=...` | 否 | 驗證實際 Parquet，而不只驗證 config 欄位；包含 dataset ID、schema、SHA-256、bytes、rows、UE IP、timestamps、breaking time、training／validation capacity 與 bounded trigger／closure。Dataset 不存在時失敗。 |
| `make dataset-show CONFIG_DIR=...` | 否 | 顯示實際 dataset identity、hash、大小、rows、時間範圍、trigger window 與 closure budget。 |
| `make dataset-load CONFIG_DIR=...` | Guest state | 將已驗證的 Path A/B artifacts 上傳到所屬 VM，Guest 再驗證 identity/hash 後原子切換 active dataset。 |

`experiment-start` 會依序執行 generate、validate 與 load；上述 targets 保留給 config 作者與
data-path 除錯。`466/92` 要真正成為可替換的建議值，subscriber/group fixtures 不能繼續是
repository-global fixed files；Phase 8 應由完整 config 產生或包含 config-specific fixtures，
並讓 config identity 同時涵蓋 authentication、subscriber 與 Internal Group input。

#### Phase 8.3 — 分域 lifecycle

分域 targets 是 `experiment-*` 的底層能力，也供進階除錯：

| 執行域 | 指令 | 功能 |
| --- | --- | --- |
| Guest | `services-start CONFIG_DIR=...` | 載入 config/dataset、套用 scoped subscriber data，啟動 MongoDB、5GC NFs、ADRF、三個 NWDAF、兩個 UPF、兩個 gNB 與六個 UE；不啟動 ML、WebConsole 或 consumer。 |
| Guest | `services-status`／`services-stop` | 顯示 23 個 units，或反向停止它們並保持 VM running。 |
| ML | `ml-start CONFIG_DIR=...` | 依 config 明確選擇 CPU 或 GPU，build images 並啟動五個 containers；CPU 路徑不要求 NVIDIA，GPU 路徑 fail-fast 驗證 runtime/CDI/CUDA。 |
| ML | `ml-status`／`ml-stop` | 顯示 health、device、CUDA、memory、image revision、config hash，或停止 containers 並保留 state。 |
| Consumer | `subscriptions-start` | 啟動單一 consumer，經 NRF 找到兩個不同的 TAI-specific NWDAFs 並建立兩筆 subscriptions。 |
| Consumer | `subscriptions-status`／`subscriptions-stop` | 顯示 callback/notification/exact locations，或 DELETE 兩個 exact resources 後停止 consumer。 |
| WebConsole | `webconsole-start CONFIG_DIR=...` | 在 Core build／啟動 optional free5GC WebConsole，使用該 config 的 MongoDB 與 NRF；不成為主要 Guest stack 的 hard dependency。 |
| WebConsole | `webconsole-status`／`webconsole-stop` | 顯示 config enable、unit、HTTP frontend、artifact 與 source identity，或獨立停止 WebConsole。登入與 MongoDB read path 由 bounded smoke 驗證，不隱含在一般 status。 |

WebConsole 是否隨 `experiment-start` 啟動，由完整 config 中明確的 enable flag 決定。現有
`webuicfg.yaml` 只有未驗證 asset，不能在 smoke 通過前宣稱正式可用。

#### Phase 8.4 — Subscriber 與 experiment state

| 指令 | 是否改變狀態 | 功能 |
| --- | --- | --- |
| `make subscriber-data-show CONFIG_DIR=...` | 否 | 對照 config-specific expected documents 與 MongoDB scoped records，列出 matching、missing、different，以及 apply／clear 的影響數量。 |
| `make subscriber-data-apply CONFIG_DIR=...` | DB 寫入 | Idempotent upsert config 宣告的 SUPIs 與 Internal Group，不影響其他 subscribers。 |
| `make subscriber-data-clear CONFIG_DIR=...` | Scoped 刪除 | 只刪除 config 宣告的 SUPIs 與 group，不 drop database 或 collections。 |
| `make reset-show CONFIG_DIR=...` | 否 | 顯示 reset 會清除的五個 ML state mounts、ADRF data/model records、model files 與 NRF ADRF registration，也明列會保留的 VM、containers、images、volume objects、dataset 與 subscriber data。 |
| `make reset CONFIG_DIR=... RESET_CONFIRM=<scenario>` | Scoped 刪除 | 在相關 processes 全停後清除上述 experiment state，完成後自動 verify；不刪 VM、containers、volumes、images、dataset 或 subscriber data。 |

公開介面不保留 `subscriber-data-validate`：目前該 action 只檢查 fixture schema，沒有比對
MongoDB；fixture/config contract 應由 `config-validate` 負責。`subscriber-data-show` 則需從目前
只有 collection counts 的輸出擴充為 expected-vs-actual diff。

`RESET_CONFIRM` 使用 config manifest 的 scenario name；`reset-show` 必須顯示 exact value 與
可直接複製的完整 reset command，避免使用者自行猜測。舊 `plan/apply/verify` 字樣不成為公開
命名：`reset-show` 是唯讀影響清單，`reset` 內建 post-delete verification。

#### Phase 8.5 — Repository tests 與 command 精簡

現有 contract/network/dataset/Compose/container smoke 的覆蓋仍有價值，但它們驗證的是
repository implementation，不是使用者實驗，不能與 user-facing validation 全部叫 `check`。
公開 developer surface 只保留：

| 指令 | 功能 |
| --- | --- |
| `make test` | 聚合 config negative contracts、Netplan renderer/collision/stale alias、dataset deterministic generation/tamper rejection、Compose wiring、Vagrant definition 與 Python/shell static checks；不建立 VM。 |
| `make test-containers` | 合併現有 `ml-cpu-smoke` 與 `ml-lifecycle-smoke`，使用獨立 project/config 啟動五個 disposable CPU containers，驗證 build、health、non-root UID、device、status/log/stop 與 retained-state semantics，最後只清理自身 containers/volumes。 |

`config-contract-smoke`、`network-config-smoke`、`dataset-smoke`、`ml-compose-check`、
`ml-cpu-smoke` 與 `ml-lifecycle-smoke` 不再是 help 中的獨立公開 targets；底層測試程式可保留，
供聚合入口與 failure isolation 使用。`fl-closure-smoke` 則改以 bounded example config 描述，
不是 repository test target。

Help 分層固定為：

| 指令 | 顯示內容 |
| --- | --- |
| `make help` | 一般實驗 lifecycle。 |
| `make help-advanced` | Config、dataset、分域控制、subscriber 與 reset。 |
| `make help-dev` | `test` 與 `test-containers`。 |
| `make help-all` | 所有仍受支援的公開 targets；不列底層 scripts。 |

#### Phase 8.6 — Fresh-checkout 與 release artifacts

其他 deliverables：

- 從隔離 fresh checkout 執行 submodule initialization、config creation/validation、dataset
  generation、repository tests、VirtualBox validation 與至少一個 bounded scenario；
- prerequisite、CPU/GPU 選擇、resource sizing、troubleshooting、architecture 與 WebConsole
  optional path 文件；
- `testbed.local.example.yaml` 只保留實際被程式使用的 Host settings；實作
  `expectedVmStorage` 的 filesystem gate，移除沒有 consumer 的 `bridgeInterface`；
- 不含實驗室 IP、SSH key、private path 或 production secret；公開 clone 前處理目前
  Intelligent-Systems-Lab submodules 的 SSH URLs、repository visibility 與 fetchability；
- 修正 README 過時的 FL gate、dataset rows、GPU-only 與 provider 描述；
- CI、issue template、contribution policy、versioned release manifest；
- parent、NWDAF、ADRF、PyAnLF、PyMTLF license 與第三方 compatibility 是 public release hard
  blocker；certificate／TLS／OAuth 仍為第一版 non-goal；
- 固定 Guest Go archive checksum 與可重現 MongoDB package policy，降低 fresh provisioning 的
  supply-chain 漂移。

Exit gate：非原開發機使用者能以 VirtualBox、CPU 或 GPU 任一明確選擇，從 fresh checkout
建立 reference topology，依同一套 `experiment-*` lifecycle 完成 bounded scenario 並安全
teardown；WebConsole optional smoke 有清楚成功／失敗邊界；release artifact、component commits、
licenses、effective config 與 dataset identity 全部可追溯。

### Phase 9 — Legacy Retirement

Deliverables：

- 新舊功能與設定對照；
- 舊 untracked／excluded assets 的保留、歸檔、替代或丟棄決策；
- Phase 0.5 backup 的保留期限與最終歸檔／丟棄決策；
- 剩餘舊 repository／runtime asset 的 exact target 清單與可恢復備份；
- 使用者確認後才執行的 cleanup runbook。

Exit gate：新環境 fresh-clone E2E 已通過，舊 site-specific 資訊已有獨立可恢復副本，
且使用者另行明確授權剩餘 exact destructive targets。Phase 0.5 只提早移除 local VM，
不授權在此之前刪除舊 source repository。

## 12. 驗證矩陣

| 層級 | 主要問題 | 最低驗證 |
| --- | --- | --- |
| Source | 到底跑哪個版本？ | clean gitlinks、branch hint、dirty flag、license inventory |
| Config | component config 與 testbed definition 是否一致？ | schema、network/placement preflight、effective config identity |
| Host | 此機器能否安全承載？ | RAM、swap、VM／Docker disk、provider、driver、port、interface preflight |
| VM | 三個 role 能否重建？ | one-project Vagrant identity、idempotent guest setup、network reachability、restart |
| Container | 五個 ML service 是否可重建且彼此獨立？ | two image digests、five health checks、config/source identity、restart isolation |
| GPU | A/B 是否真的使用 GPU 且不超出單卡能力？ | container CUDA probe、effective device、一次 bounded concurrent peak VRAM、no silent CPU fallback；失敗才做 single-client diagnosis |
| Hybrid network | VM 與 Host ML endpoint 是否穩定互通？ | stable Host address、published ports、bidirectional route、firewall、no container-IP dependency |
| PseudoDriver | Dataset replay 是否在 Path RAM 內安全完成？ | file hash/bytes/rows、subscription fan-out、pre-replay headroom、UPF/VM peak、no OOM/kill |
| Core | 真實 5GC control path 是否成立？ | NRF registration、auth、registration、policy、PDU Session |
| Path | TAI 是否選到正確 UPF？ | A/B UE pool、PFCP、N3/N6、serving-SMF evidence |
| Observation | 進行中能否定位目前狀態與錯誤？ | compact health、clock skew、prefixed/filterable journald＋Docker live log |
| Analytics | A/B/C ownership 是否正確？ | discovery、subscription、WAPE、monitor owner selection |
| FL | closed loop 是否完整？ | ADRF retrieval、two rounds、FedAvg、validation、publication、cutover |
| Reproducibility | 他人能否重做？ | fresh clone、public testbed definition、artifact/source/config identity |

任何高層 E2E success 都不能取代下層 identity checks；避免 stale binary、殘留 MongoDB
state 或舊 VM process 偶然使 scenario 通過。

## 13. Repository 變更範圍

依本計畫實作時預期涉及：

| Repository | 預計變更 |
| --- | --- |
| `testbed-docs` | 本計畫、migration/network inventory、未來操作與驗證報告 |
| `5G_NWDAF_Infrastructure` | 更新 PyAnLF／PyMTLF gitlinks；新增 container images、Compose、Host ML endpoints/lifecycle；移除通過驗證後才確定過時的 guest ML setup |
| `nwdaf-resources` | 不作 submodule/runtime dependency；只有後續確認要去除已移植重複工具時才另行修改 |
| `PyAnLF` | 已完成 configurable CUDA inference；commit `9e64417` 已推送 feature branch |
| `PyMTLF` | 已完成 configurable CUDA training；commit `e9c5b08` 已推送 feature branch |
| 其他 component repo | 只在確實發現 component-owned 缺口時另開 branch 修改 |
| 舊 `5G_Infrastructure` | 預設唯讀；只作 inventory／migration source，不直接清理或重構 |

不得把跨 repository 的變更混成單一 commit。每個 component change 先在自己的 branch
驗證，再更新 integration repo 的 gitlink。

## 14. 已確定與尚待決策

已確定：

- repository 名稱使用 `5G_NWDAF_Infrastructure`，目前只保留本機 `main`，暫不設定 remote；
- 舊 local VM 已依使用者授權永久移除，舊 source repository 保留；
- 第一版採三台 network VM 加五個 Host ML containers，不採全 VM 或第四台 GPU VM；
- PyAnLF-A/B 與 PyMTLF-C 維持 CPU；PyMTLF-A/B 目前 validated reference 使用單張 RTX 3080，
  Phase 8 將 CPU/GPU 改為完整 config 的明確使用者選擇，兩者都不得 silent fallback；
- GPU activation 採既有 rootful Docker 加 NVIDIA runtime CDI mode；不另建 rootless daemon、
  不改變 `default-runtime`，也不以 restart 共用 daemon 作為正常流程；
- consumer 第一版仍為 Core VM 的單一 process，未來可另案 containerize；
- v0.1 固定三 VM／雙 TAI reference topology，`466/92` 與既有網路、subscriber、resource values
  是可修改的 committed 建議設定；
- VirtualBox 是 v0.1 唯一正式 provider；
- WebConsole 保留為 optional component 並納入獨立 bounded smoke，不測 billing、TLS 或 cert；
- `nwdaf-resources` 不成為 submodule/runtime dependency；certificate、TLS、OAuth 暫不支援。

Phase 7 runtime gates 已完成；Phase 8 尚需確認或完成：

1. CPU production mode 的 config/Compose/runtime contract 與 bounded full-stack E2E；
2. WebConsole Core build、optional lifecycle、frontend/login/MongoDB read smoke；
3. config-specific subscriber/group fixtures，讓非 `466/92` reference config 可生成、驗證與套用；
4. public submodule URLs、repository visibility、parent remote 與 fresh recursive checkout；
5. parent／NWDAF／ADRF／PyAnLF／PyMTLF license 與第三方 compatibility；
6. Host prerequisite installer boundary、Go checksum、MongoDB version reproducibility；
7. public default 的 outbound NAT 與 scenario data-network isolation policy；
8. `expectedVmStorage` gate、未使用 local fields 移除，以及實際 disk growth 的 release sizing；
9. CI、contribution、issue templates、release manifest 與 fresh-checkout acceptance；
10. compatibility targets 的 deprecation window，以及是否有外部 automation 仍使用舊名稱。

這些決策不阻擋先建立 non-privileged container definition 和 static checks，但凡涉及 Host
package/driver/network mutation、建立 VM 或長時間 GPU run，必須在對應步驟明確確認。

## 15. 下一個執行範圍

舊 VM inventory／移除、本機 repository baseline、component-owned GPU 支援，以及 parent
gitlink／component lock 更新已完成。
下一個工作包按可回復且可階段 commit 的順序執行：

1. 已完成 Host 資源／Docker data-root／port 盤點，凍結 `192.168.57.1` public candidate，並
   更新 `testbed.yaml`、local example、native config、renderer/checker 與 preflight；VM route
   需等 provider 建立 host-only network 後驗證；
2. 已完成共用 ML base/layers、兩種 image target、five-service `compose.yaml`、static checks 與
   bounded CPU-only config/health smoke；smoke 只清除自身 containers／volumes 並保留 images；
3. 已完成 `ml-start`／`ml-status`／`ml-stop`、project-scoped rollback/retention，以及
   observe/log integration；bounded CPU lifecycle smoke 已驗證 start/status/log/stop；
4. 已完成 Host toolkit、CDI spec、NVIDIA runtime registration、disposable `nvidia-smi` probe，
   以及 production PyMTLF-A/B CUDA activation；daemon reload 沒有改變 PID 或中斷八個既有
   containers；
5. 已移除 guest PyAnLF／PyMTLF provisioning、systemd 與舊 endpoint assumptions；五個 Host
   ML containers 的 idle/sync RSS 已量測，training 與 PseudoDriver peak 仍待後續 data-path
   gate 更新 Host／Path resource budget；
6. 已建立並 provision 三台 VM，完成 Core／Path guest lifecycle、six-UE registration、
   PDU Session、Host ML production activation、雙向 sync，以及 subscription／Nupf Event
   Exposure／PseudoDriver／PyAnLF／consumer analytics callback 短閉環；`fl-closure-smoke` 與
   `full-core-cat-transition` 都已完成 ADRF retrieval、concurrent GPU training、two-round
   FedAvg、publication、A／B reprovision 與 monitor cutover。Phase 7 的下一步是把既有操作與
   identity evidence 收斂成可由非原開發機執行的 Phase 8 quick start 與 release checks。
7. Phase 8 先實作分層 Make interface：一般使用者只接觸 `experiment-*`、`vm-*` 與 `logs`；
   config/dataset、分域 lifecycle、subscriber/reset 放在 advanced help；repository regression
   收斂為 `test` 與 `test-containers`。先以 aliases 保持相容，再依使用紀錄移除重複 targets。
8. 接著完成 config-owned CPU/GPU policy、config-specific subscriber fixtures 與 optional
   WebConsole lifecycle；每項先做 isolated/CPU smoke，再分別通過 bounded full-stack gate。
9. 最後才做 fresh-checkout acceptance、公開 URLs/licenses、release metadata 與文件收斂；在
   license、component fetchability 與 fresh-checkout E2E 通過前，不宣稱 repository 已可公開發布。

每一步先通過該層驗證再 commit；provider 安裝／修復、Host toolkit/network mutation、新 VM
建立與 privileged E2E 仍需在對應步驟明確執行。新 repository 不以加入
`nwdaf-resources` submodule 作為前置。
