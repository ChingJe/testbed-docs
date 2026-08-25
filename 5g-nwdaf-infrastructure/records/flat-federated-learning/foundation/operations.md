# Host／Guest Scripts 與 Build Lifecycle

[返回基礎建置計畫](plan.md)

## 10. Host／Guest Scripts 與 Build Lifecycle

### 10.1 Host scripts

`scripts/host/` 在實體機執行，協調 source identity、Vagrant、三台 VM、Docker ML services
以及 guest service／subscription lifecycle；不在 Host 編譯 guest component，第一版也不把
consumer 搬出 Core VM：

- `preflight.sh`：唯讀診斷 provider、RAM、swap、VM／Docker storage free space、submodule、
  testbed files、active config set、必要網段與 port；ML slice 另診斷 Docker、GPU driver、
  NVIDIA Container Toolkit 與容器內 CUDA probe；它可呼叫 config／dataset checker 產生完整
  findings，但不被 start path 當成授權 gate，也不安裝 toolkit、修改 host network 或建立 VM；
- `dataset.py`／`dataset-stage.sh`：由 topology/profile 生成或重掃 A/B Parquet，plan 分發
  mapping，apply 只上傳 role-specific artifact 並要求 Guest 原子啟用；
- `config-render.py`：由 default baseline、explicit topology definition 與必填 scenario YAML path 產生
  `config/local/<name>/` native configs 和 manifest，不覆寫 committed/default files；
- `config-check.py`：唯讀診斷 default／local config set 的跨 component network、
  identity、placement、A/B mapping 與所選 testbed definition 一致；
- `services-start.sh`：解析 effective `TESTBED`／`CONFIG_DIR`，stage、activate 完整
  config set 與 Path-specific dataset；各 Guest 先驗證 active config 所產生的 persistent
  Netplan aliases 已由 networkd 套用，只有 drift 時才 reconcile，再依 dependency order 透過
  Vagrant SSH／guest service interface 啟動 Core、Path A 與 Path B 的 database、NF、RAN 與
  Go NWDAF process；不隱含啟動 ML container；
- `services-stop.sh`：停止 experiment stack service/process，不等於 `vagrant halt`，
  更不等於 destructive `vagrant destroy`；
- `services-status.sh`：只彙整 core NF、UPF、gNB／UE、Go NWDAF 與 database health，並回報
  Path `MemAvailable`、UPF memory accounting 與 PseudoDriver replay state；
  VM power state 另外由 `vagrant status` 或 `make vm-status` 回報；
- `ml-start.sh`：解析所選 config 並以 Compose 建置／啟動 `pyanlf-a`、`pymtlf-a`、
  `pyanlf-b`、`pymtlf-b`、`pymtlf-c`；Host bind、GPU runtime、Compose build／health 等實際
  無法繼續的 runtime failure 仍 fail 並 rollback，但不先要求完整 config／resource diagnostics 通過；
- `ml-status.sh`：彙整 container state、application health、effective device、GPU visibility、
  image/source identity 與 endpoint reachability；
- `ml-stop.sh`：停止五個 ML containers，但不刪 image、volume、VM、guest process 或
  subscription state；
- `subscriptions-start.sh`：透過 Vagrant SSH 在 Core 啟動單一 consumer callback，確認
  A/B callback reachability 與兩台 Path 都具有 dataset manifest 要求的 replay headroom，
  再要求 consumer 經 NRF discovery 找到兩個不同的 NWDAF 並建立兩筆 subscription；部分
  成功時負責 rollback；
- `subscriptions-status.sh`：彙整 Core consumer／callback 狀態、discovered A/B NF
  identities、兩筆 subscription identities 與最近 notification；
- `subscriptions-stop.sh`：要求 Core consumer 依保存的 exact `Location` 退訂兩側，成功後
  停止 callback；失敗時保留可重試的 state，不等於停止 5GC services；
- `observe.sh`：週期性顯示三台 VM、guest service、ML container、GPU、discovered A/B、
  subscription 與最近 callback 的 compact terminal view，不解析完整 business log；
- `logs.sh`：並行 follow guest journald 與 `docker compose logs`，為每行加上 VM／unit 或
  Host／container prefix，支援依來源、service 與 since time 過濾；結束 follow 不停止任何
  process。

Renderer／checker 在 Host 執行，因為它們在 VM 建立前產生或診斷 effective config
set；guest 只接收使用者明確選擇的完整 config set，不在 VM 內自行組合或回寫設定。
未通過 checker 不等於使用者不得啟動；實際 stage、parse、integrity 或 process start 失敗才由
execution path 停止。

### 10.2 Guest scripts

`scripts/guest/` 由 Vagrant 上傳／執行：

- `common.sh`：三台 VM 共用的 OS package、Go、`uv`、runtime user、directory 與基本
  observability；
- `config-activate.sh`：驗證 staged config identity 與 guest role，在 services／consumer
  都停止時原子切換 `/etc/5g-nwdaf-infrastructure/active` symlink，並同步原子更新該組 config
  產生的 persistent Netplan fragment；失敗時恢復上一組 active config 與 fragment，不修改
  committed/shared source；
- `network-setup.sh`：只讀 active config set 的 `network/<role>.yaml`，以 Vagrant 建立的
  base address 作為 anchor 找到實際介面，產生由 infrastructure 管理的
  `/etc/netplan/60-5g-nwdaf-aliases.yaml`。候選檔先在隔離 root 通過 `netplan generate`，再原子
  安裝，要求 networkd reload／只 reconfigure 受影響介面並驗證 effective addresses；切換時
  移除上一組不再宣告的 aliases，`--clear` 則移除整份 fragment。它不內建固定 IP 清單，也不再
  用 `ip address add` 建立只存在於 runtime 的位址；
- `core.sh`：Core 的 MongoDB、core NF、ADRF、NWDAF-C 與 optional
  webconsole setup／build；
- `path.sh A|B`：Path A/B 的 kernel headers、network、gtp5g、UPF、UERANSIM 與 NWDAF
  setup／build。

概念流程：

```text
core   -> common.sh -> core.sh
path-a -> common.sh -> path.sh A
path-b -> common.sh -> path.sh B
```

Provisioning 與 deployment 仍是不同 lifecycle 語意，但不各占一個頂層目錄；scripts
內部以 idempotent setup／build／start／stop action 區分。Guest 不自行 clone branch、
不編輯 submodule source，也不把 generated data copy 回 source tree。

#### Guest network ownership

Vagrant、active config 與 Guest OS 的責任必須分開：

- Vagrantfile／`testbed.yaml` 管理 VM adapters、host-only segments 與每張介面的 base／anchor
  address，並由 Vagrant 產生 `/etc/netplan/50-vagrant.yaml`；
- active config 的 `network/<role>.yaml` 管理該 role 應具有的 NF／database／consumer aliases；
- infrastructure renderer 管理單一 `60-5g-nwdaf-aliases.yaml` fragment；
- Netplan／systemd-networkd 管理 anchor 加 aliases 的 effective persistent addresses。

後置 fragment 只宣告 aliases，不複製 Vagrant 的 anchors。Netplan 將兩份 address sequence
合併，因此 Vagrant 每次完成自己的 `50-vagrant.yaml` 並執行 `netplan apply` 時，也會自然把
aliases 一起套用。不得把全部 NF aliases 寫進 Vagrantfile，也不得再依賴 Vagrant post-up
hook、Make wrapper 或 boot-time `ip address add` 修補時序。

`5g-nwdaf-network.service` 保留為 config activation、manual repair 與 NF dependency 可呼叫的
oneshot reconciler，但 boot correctness 不再依賴它早於或晚於 Vagrant 執行。既有 VM 遷移後
應取消 unconditional boot-time imperative alias mutation；`services-start` 先做 effective
address verification，只有 fragment、networkd output 或實際 address drift 時才呼叫
reconcile。候選套用不得廣泛重設 NAT／management SSH 介面，優先使用 `netplan generate`、
`networkctl reload` 與受影響 host-only interfaces 的 targeted reconfigure。

### 10.3 Guest 與 Host build responsibility

Host 只將 parent 已固定的 clean source snapshot 與 config 交給對應 VM。三台 VM 分工：

| VM | Guest-local 建置／環境 |
| --- | --- |
| `core` | NRF、AMF、SMF、UDR、UDM、AUSF、NSSF、PCF、NWDAF-C、ADRF；optional webconsole |
| `path-a` | gtp5g、UPF-A、UERANSIM、NWDAF-A |
| `path-b` | gtp5g、UPF-B、UERANSIM、NWDAF-B |

UERANSIM 在 `path-a`／`path-b` 各編譯一次，產生各 VM 自己的 `nr-gnb`／`nr-ue`。
這避免 host C++ toolchain、額外 builder 與 host/guest library compatibility 門檻；A/B
差異仍來自 config，不是不同 source revision。每台記錄 source commit、guest OS、
compiler/CMake 與 binary hash，可有 keyed local cache，但 cache miss 必須可 clean build。

`gtp5g` 必須配合各 path guest kernel 建置。先測 pinned upstream 原版；只有實際 guest
kernel 需要時，才把 reviewed patch 放在 `kernel/` 內靠近 gtp5g，不預先建立泛用
`patches/`。建置後可清除 apt、Go 與 compiler cache，以控制動態磁碟占用。

Host 以 parent gitlink 固定的 `ML/PyAnLF`／`ML/PyMTLF` build 兩種 image。兩者應共用
Python 3.12、PyTorch 2.5.1＋CUDA 12.1 的 immutable base/layers，避免把相同 CUDA wheel
與 NVIDIA runtime dependency 各存一份；application layer 仍各自由自己的 lockfile 和
`uv sync --frozen` 建立。Build 必須避免永久保留不必要的 `uv` download cache，並記錄
source commit、lock hash、base image digest 與 final image digest。

PyAnLF／PyMTLF image 可以具備 CUDA runtime，但 device 是否交給 container 由 Compose
service 設定。這讓 PyAnLF 預設 CPU、PyMTLF-A/B 使用 GPU、PyMTLF-C 使用 CPU，而不需要
為 CPU/GPU placement 維護不同 source branch 或修改 lockfile。Production Compose 的 GPU
宣告使用 NVIDIA runtime CDI mode；CPU smoke override 必須改回 `runc` 並將
`NVIDIA_VISIBLE_DEVICES` 設為 `void`，不能殘留 legacy `gpus:`、Host device mapping 或 GPU
injection。

### 10.4 Operations

VM power、5G/NWDAF guest services、ML containers 與 NWDAF subscriptions 是四組獨立操作：

```text
make preflight

make config-check
make config-render NAME=<name> TESTBED=<definition>

make vm-up
make vm-status
make vm-halt

make services-start
make services-status
make services-stop

make ml-start
make ml-status
make ml-stop

make subscriptions-start
make subscriptions-status
make subscriptions-stop

make observe
make logs
```

`make vm-up` 對應 multi-machine `vagrant up`：第一次建立 VM 時執行 guest setup/build，
之後只負責讓既有 VM 開機；它不啟動 experiment stack。`make services-start` 只管理
MongoDB、core NF、UPF、Go NWDAF 與 UERANSIM；`make ml-start` 只管理 Host ML containers。
啟動 full stack 時由高層 orchestration 按 dependency stage 呼叫兩者，不把 Docker
lifecycle 藏進 Vagrant 或 guest systemd。

Guest setup 不應將這些 experiment services 設為隨 VM boot 自動啟動；套件安裝若預設
啟動 MongoDB 或其他 daemon，setup 必須停止並 disable，再交由
`services-start.sh` 管理。`make services-stop` 後三台 VM 仍保持開機，可以修改設定、
重新 build 或執行下一輪 service start；`make vm-halt` 才關閉 VM。

實際實作可由 Makefile 呼叫 host scripts／Vagrant／provider tools，但使用者不應需要
記住 guest 內部 source path 或逐一 SSH 啟動 process。

`Makefile` 是 user-facing command surface，不要求每個 target 都有一支同名 script。
第一版預計對應如下：

| Make target | 第一版 implementation | 是否需要獨立 host script |
| --- | --- | --- |
| `experiment-validate` | 呼叫 `scripts/host/experiment-validate.sh` | 是；聚合 host／provider／config／dataset／Compose 唯讀診斷，不由 start 自動執行 |
| `config-validate` | 呼叫 `scripts/host/config-check.py` | 是；診斷 selected testbed/config set 的跨檔一致性 |
| `config-create` | 呼叫 `scripts/host/config-render.py` | 是；要求明確 `FROM=<scenario-yaml-path>` 產生 gitignored complete config set |
| `vm-up` | 直接呼叫 multi-machine `vagrant up` | 否；先避免沒有邏輯的一行 wrapper |
| `vm-status` | 直接呼叫 `vagrant status` | 否 |
| `vm-halt` | 直接呼叫 `vagrant halt` | 否 |
| `services-start` | 呼叫 `scripts/host/services-start.sh` | 是；包含 config／dataset activation、跨 VM stage、readiness 與 rollback，不包含完整 validation gate |
| `services-status` | 呼叫 `scripts/host/services-status.sh` | 是；包含 process 與 application health |
| `services-stop` | 呼叫 `scripts/host/services-stop.sh` | 是；包含反向 dependency order |
| `ml-start` | 呼叫 `scripts/host/ml-start.sh` | 是；包含 Compose build/up、GPU gate、health 與 rollback |
| `ml-status` | 呼叫 `scripts/host/ml-status.sh` | 是；包含 container、device、image 與 application health |
| `ml-stop` | 呼叫 `scripts/host/ml-stop.sh` | 是；只停止 ML containers，不刪 image／volume |
| `subscriptions-start` | 呼叫 `scripts/host/subscriptions-start.sh` | 是；包含 callback、NRF discovery、雙訂閱與 rollback |
| `subscriptions-status` | 呼叫 `scripts/host/subscriptions-status.sh` | 是；包含 discovery／subscription state 與 notification status |
| `subscriptions-stop` | 呼叫 `scripts/host/subscriptions-stop.sh` | 是；包含精確退訂與 failure-state preservation |
| `observe` | 呼叫 `scripts/host/observe.sh` | 是；週期性組合 VM／guest service／ML／subscription 摘要 |
| `logs` | 呼叫 `scripts/host/logs.sh` | 是；跨 VM journald 與 Host Compose logs 的 follow、prefix 與 filter |

如果後續 `vm-up` 需要 provider 選擇、storage preparation 或額外錯誤處理，再抽成
`scripts/host/vm-up.sh`；第一版不為了名稱對稱先建立空殼 wrapper。Guest scripts 則只處理
VM 內 provisioning／build／systemd unit 安裝，不作為 Host 的使用者入口。

#### Effective config activation

`services-start` 不讓 process 直接讀 Host `config/`，而是執行以下流程：

1. 依 `CONFIG_DIR` explicit argument 或 selected `TESTBED:config.directory` 解析完整 config set，
   並解析同一次命令使用的 `TESTBED` definition；
2. 計算 config manifest/hash 並確認 stage 所需的必要檔案可讀。跨檔一致性由獨立
   `config-validate` 診斷，不作為 start gate；Vagrant base interface 或 active network 實際無法
   套用時，execution path 仍停止並 rollback，不在 service start 時重建 VM 網卡；
3. requested config identity 與 active config 相同時可 idempotent reuse；兩者不同時，必須
   確認三台 VM 上的 experiment services、五個 ML containers 與 Core
   consumer/subscriptions 都已停止，不得 hot switch；
4. 透過 Vagrant upload／SSH 將各 guest role 需要的 files 與完整 manifest stage 到
   `/etc/5g-nwdaf-infrastructure/config-sets/<config-name>-<short-hash>/`；
5. 三台 Guest 先各自驗證 staged files；全部 prepare 成功後，Host 才要求
   `config-activate.sh` 原子更新 `/etc/5g-nwdaf-infrastructure/active` symlink。任一 guest
   activation 失敗時，Host 將已切換的 guest 回復到先前 link，且不啟動 process，避免
   half-activated set；
6. Guest 切換 active symlink 後，`5g-nwdaf-network.service` 由 `network/<role>.yaml` 產生
   persistent Netplan fragment，先在隔離 root 驗證，再原子安裝並要求 networkd targeted
   reconfigure。任一 anchor、role、address、Netplan generation 或 effective address 驗證失敗
   時，同時恢復上一組 active config 與 fragment，且不啟動 process；
7. systemd units 使用固定 path，例如
   `ExecStart=... --config /etc/5g-nwdaf-infrastructure/active/nrfcfg.yaml`，再由既定
   dependency order 啟動；unit file 不因 config set 改變而重寫；
8. `ml-start` 將同一 config set 中 PyAnLF／PyMTLF 所需檔案 read-only bind mount 到各自
   container，並把 config hash、source commit 與 image digest 暴露給 `ml-status`；container
   不回寫 committed config；
9. `services-status`／`ml-status`／`observe` 回報三台 VM 與五個 containers 的 active config
   hash，任一者不同或與 Host selected config 不同時視為錯誤。

因此只改 component 參數或 process alias、且 Vagrant base network identity 不變時，可以先
`subscriptions-stop`、
`ml-stop`、`services-stop`，再以另一個 `CONFIG_DIR` 重新啟動，不必重建 VM。若變更
management/SBI/N2/N3/N4/N6 的 VM base interface 或 Vagrant network，則必須先讓 VM
套用相同 `TESTBED` definition。後續
`subscriptions-start` 只讀 Core 已 active 的 `consumer.yaml`，不接受另一組臨時 config，
避免 services 與 subscriptions 使用不同組合。

`destroy`、database clear、VM removal 與舊環境清理必須是獨立的 destructive command，
具備 exact target、dry-run／confirmation 與 recovery 說明，不能隱含在 `vm-up`、
`services-start` 或 scenario retry 中。

### 10.5 Process supervision 與跨 VM 啟動順序

舊環境主要依賴使用者逐台 SSH、直接執行 binary，或由 `run.sh` 以 background process／
PID list 管理。新環境改成：每個 guest 長時間 component 使用一個 systemd service unit；
每個 Host ML backend 使用一個 Compose service。兩者 install/build 後都不隨 VM boot 或
Docker daemon 自動啟動。

概念 service 包含：

- Core：MongoDB、NRF、NSSF、UDR、UDM、AUSF、PCF、AMF、SMF、ADRF、NWDAF-C，
  以及 optional webconsole；
- Path A/B：UPF、gNB、UE instances 與 NWDAF；
- Host：`pyanlf-a`、`pymtlf-a`、`pyanlf-b`、`pymtlf-b`、`pymtlf-c` 五個 containers；
- UE 可使用 systemd template unit，例如 `ueransim-ue@1.service`，讓 A 啟動 UE1–3、
  B 啟動 UE4–6。

Unit 至少明確指定 `User`、`WorkingDirectory`、`ExecStart`、config path、restart policy
與 journald output。Guest `core.sh`／`path.sh` 負責安裝 unit file 和 `daemon-reload`，但
不執行 `systemctl enable`，也不在 setup 結尾啟動 experiment service。

Systemd 與 Compose 都只能回答各自 execution domain 的狀態，無法保證另一個 VM 或 Host
endpoint 已 ready。因此完整 bring-up 由 Host 分 stage 呼叫 guest 與 ML lifecycle，並在
每個 dependency barrier 等待 readiness，例如：

```text
1. Core MongoDB -> wait database ready
2. Core NRF -> wait NRF SBI ready
3. NSSF/UDR/UDM/AUSF/PCF/AMF -> wait health and NRF registration
4. Path A/B UPF + Core SMF -> wait PFCP association and SMF registration
5. Core ADRF -> wait storage/model endpoint ready
6. Host five ML containers -> wait process health and verify effective CPU/CUDA devices
7. NWDAF-C then NWDAF-A/B -> wait NRF registration, role identity and ML cross-endpoint health
8. gNB-A/B -> wait NG setup
9. UE1–6 -> wait registration, PDU Session and expected UE pools
```

可安全平行的 A/B stage 可以平行執行，但每一 stage 必須有 bounded timeout、明確失敗
訊息與當下 service status。單一 lifecycle command 只停止自己管理的 domain；完整 stack
shutdown 則先停止 subscriptions，再反向停止 guest dependency 與 ML containers，避免先停
database／core 而留下仍在寫入的 UE、NWDAF 或 training job。為了診斷而只停其中一層時，
另一層可保持啟動，但其 connection retry 必須 bounded 且可觀察。

`services-status.sh` 同時檢查 systemd active state 與必要的 application-level health；
`ml-status.sh` 同時檢查 container state、HTTP health、effective device 與 GPU visibility。
只看到 process/container active 不代表 NRF registration、PFCP、PDU Session 或 NWDAF
backend 已 ready。Guest log 留在 journald，ML stdout/stderr 交由 Docker log driver。

需要單步除錯時仍可維持原本操作感：Guest 可先停止 unit，再 SSH 進 VM 手動執行 binary；
ML 可只停止單一 Compose service，再以 interactive container 或 foreground command 執行。
結束後再交回 systemd／Compose。自動化是取代重複操作，不限制人工診斷。

### 10.6 NWDAF consumer 與訂閱生命週期

既有 `5G_Infrastructure/NWDAF/nwdaf_uecomm_consumer/` 是目前實驗的實際入口：Python CLI
啟動 HTTP callback server，向 Nnwdaf Events Subscription API 建立 `UE_COMMUNICATION`
訂閱，從 `Location` 保存 subscription ID，最後再 DELETE 退訂。舊 `run.sh` 同時以
`config1.json`／`config2.json` 啟動兩個 background process；舊 `clean.sh` 則依 state
file、listener port 與 PID 清理。`5GC/provision.sh` 將整個目錄複製到 5GC guest。

新環境保留這個功能，但不照搬 background PID 和 `127.0.0.11`：

- consumer implementation 移植／整理至 `tools/nwdaf-consumer/`，由 infrastructure repo
  維護，並記錄舊來源 revision、既有 local config diff 與 license；目前舊 `config1.json`／
  `config2.json` 對 base URL 和 reporting period 的調整尚未提交，不能只從 HEAD 重建；
- 單一 consumer process 執行在 `core` VM，使用一個 A/B 都能到達的固定 SBI/private
  address 接收 callback；Host 不直接當 callback endpoint，避免綁定 host firewall、
  provider 與實驗室 LAN；
- `config/consumer.yaml` 只保存 NRF URI、requester identity、`UE_COMMUNICATION`、同一個
  Internal Group ID、TAI `000001`／`000002`、reporting policy，以及一個 callback bind／
  advertised URI；不保存 NWDAF-A/B 的固定 API address；
- consumer 先向 NRF 執行一次 analytics-provider discovery：
  `target-nf-type=NWDAF`、`service-names=nnwdaf-eventssubscription`、
  `nwdaf-event-list=UE_COMMUNICATION`，再從 `REGISTERED` profiles 的 service endpoint 與
  TAI scope 將唯一候選映射成 A/B。A/B 任一 TAI 沒有候選、同一 TAI 有多個未能唯一選擇
  的候選，或兩者解析成相同 NF instance，都必須失敗，不退回 hard-coded IP；
- `requester-nf-type` 必須代表這個 consumer 的實際角色，不能假裝成 NWDAF-A/B；第一版
  預計使用 `AF`，並在 Phase 3 先以 team NRF 做 compatibility gate；
- 同一個 callback server 接收兩個 NWDAF 的 notifications，以不同 `notifCorrId`／
  subscription ID 辨識來源；不需要兩個 consumer、兩個 listener 或兩個 port；
- callback listener 可由 disabled `nwdaf-consumer.service` 管理，但不納入
  `services-start`；它只在訂閱週期內啟動；
- subscription state 放在 core guest 的明確 transient state directory，不寫回 git
  working tree，也不視為 per-run archive。單一 state file 內保存 discovery response
  identity，以及 A/B 各自的 NF instance、service endpoint、TAI、Location、subscription
  ID、notification correlation ID 與建立時間，供精確退訂與避免重複訂閱；
- 第一版不因此重新引入 run ID、runtime archive 或 log collection framework。

Consumer 未納入目前五個 ML containers 的改造範圍。後續可把它改成第六個獨立 container，
同時承擔長時間 callback server、subscription ownership 與 notification stdout log；但必須先
確認 Host callback published address 可由 A/B NWDAF 穩定到達，且不能因此改回兩個 consumer
或把 subscription lifecycle 併入 ML lifecycle。

四組可獨立啟停的生命週期因此明確分開；traffic／degradation 等實驗動作在訂閱成立後
另外執行：

```text
time -------------------------------------------------------------->
VM:             vm-up ====================================== vm-halt
Services:              services-start =============== services-stop
ML containers:                  ml-start ============= ml-stop
Subscriptions:                            start ===== stop
Actions:                                      traffic / degradation
```

`subscriptions-start.sh` 必須先確認 `services-status`、`ml-status`、A/B NWDAF readiness 與
Core callback address 可達，再依序：

1. 啟動單一 callback listener，確認 port ready，並從兩台 Path VM 驗證 callback address
   可達；
2. 查詢 NRF，要求 discovery result 能唯一對應 TAI A 與 TAI B 的兩個不同 NWDAF；
3. 由回傳 profile 組出兩個 Events Subscription API URI，經同一個 local barrier 並行
   POST `UE_COMMUNICATION` subscription；
4. 要求兩者都回傳成功狀態與有效 `Location`，才宣布 subscription set ready；
5. 分別保存 A/B request／response timestamp；各 path 的 analytics start 是自己的
   subscription creation，兩側都完成的較晚時間則是後續共同 traffic action 的 ready
   barrier；
6. 若只有一側成功，立即 DELETE 已成功的一側並停止 listener，不留下 half-started
   experiment。

`subscriptions-status.sh` 回報單一 consumer/callback listener、discovered A/B identity、
兩筆 subscription identity 與最近一次 notification 狀態。`subscriptions-stop.sh` 依各自
保存的 exact `Location` 退訂，成功後停止 listener 並清理 transient state；若 DELETE
失敗，不應像舊腳本一樣直接丟掉唯一的 subscription identity，必須保留可重試資訊或要求
明確 force cleanup。

如此可在同一組已開機 VM、同一批已啟動 5GC/NWDAF services 上執行多次獨立訂閱週期；
建立／刪除訂閱不會隱含重建 VM、重啟 NF、清空 MongoDB 或建立 run archive。

### 10.7 最小 Live Observation

第一版不依賴 `5g-viz`，也不重新引入 session／run archive，但必須能回答兩類不同問題：

- 「現在是否正常」：由 `make observe` 顯示 VM power、systemd／application health、NRF
  registration、PFCP、UE registration／PDU Session、五個 ML container health、effective
  device／GPU usage、consumer discovery mapping、A/B subscription identity、callback count
  與最後通知時間；
- 「剛才發生什麼」：由 `make logs` 即時 follow guest journald 與 Host Docker logs，查看
  原始 process log。

所有由本 repo 啟動的長時間 process 都維持 stdout／stderr logging：Guest process（包括
第一版 consumer）交給 systemd journald，Host ML process 交給 Docker log driver。
`logs.sh` 不要求 PyAnLF／PyMTLF 或其他 component 新增固定路徑 application log，也不把 log
copy 回 repository；它組合 multi-machine `journalctl --follow` 與 `docker compose logs
--follow`，並輸出例如：

```text
[core/nrf] ...
[core/nwdaf-c] ...
[path-a/nwdaf] ...
[host/pyanlf-a] ...
[host/pymtlf-a] ...
[path-b/nwdaf] ...
[host/pymtlf-b] ...
[core/nwdaf-consumer] ...
```

預設 view 只 follow 與目前 analytics／FL 路徑直接相關的 units，避免把所有 NF debug log
混成不可讀輸出；使用者可用 VM／service filter 看單一 process，或明確要求 full stack。
Ctrl-C 只結束 Host 上的 log follow sessions，不停止 container、consumer、subscription、
NF 或 VM。

為讓三台 VM log 可比較，guest setup 必須啟用同一可靠 time source，`preflight`／
`services-status` 回報相對 Host 的 clock skew；超過容許值時不得把跨 VM log ordering 當成
可靠證據。Journald 與 Docker log rotation 都採明確且受磁碟上限約束的 policy，只提供近期
診斷；它們不是實驗結果保存機制。若日後需要可回放或長期稽核，再另立
session／collection 設計，不在第一版 live observation 中暗中加入。
