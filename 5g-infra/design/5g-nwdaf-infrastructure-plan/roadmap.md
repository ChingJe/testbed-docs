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
- 定義 `testbed.yaml` 完整 topology schema、Host ML published endpoints／device placement 與
  explicit `TESTBED=<path>` selection，並診斷它們和 component config 的 network／placement 一致；
- 實作 `config-render.py`，只寫入 gitignored `config/local/<name>/`；實作同時
  診斷 default／local set 的 `config-check.py`；
- 盤點 `nwdaf-resources` 中與公開 VM 環境直接相關的 preflight、subscriber/group
  provisioning 與 config preparation；只移植或重寫必要小工具；
- 以既有 `nwdaf_uecomm_consumer` 為來源，整理 infrastructure-owned consumer CLI、
  callback reachability、transient subscription state 與 provenance；
- 移植時記錄原 repository、source commit 與 license，並明確指定新 owner；不複製
  整個 repository，也不讓新環境依賴它的 workspace layout；
- `nwdaf-resources` 保留既有 isolated／single-host regression，是否清理重複工具屬於
  後續獨立變更。

Exit gate：新 repository 不 clone `nwdaf-resources`、不執行 renderer，也能以 default 完成
non-mutating diagnostics 與準備三 VM；renderer 輸出和手工 local set 都能由相同
checker 列出 consistency findings，任何移植工具的 provenance 與 ownership 清楚。

2026-08-09 implementation record：`testbed.yaml` 已將 Core／Path A／Path B 固定為
4096／3072／3072 MiB、三顆 40 GiB dynamic logical disk，並將五個 Python backend placement
移到 Host containers。Default native config 與 renderer 已分離 container bind address
（`0.0.0.0`）和 VM-visible advertised endpoints；checker 會驗證 placement、port、device、
callback/artifact URL、PseudoDriver profile／guest active path 與 dataset contract。Preflight 以
10 GiB VM allocation 加 6 GiB Host reserve 作為建議資源線，RAM／swap 不足都由 diagnostics 顯示風險；Compose
lifecycle 仍屬 Phase 6.5，這筆紀錄不宣稱 container 或 VM 已啟動。

### Phase 4 — 三 VM Skeleton 與 Network

Deliverables：

- provider 選型與 prerequisite check；
- root multi-machine `Vagrantfile` 內的 `core`、`path-a`、`path-b` VM；
- management、SBI、N2、N3-A/B、N4、N6-A/B plane；
- Host SBI address 到三台 VM 的雙向 reachability，以及五個 ML published port 的 conflict／
  firewall smoke；
- explicit complete `TESTBED=<path>` selection 與 public isolated default；
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
| `make experiment-validate CONFIG_DIR=...` | 否 | 診斷 VirtualBox、Docker、submodules／locks、RAM、磁碟、ports、CPU/GPU prerequisites、config contract、Compose wiring 與既有 dataset；不建立 VM、不啟動服務，也不作為後續 start 的必須授權。 |
| `make vm-up` | 是 | 建立或啟動三台 VM；首次建立會 provision/build，但不啟動實驗 processes。 |
| `make vm-status` | 否 | 顯示三台 VM 為 running、poweroff 或不存在。 |
| `make vm-halt` | 是 | Graceful shutdown 三台 VM，不刪 VM 或 virtual disks。 |
| `make experiment-start CONFIG_DIR=...` | 是 | 產生可建構的 dataset、載入完整 artifacts，啟動 Guest services、config 指定的 CPU/GPU ML containers、optional WebConsole、consumer 與兩筆 subscriptions；不先執行完整 validation gate，實際 runtime 操作失敗時才反向停止本次已啟動項目。 |
| `make experiment-status` | 否 | 集中顯示 VM、23 個 Guest services、五個 ML containers、effective device、WebConsole、consumer、subscriptions、config identity 與基本 Host resources。 |
| `make experiment-stop` | 是 | 精確刪除兩筆 subscriptions，停止 consumer、optional WebConsole、ML 與 Guest services；保留 VM、dataset、MongoDB、ADRF/model state、containers、images 與 volumes。 |
| `make logs` | 否 | 依來源、VM、service、時間與 follow mode 查看 Guest、ML、consumer 與 WebConsole logs。 |

建議流程可先執行唯讀診斷：

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
與 subscriptions 仍可分開控制的 ownership。它不能自動開機、reset state 或關閉 VM；
使用者即使不先執行 `experiment-validate`，或已知診斷有 findings，仍可直接要求 start。

#### Phase 8.2 — Config 與 dataset 工具

由 `make help-advanced` 顯示：

| 指令 | 是否改變狀態 | 功能 |
| --- | --- | --- |
| `make config-create NAME=<name> FROM=<scenario-yaml-path>` | 本機檔案 | 以必填的 repository-relative scenario YAML 路徑建立完整 `config/local/<name>/`；不接受 short name，也沒有預設 example。 |
| `make config-validate CONFIG_DIR=...` | 否 | 對照指定 config、`testbed.yaml`、manifest-selected scenario、subscriber fixtures、PyMTLF seed metadata 與 NF／ML／UERANSIM configs，診斷 PLMN、TAI、S-NSSAI、DNN、endpoints、callbacks、network aliases、FL policy、timeouts 與 source hashes；findings 不成為 start gate。 |
| `make dataset-generate CONFIG_DIR=...` | 本機檔案 | 依 UE pools、traffic profiles、sampling 與 model shape 產生 Path A/B Parquet artifacts；只有無法解析或生成 artifact 的 structural error 才失敗，實驗容量風險只列為診斷。 |
| `make dataset-validate CONFIG_DIR=...` | 否 | 診斷實際 Parquet 的 dataset ID、schema、SHA-256、bytes、rows、UE IP、timestamps、warm-start boundary、training／validation capacity 與 bounded trigger／closure。Dataset 不存在時回報失敗，但不授權或禁止後續 start。 |
| `make dataset-show CONFIG_DIR=...` | 否 | 以人類可讀格式顯示 dataset identity、Path A/B hash／rows／UEs，以及 raw interval、warm-start boundary、traffic transition、sampling、monitor、trigger 與 closure 時間。 |
| `make dataset-load CONFIG_DIR=...` | Guest state | 將已驗證的 Path A/B artifacts 上傳到所屬 VM，Guest 再驗證 identity/hash 後原子切換 active dataset。 |

`experiment-start` 只執行實際所需的 generate 與 load，不把 `config-validate`、
`dataset-validate` 或 `experiment-validate` 作為前置 gate；上述唯讀 targets 保留給 config 作者、
CI 與 data-path 除錯。`466/92` 要真正成為可替換的建議值，subscriber/group fixtures 不能繼續是
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
- preflight 直接回報 provider 實際 VM storage filesystem，不依賴 local overlay 或未被使用的
  `expectedVmStorage`／`bridgeInterface` 欄位；
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

#### Phase 8.7 — Experiment authoring 與 non-gating diagnostics

本工作包不修改 NF／ML／RAN submodule，只調整 Infrastructure-owned definitions、
renderer／resolver／checker、Host lifecycle scripts、tests 與文件。

Deliverables：

1. 將使用者可執行的 scenario 與 traffic profiles 從 `fixtures/full-core/` 移到
   `experiments/examples/<name>/`，新增 gitignored `experiments/local/`；只有程式測試輸入才使用
   `tests/fixtures/`。
2. Scenario schema v2 以 scenario directory 作為 `trafficProfiles` 的相對路徑基準；
   config renderer、checker 與 dataset resolver 共用一個 repository-bounded resolver。
3. `make config-create` 要求 `FROM=<scenario-yaml-path>`；不提供 example short name、預設
   selection、舊 config migration 或 `fixtures/full-core` compatibility layer。
4. 將 dataset structural resolution 與 advisory capacity diagnostics 分開；可產生但不符合
   reference timing／sample／A-B policy 的 definition 仍可 generate 並進入 runtime。
5. `experiment-start`、`services-start`、`ml-start`、`webconsole-start` 和 start path 中的
   subscriber apply 移除完整 validation gates。必要檔案、artifact integrity、runtime capability、
   lifecycle collision、process start 與 destructive scope 仍是 hard execution boundary。
6. `dataset-show` 改為人類可讀的 row／observation／report／sample 與時間線摘要；
   manifest 仍作為 machine-readable artifact。
7. Infrastructure repository 新增 testbed、scenario、traffic profile、native config 與 dataset
   field references，並同步 README、commands、operations 與 troubleshooting，不留過期路徑或
   start-gating 說明。

Validation：

- 兩個 committed examples 和一個 repository-local custom scenario 都能 render；
- missing／absolute／escaping profile path 與錯誤 schema／Path identity 有清楚錯誤；
- 產生後 native configs 除 provenance／identity 外與調整前 runtime 語意一致；
- 一組刻意不一致但仍可建構的 config 會讓 validation 回報 findings，卻不被
  dataset generation 或 start orchestration 預先拒絕；
- 真正缺檔、parse failure、tampered dataset、runtime start failure 與 rollback 仍受測試保護；
- `make test`、兩個 example 的 dataset generate／validate、read-only `experiment-validate`
  與文件連結檢查通過。若產生的 runtime config 有非預期差異，先停止而不啟動 VM E2E。

Exit gate：使用者可只靠 Infrastructure 文件找到所有可編輯輸入、以明確 path 複製／
選擇自訂 experiment，並能在了解 validation findings 後自行決定是否啟動；
start 不再把建議性診斷當成授權。

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

下一個工作包是 Phase 8.7，按以下可階段驗證與 commit 的順序執行：

1. `refactor(config): organize experiment definitions`
   - 搬移兩個 examples，建立 `experiments/local/`、scenario schema v2 與共用 profile resolver；
   - 更新 default manifest、renderer、checker、dataset resolver 與相關 tests；
   - 不更改舊 local config，不保留舊 definition compatibility files。
2. `feat(config): require explicit scenario paths`
   - `FROM` 改為必填 YAML path，移除 short name 與預設 example；
   - 測試 example、custom、missing／escaping path 與 existing output refusal。
3. `refactor(validation): separate diagnostics from execution`
   - 將 dataset structural requirements 與 advisory findings 分開；
   - 移除 aggregate／Guest／ML／WebConsole start 前的完整 validation gate；
   - 保留 artifact integrity、runtime capability、lifecycle collision、start failure 與 rollback。
4. `feat(dataset): present resolved timeline summaries`
   - `dataset-show` 顯示 row／observation／report／sample 關係與兩條 Path 時間線；
   - 用 5 秒與 30 秒 sampling cases 驗證 raw row aggregation 推導。
5. `docs(config): document experiment configuration contracts`
   - 完成五份 field reference 與「想改什麼應該改哪裡」入口；
   - 同步 runtime README、commands、operations 與 troubleshooting，清除舊路徑與 gating 敘述。

每階段先執行專門測試再 commit，最後執行 `make test`、兩個 examples 的
dataset generate／validate 和 read-only `experiment-validate`。本工作包不修改 submodule、
不刪除現有 local config／dataset／runtime state，不建立 VM 也不啟動 full E2E；若 native
runtime config 出現非預期差異，先停止並回報。
