# Repository、Component 與 Topology

[返回基礎建置計畫](plan.md)

## 5. 暫定目錄

```text
5G_NWDAF_Infrastructure/
├── LICENSE
├── README.md
├── Makefile
├── Vagrantfile
├── .gitmodules
├── NFs/
│   ├── amf/                  # upstream free5GC
│   ├── ausf/                 # upstream free5GC
│   ├── nrf/                  # team fork / R18 discovery
│   ├── nssf/                 # upstream free5GC
│   ├── pcf/                  # upstream free5GC
│   ├── smf/                  # smf-nwdaf-ext
│   ├── udm/                  # team fork
│   ├── udr/                  # team fork
│   ├── upf/                  # team go-upf
│   ├── nwdaf/                # Go NWDAF
│   └── adrf/                 # ADRF
├── ML/
│   ├── PyAnLF/
│   └── PyMTLF/
├── containers/
│   └── ml/                   # 共用 CUDA base/layer 與 PyAnLF/PyMTLF image targets
├── compose.yaml              # 五個 ML service；不管理 VM 或 guest NF
├── RAN/
│   └── UERANSIM/
├── kernel/
│   └── gtp5g/
├── webconsole/               # optional free5GC webconsole
├── config/
│   ├── default/              # committed, directly runnable baseline
│   │   ├── nrfcfg.yaml
│   │   ├── amfcfg.yaml
│   │   ├── smfcfg.yaml
│   │   ├── ...               # other NF / NWDAF / ML configs
│   │   ├── upfcfg-a.yaml
│   │   ├── upfcfg-b.yaml
│   │   ├── nwdafcfg-a.yaml
│   │   ├── nwdafcfg-b.yaml
│   │   ├── nwdafcfg-c.yaml
│   │   ├── consumer.yaml
│   │   ├── ueransim/
│   │   │   ├── gnb-a.yaml
│   │   │   ├── gnb-b.yaml
│   │   │   └── ue1.yaml ... ue6.yaml
│   │   └── manifest.yaml
│   └── local/                # gitignored rendered/manually adjusted complete sets
├── experiments/
│   ├── examples/             # committed scenario + Path A/B traffic definitions
│   └── local/                # gitignored user experiment definitions
├── tools/
│   └── nwdaf-consumer/       # infra-owned subscription/callback CLI
├── scripts/
│   ├── host/
│   │   ├── preflight.sh
│   │   ├── config-render.py
│   │   ├── config-check.py
│   │   ├── services-start.sh
│   │   ├── services-stop.sh
│   │   ├── services-status.sh
│   │   ├── ml-start.sh
│   │   ├── ml-stop.sh
│   │   ├── ml-status.sh
│   │   ├── subscriptions-start.sh
│   │   ├── subscriptions-stop.sh
│   │   ├── subscriptions-status.sh
│   │   ├── observe.sh
│   │   └── logs.sh
│   └── guest/
│       ├── common.sh
│       ├── config-activate.sh
│       ├── core.sh
│       ├── path.sh
│       └── systemd/           # disabled-by-default service unit templates
└── testbed.yaml
```

目錄原則：

- 所有 3GPP NF implementation submodule 都在 `NFs/`；
- PyAnLF／PyMTLF 是 containing NWDAF 的 ML backend，不假裝成獨立 3GPP NF，因此
  放在 `ML/`；
- UERANSIM 是 RAN／UE emulator，放在 `RAN/`；
- `gtp5g` 是 guest kernel dependency，放在 `kernel/`；
- `webconsole` 依 free5GC superproject 慣例保留在 root，作為 optional management
  tool；
- `config/` 比照 free5GC 保存 component 實際讀取的 native 設定；`default/` 是可直接執行
  的 committed baseline，`local/` 保存 gitignored complete sets；
- `experiments/examples/` 保存可執行的 committed scenario／traffic definitions，
  `experiments/local/` 供使用者複製修改；這些不是 test fixtures；
- `tools/nwdaf-consumer/` 是 infrastructure-owned 實驗控制工具，不是 NF submodule，也不從
  `nwdaf-resources` 整包帶入；
- `containers/`／`compose.yaml` 只定義 Host 上的 ML runtime；PyAnLF 與 PyMTLF 使用兩種
  image target，五個長時間 service 各自一個 container，不把多個 daemon 綁進同一個
  container；
- `scripts/host/` 在實體機協調 Vagrant 與 Docker，`scripts/guest/` 由 Vagrant 在 VM 內執行；
- 第一版不先建立空的 `tests/`、泛用 `fixtures/` 或頂層 `patches/`；有實際驗證程式
  或相容 patch 時再放到最接近 owner 的位置；
- 不建立泛用 `resources/`，不嵌入 `free5gc-main/`，也不嵌入 `nwdaf-resources/`。

## 6. 第一版 Component Scope

### 6.1 必須納入並通過 E2E

| 路徑 | 來源類型 | 第一版用途 |
| --- | --- | --- |
| `NFs/nrf` | team fork | NF registration、R18 discovery、NWDAF role／scope resolution |
| `NFs/amf` | pinned upstream | NGAP、registration、UE location／TAI、PDU Session control path |
| `NFs/smf` | team `smf-nwdaf-ext` | session、TAI-aware UPF selection、PCF、Nsmf/Nupf Event Exposure |
| `NFs/udr` | team fork | subscriber、group 與 session-related data access |
| `NFs/udm` | team fork | authentication/subscription ownership、group 與 SMF registration data |
| `NFs/ausf` | pinned upstream | UE authentication path |
| `NFs/nssf` | pinned upstream | slice selection path |
| `NFs/pcf` | pinned upstream | SM Policy Association；full-core profile 不允許省略 |
| `NFs/upf` | team go-upf | two-UPF user plane、PseudoDriver、Nupf Event Exposure |
| `NFs/nwdaf` | team implementation | A／B analytics 與 FL Client；C model owner／coordinator |
| `NFs/adrf` | team implementation | analytics/training data 與 ML model persistence |
| `ML/PyAnLF` | team implementation | A／B inference、collection 與 accuracy path |
| `ML/PyMTLF` | team implementation | A／B FL Client、C FL Server／monitor coordination |
| `RAN/UERANSIM` | pinned upstream | two gNB、six UE、two TAI registration/PDU Session |
| `kernel/gtp5g` | pinned upstream | UPF guest kernel data-plane dependency；先測原版，必要時才另加相容 patch |

MongoDB 是 runtime service，不是 source submodule。其版本、bind、database ownership、
health check、資料清理範圍與 persistence policy 必須由 scripts 與 component config
明確管理。

### 6.2 第一版包含但預設不啟動

- `webconsole`：提供手動 subscriber 管理與 onboarding；後續自動化 E2E 仍使用受限、
  可重現的 subscriber/group data，不讓 UI 操作成為實驗前置條件。

### 6.3 延後支援

| NF | 延後理由 |
| --- | --- |
| CHF | 目前 scenario 不驗證 charging path |
| NEF | 目前 scenario 不經由 NEF exposure path |
| BSF | PCF profile 已明確關閉 BSF integration |
| N3IWF | 第一版只支援 3GPP access |
| TNGF | 第一版不支援 trusted non-3GPP access |

這些 NF 未來應由新增 scenario 與 acceptance criteria 帶入，不只是在 `NFs/` 多加一個
submodule 或讓預設 launcher 多啟動一個 process。

## 7. Submodule 與版本策略

1. Parent gitlink 是實驗 source revision 的唯一 authoritative lock；branch 名只作
   維護提示，runner 不自動 checkout branch。
2. 公開 repository 的 `.gitmodules` 優先使用 HTTPS URL，讓匿名 clone 不依賴 SSH
   key；開源前必須確認所有 required remote 都可讀且 license 相容。
3. 文件與 CI 使用 `git submodule status --recursive` 產生 source manifest；build/status
   可回報 dirty flag、binary hash 與 config identity，但第一版不建立 per-run archive。
4. 禁止在可重現 run 中使用 `git submodule update --remote`、guest-side branch clone
   或未固定版本的 download URL。
5. Component feature 修改在各自 repository／branch 完成；整合通過後才更新 parent
   gitlink。
6. 不用 submodule 保存 generated config、model artifact、MongoDB data、VM disk、venv、
   Go cache、log、pcap 或 evidence。

## 8. 三 VM 加 Host ML Containers 的 Compact Topology

### 8.1 角色分配

| VM | 主要元件 | 邊界 |
| --- | --- | --- |
| `core` | NRF、AMF、SMF、UDR、UDM、AUSF、NSSF、PCF、NWDAF-C、ADRF、MongoDB；optional webconsole | shared control、model owner／FL coordinator 的 Go NF、persistent services |
| `path-a` | UPF-A、gNB-A、UE1–3、NWDAF-A | TAI `000001` network、analytics／FL client 的 Go NF |
| `path-b` | UPF-B、gNB-B、UE4–6、NWDAF-B | TAI `000002` network、analytics／FL client 的 Go NF |

Python ML backend 改在實體 Host 以五個獨立 container 執行，但只建置兩種 image：

| Container | Image 類型 | Device default | 角色 |
| --- | --- | --- | --- |
| `pyanlf-a` | PyAnLF | CPU，可設定 `cuda:0` | Path A inference、collection、accuracy |
| `pymtlf-a` | PyMTLF | `cuda:0` | Path A FL Client |
| `pyanlf-b` | PyAnLF | CPU，可設定 `cuda:0` | Path B inference、collection、accuracy |
| `pymtlf-b` | PyMTLF | `cuda:0` | Path B FL Client |
| `pymtlf-c` | PyMTLF | CPU | FL Server、sample-count-weighted FedAvg 與 model serialization |

每個 container 只執行一個長時間 process，不把 PyAnLF、PyMTLF 或多個角色包成單一
supervisor container。PyAnLF 預設使用 CPU，是為兩個 PyMTLF client 保留 RTX 3080 的
10 GiB VRAM；是否能讓 A/B training 同時使用 GPU，必須由實機 VRAM smoke 決定，不能只
根據單一 process 測試推論。

初始 logical mapping 延續已驗證 scenario：

- PLMN `466/92`；
- S-NSSAI `sst=1, sd=010203`；
- DNN `internet`；
- Path A：TAI `000001`、UE pool `10.60.0.0/16`；
- Path B：TAI `000002`、UE pool `10.61.0.0/16`；
- 同一 Internal Group 的六個 UE 分散在兩個 TAI；
- A 可切換到 degraded traffic profile，B 保持 stable control profile。

Phase 7 不以單一 dataset 同時承擔所有用途。Infrastructure 提供一個保留舊 testbed
CAT transition 語意的主 example，以及一個縮短等待時間的 FL closure smoke；兩者共用相同
三 VM／五 container topology、真實 subscription／collection／ADRF 路徑與兩輪 FedAvg，
但不得互相取代 acceptance claim。詳細 profile contract 見 9.4.1。

### 8.2 為什麼不繼續採用全 VM

5GC、UPF、UERANSIM 與 Go NWDAF 仍留在 VM，因為它們需要清楚的 kernel、network plane、
TAI 與 failure boundary；Python ML backend 移到 Host container，則是本機只有一張 RTX
3080 所造成的實際部署選擇，不是因為 container 比 VM 更像 3GPP NF。

一般 KVM／VirtualBox guest 不能像 container 一樣自然共用同一張實體 GPU。PCI
passthrough 通常把整張 GPU 專屬交給一台 VM；交給 `path-a` 後，Host 與 `path-b` 便不能
同時正常使用。RTX 3080 也不應假設具有可供兩台一般 VM 分割使用的 NVIDIA vGPU 能力，
更不能把相應硬體、driver 與授權要求當作公開 testbed 的預設前置。

新增第四台「GPU VM」雖可在該 VM 內啟動五個 ML process，但仍有以下問題：

- GPU 被整台 guest 獨占，增加 passthrough、IOMMU、driver 與 provider 相依性；
- 多出一份 Ubuntu、virtual disk、RAM 與 CUDA runtime 開銷；
- ML backend 最終仍不在 `path-a`／`path-b` guest，並未比 Host containers 更忠實地保留
  logical A/B placement；
- PyAnLF／PyMTLF process、config、health 與 log 仍需分別管理，將它們放進同一 VM 不會
  消除 service orchestration 問題。

Host containers 可在保留五個獨立 service identity 的同時共享同一 NVIDIA driver、CUDA
runtime image layers 與 GPU，且不在兩台 Path disk 重複安裝數 GiB 的 Python/CUDA
dependencies。代價是必須明確驗證 Host-to-VM route、published ports、firewall 與 endpoint
advertisement；這些設定納入 topology/config preflight，不靠 Docker 動態 container IP。

若未來有至少兩張可分別 passthrough 的 GPU，或正式目標改為完全 CPU-only，才重新評估
全 VM placement。以目前單張 RTX 3080、兩個 FL client 都需 GPU 的條件，三台 network VM
加五個 Host ML containers 是第一版部署基線。

### 8.3 Network planes

第一版 topology 至少明確區分：

- management/provisioning：Vagrant SSH、artifact deployment、health collection；
- SBI/service：core NF、NWDAF、Host ML published endpoint、ADRF、MongoDB 與 callback；
- N2：兩個 gNB 到 AMF；
- N3-A／N3-B：各 gNB 到對應 UPF；
- N4：SMF 到兩個 UPF；
- N6-A／N6-B：UE data network／egress；
- UE pools：兩條 path 不重疊。

Public default 應優先使用 isolated private／host-only network 與受控 NAT，不把所有
plane bridge 到實驗室實體 LAN。需要另一組實體 bridge、provider storage expectation
或 lab gateway 時，使用者建立一份完整且明確選用的 `TESTBED=<path>` definition，
不使用 implicit local overlay。衝突診斷會提示風險，但不作為 start 授權。

五個 container 使用一般 Docker bridge network，對 VM 發布固定 Host port；Go NWDAF 與
PyAnLF／PyMTLF config 只引用 stable Host SBI address 加 published port，不引用 Docker
container IP。實體 Host 必須能到達三台 VM 的 SBI address，三台 VM 也必須能到達所選
Host SBI address；exact interface、address、route 與 firewall rule 必須在建立 container
前由 site inventory 和 provider smoke 確認。第一版不預設 `host` networking 或 macvlan。

### 8.4 初始資源預算

資源配置以目前 full-core scenario 的實際併發度作為基線，不作 capacity benchmark
承諾：

| VM | Compact RAM | vCPU | Primary logical capacity | 預期 guest 使用量 |
| --- | ---: | ---: | ---: | ---: |
| `core` | 4 GiB | 4 | 40 GiB | 約 20 GiB 內 |
| `path-a` | 3 GiB | 3 | 40 GiB | 約 18 GiB 內 |
| `path-b` | 3 GiB | 3 | 40 GiB | 約 18 GiB 內 |
| 合計 | 10 GiB | 10 | 120 GiB logical | 約 56 GiB 內 |

這是把 Python environment 與 training/inference RSS 移出 guest 後的 provisional target，
不是已由完整 VM E2E 證明的 minimum。Core 4 GiB 包含 full control plane、MongoDB、ADRF
與 NWDAF-C；各 Path 3 GiB 包含 UPF、gNB、三個 UE 與 NWDAF。MongoDB WiredTiger cache
仍先限制為 256 MiB。第一次 bring-up 必須記錄 idle、registration/PDU Session、analytics
baseline 與 FL 各 stage 的 guest peak RSS；若發生 guest OOM、reclaim thrashing 或明顯
latency，先以 512 MiB step 調整，不把偶然成功當作 sizing evidence。

各 Path 的 3 GiB 也必須包含 go-upf PseudoDriver 的 dataset replay，不可只算 idle UPF。
下表是 2026-08-09 對舊 pinned go-upf `pre_data` 的歷史 RAM probe，不再是新情境預設輸入：

| Path | Dataset | Compressed bytes | 實際掃描 rows | 單 context probe peak RSS |
| --- | --- | ---: | ---: | ---: |
| A | `pre_data/group1/training_packets_run001.parquet` | 21,830,425 | 2,720,063 | 約 22.9 MiB process RSS |
| B | `pre_data/group2/training_packets_run001.parquet` | 44,050,349 | 5,759,921 | 約 22.7 MiB process RSS |

目前 implementation 逐列掃描 Parquet 兩次，將資料聚合成 `(subscription, UE, window)` maps
與 Phase 2 windows，不把數百萬筆 raw rows 全部常駐記憶體。因此上述 dataset 本身暫時
不足以支持直接把 Path RAM 增加數 GiB；但 probe 只隔離 PseudoDriver algorithm，不包含
完整 UPF、Parquet page cache、gtp5g、gNB、三個 UE、NWDAF、journald 或多 subscription
fan-out，不能當成 VM peak。

新情境改由 Infrastructure 依 topology/profile 生成每 Path 27,000 rows、約 0.84 MiB 的
content-addressed Parquet；這大幅縮小磁碟輸入，但不能直接取代完整 replay peak RAM 實測。

第一輪實機必須在 subscription warm-start 前保有至少 512 MiB guest `MemAvailable`，並以
systemd memory accounting／VM metrics 記錄 UPF replay 前、scan/aggregation peak、Phase 2
retained windows 與 replay 完成後的 RSS。未達 headroom、發生 OOM/reclaim thrashing，或
dataset／subscription fan-out 超出 manifest 時，停止 E2E 並將該 Path 以 512 MiB step
增加；不允許因 3 GiB target 而讓 kernel 強制 kill UPF 或其他 NF。

VirtualBox/Bento 路線先把 40 GiB 視為每台 primary disk 的 logical floor。Vagrant
`vm.disk ... primary: true` 是 primary disk expansion interface，不能把 base box disk
縮小；若 box 本身為 40 GiB，設定 18／20 GiB 不會得到較小的 primary disk。三台因此是
120 GiB logical capacity，但 VMDK/VDI 採 dynamic allocation，不會在建立時立刻實佔
120 GiB。

移除 guest PyAnLF／PyMTLF virtualenv 和 CUDA packages 後，Core 約 20 GiB、各 Path 約
18 GiB 應視為 guest used space／Host backing-file allocation 的量測目標，而不是 virtual
disk size。第一次 clean build 應記錄 base box、toolchain、source、artifact 與 journal
各自占用。若真要把 logical disk 做到 40 GiB 以下，必須選用或自行建置較小的 base box，
不能靠一般 Vagrant resize；在 dynamic disk 未實際膨脹時，這通常不值得增加維護成本。

Host 另需承擔五個 container、共享 PyTorch/CUDA image layers 和 CPU-side tensor/data。
Preflight 暫以 VM 10 GiB 加 Host/ML 6 GiB available RAM 作為自動啟動門檻，並另外回報
Docker data-root free space；container RSS 不是在啟動 VM 時一次預留完，但 training 時會
按實際 workload 成長。GPU preflight 必須確認 RTX 3080、R535 driver、Docker 與 NVIDIA
Container Toolkit CDI support；Host 不需要安裝完整 CUDA toolkit，image 使用 PyTorch 2.5.1
的 CUDA 12.1 runtime。實機 Docker 27.4.1 無法只靠 reload 啟用 native CDI device request，
因此第一版採既有 rootful Docker daemon 加 NVIDIA runtime CDI mode：Host 安裝
`nvidia-container-toolkit-base`、產生 `/var/run/cdi/nvidia.yaml`，並註冊 `nvidia` runtime，
但保留 `runc` 為 `default-runtime`。Compose 以 `runtime: nvidia` 和 CDI-qualified
`NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=all` 交付 GPU。runtime registration 可由 daemon
reload 生效，不把共用 Docker daemon restart 當成正常安裝步驟。啟動 production project
前仍須通過 CDI inventory 和 disposable container probe。A/B 同時 training 的 VRAM peak、
OOM 行為與 sequential fallback 必須在長時間運行前完成 bounded smoke。

此 Host 的 system Docker 為多人共用且 `live-restore=false`；2026-08-09 清點時有八個
running containers，完整 daemon restart 會中斷其他使用者，且部分 service 沒有自動 restart
policy。因此 CDI probe 失敗時不得由 automation 改寫 `/etc/docker/daemon.json` 或 restart
daemon；先停止本 testbed 的 GPU activation，再另行評估 daemon reload、維護窗口或專用
rootless Docker。Rootless daemon 只作 fallback：目前 Host 使用 cgroup v1，無法假設
rootless mode 能落實 Compose 的 CPU／RAM limits，而且會建立獨立 image/data store 並增加
networking 驗證與磁碟占用。

正式建立 VM 前仍強烈建議量測 host available RAM、swap、free disk、Docker image/cache 與
現有 workload。沒有 swap 不代表必然失敗，但 shared Host 在瞬時壓力下會更快進入 OOM
killer，因此 preflight 必須顯著警告長時間無人運行的風險，但 RAM、swap 與
storage reference thresholds 不作為 start hard gate；deployment script 也不自動建立或清除 swap。動態 disk ceiling 加上
provider metadata、box image、container layers 與暫存下載仍不適合在磁碟壓力下啟動。

清理不等於直接刪目錄。執行前必須先確認每台舊 VM 的 Vagrant project、provider name、
UUID、storage path、disk/snapshot size 與 state，保存 guest-only config／script／data／log
或整台 VM 的可恢復副本，驗證 backup manifest 後，再列出 exact removal targets 讓
使用者確認。預設優先由原 Vagrant project 執行 destroy；只有 Vagrant metadata 已失效
時才考慮 provider-specific unregister/delete。不得以 broad path 或 glob 清除。

2026-08-09 在實際 Host context 的唯讀 smoke 中，VirtualBox 6.1.50 的 CLI／driver 可初始化，
Docker 27.4.1 與 Compose 2.32.1 也可由目前使用者存取；因此 VirtualBox 是第一個可測的
provider candidate，但仍不是完成 network gate 的意思。受限 sandbox 看不到 `/dev/vboxdrv`
或 Host netlink/socket，不可把該視圖誤判為 Host driver 壞掉。正式 `vagrant up` 前仍須由
preflight 重跑 provider、host-only address、route 與 port conflict；Topology 與 config
不綁死 provider-specific 介面名稱。

### 8.5 單一 multi-machine Vagrant project

Root `Vagrantfile` 在同一個 project 內定義三個 named machines：

```text
core
path-a
path-b
```

常用操作因此集中在 repository root：

```text
vagrant up
vagrant up core
vagrant ssh path-a
vagrant status
vagrant halt
```

這三台仍是三個獨立 provider VM；單一 `Vagrantfile` 只代表它們由同一份 topology、
network 與 lifecycle 定義管理，不代表合併成一台 VM。相較舊環境每個 VM 目錄各有
一份 `Vagrantfile`，可減少 box、network 與 shared setting 重複。

Repository 的 `.vagrant/machines/{core,path-a,path-b}/` 只保存 Vagrant ID、SSH 與
provider metadata，必須 gitignore；它不是虛擬磁碟。第一版使用 VirtualBox，真正的 VM
disk 位於 VirtualBox global machine folder，box cache 則位於 Vagrant 自己的 cache，兩者
都在 repository 外。

Preflight 直接查詢 provider 的實際 VM storage location 與 free space，不透過額外
local overlay 推測，也不在 `vagrant up` 時靜默修改 provider 的 global machine folder。
任何 global storage 變更都屬於獨立 host setup，必須先確認對既有 VM 的影響。
