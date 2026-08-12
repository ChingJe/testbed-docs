# `5G_NWDAF_Infrastructure` 建置與遷移計畫

建立日期：2026-08-05
最近更新：2026-08-10

狀態：本機 infrastructure baseline 與 component pinning 已實作；PyAnLF／PyMTLF 的可設定
GPU 支援已完成並推送 component branch。三台 VM 加 Host Docker ML services 的 topology、
native config、renderer/checker、non-mutating preflight、兩種 image target、five-service
Compose、正式 ML lifecycle 與 bounded CPU health/lifecycle smoke 已完成；尚未驗證 GPU
container runtime、Host-to-VM network、建立 remote、建立新 VM 或完成 privileged
full-scenario E2E。

## 1. 目的

本計畫定義一個新的、可公開釋出且可重現的 5G NWDAF 實驗環境。暫定 repository
名稱為 `5G_NWDAF_Infrastructure`。它將取代舊 `5G_Infrastructure` 作為新版
NWDAF full-core 實驗的整合層，但不直接繼承舊 repository history 或 working tree。

第一個完整支援目標是目前已在單機 runner 驗證的 three-NWDAF、two-TAI、two-UPF
federated-learning closed loop：

- NWDAF-A／B 分別服務 TAI A／B，負責 analytics、model consumption、accuracy
  reporting 與 FL Client；
- NWDAF-C 擁有模型，負責 Model Provision、Model Monitor coordination 與 FL Server；
- UE 經真實 UERANSIM registration、authentication、PDU Session、serving-SMF
  resolution 與 UPF selection；
- UPF Event Exposure、ADRF、model monitoring、federated retraining、model publication
  與 reprovision 形成可驗證的 E2E evidence chain。

本計畫同時處理：

- 新 integration repository 的責任與目錄；
- component submodule 與版本固定策略；
- 三台精簡 VM 與 Host ML containers 的角色、資源及網路邊界；
- component config、testbed topology 與本機 override 邊界；
- host orchestration 與 guest setup／build／service lifecycle；
- 從舊 testbed 遷移並最終退場的安全條件。

## 2. 現況與問題

舊 `5G_Infrastructure` 適合保存歷史實驗與實驗室網路設定，但不適合作為新版公開
環境的起點：

- 它不是目前使用者主要維護的 repository，且長期保存大量 tracked／untracked
  本地差異；
- source 有時由 guest provisioning 直接 clone，有時由 host submodule mount 後再
  copy，實際版本不易由 parent repository 完整回答；
- 多台 VM 各自保存 free5GC checkout、build cache 與可修改 source，造成重複占用和
  version drift；
- Vagrant provisioning、`setup.sh`、guest source tree 與 runtime state 共同決定
  最終狀態，重建邊界不清楚；
- 舊雙 path profile 的兩個 gNB 都使用同一 TAC，不等於新版 two-TAI scenario；
- 盤點初期 host 只剩約 56 GiB 可用磁碟、swap 幾乎用滿，且 VirtualBox kernel driver
  不可用；舊 VM 清除後雖已回收空間，共用主機的 RAM、disk 與 provider 狀態仍會變動，
  每次新建或長時間運行前都必須重新 preflight。

舊環境仍保有不可遺失的 site-specific 資訊。其 host interface、bridge、IP、route、
VM NIC、MongoDB 與舊 topology 已另行保存於
[local-network-settings-inventory-2026-08-05.md](../reports/local-network-settings-inventory-2026-08-05.md)。
這些資訊是遷移輸入，不是新環境的 public default。

## 3. 設計原則

1. **新的整合邊界**：新建 repository，不複製舊 Git history，也不把舊 dirty tree
   當作基底。
2. **可重現 source identity**：所有直接整合的 source repository 由 parent
   submodule gitlink 固定 exact commit。
3. **類 free5GC superproject 結構**：NF source 集中在 `NFs/`，但不再嵌入一份
   完整 free5GC main repository。
4. **source 不由 guest 決定**：VM provisioning 不自行 clone branch 或 latest
   source；guest 只接收 parent 已固定的 source／artifact 與設定。
5. **free5GC-like config ownership**：`config/` 只保存 NF、RAN、ML 與相關 service
   實際讀取的設定；三 VM topology 與本機差異由 root-level testbed files 管理。
6. **最小但完整**：第一版只支援已驗證的 full-core path；未驗證 NF 不因 free5GC
   main 預設啟動就宣稱支援。
7. **不內嵌 `nwdaf-resources`**：它保留為獨立開發／回歸 repository，不成為新
   infrastructure 的 submodule 或執行前置；只在確認 ownership 後移植必要小工具。
8. **generated state 與 source 分離**：VM disk、container image/layer、binary cache、Python
   environment、MongoDB、ADRF data、log 與 pcap 都不是 submodule。
9. **第一版不支援憑證**：不提供 certificate、TLS 或 OAuth 管理；環境明確限定
   在隔離實驗網路以 HTTP 執行，不暗示 production security readiness。
10. **先保存、再清理舊 VM**：由於本機空間不足，新 VM 建立前先盤點、備份並在使用者
    確認 exact targets 後移除舊 local VM；舊 `5G_Infrastructure` repository、本地腳本
    與 migration inventory 仍保留到新環境通過 fresh-clone E2E。
11. **分離 execution lifecycle**：VM power、guest 5GC／Go services、Host ML containers、
    NWDAF subscriptions 與 traffic/degradation action 分別啟停；任何一層都不隱含重建或
    停止其他層。
12. **GPU 留在 Host**：Python ML backend 以 Docker 使用實體機 GPU，不把 CUDA runtime
    和 NVIDIA device passthrough 塞入一般 Vagrant guest；CPU/GPU device selection 仍由
    component config 控制，不寫死在程式碼。

## 4. Repository 責任與邊界

`5G_NWDAF_Infrastructure` 負責：

- component revision 組合；
- build orchestration 與 artifact identity；
- VM topology、network、single-project Vagrant orchestration 與 Host container placement；
- host-side preflight、VM／service／container／subscription start／stop／status；
- guest-side OS setup、role-local non-ML build、network 與 service lifecycle；
- PyAnLF／PyMTLF image build、GPU runtime gate 與 Host-to-VM endpoint wiring；
- component config 與本機 testbed override；
- public quick start 與後續 E2E bring-up。

它不負責：

- 在 parent repo 直接維護各 NF、NWDAF、PyAnLF 或 PyMTLF 的 feature code；
- 保存 VM image、runtime database、dataset output、log 或 build cache；
- vendor 整個 `nwdaf-resources` 或要求使用者另行 clone 它才能建立三 VM；
- 承諾 production security、HA、capacity benchmark 或真實 application traffic
  performance；
- 自動套用本實驗室的 public interface、IP 或 bridge。

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
│   ├── generated/            # gitignored renderer outputs
│   └── local/                # gitignored manually maintained sets
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
├── testbed.yaml
├── testbed.local.example.yaml
└── testbed.local.yaml         # gitignored
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
  的 committed baseline，`generated/`／`local/` 不作為 source history；
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
plane bridge 到實驗室實體 LAN。只有 gitignored `testbed.local.yaml` 可以選擇實體
bridge、provider-specific storage expectation 或 lab gateway，且必須通過衝突檢查後
才套用。

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

正式建立 VM 前仍必須量測 host available RAM、swap、free disk、Docker image/cache 與
現有 workload。沒有 swap 不代表必然失敗，但 shared Host 在瞬時壓力下會更快進入 OOM
killer，因此 preflight 必須警告並禁止無人長時間運行；是否把 swap 設為 hard gate，應由
本機政策另定，不在 deployment script 中自動建立或清除 swap。動態 disk ceiling 加上
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

`testbed.local.yaml` 可以記錄預期 provider 與 VM storage location，讓 preflight 回報
實際位置和 free space；它不應在 `vagrant up` 時靜默修改 provider 的 global machine
folder。任何 global storage 變更都屬於獨立 host setup，必須先確認對既有 VM 的影響。

## 9. 設定檔與 Testbed 定義

### 9.1 `config/default/`：可直接執行的 baseline

`config/default/` 保存 component 真正讀取的完整 native config，延續 free5GC-style
flat filenames；只有 UERANSIM 等本來就適合分組的檔案使用子目錄。Host orchestration
再依 placement 將同一 config set 的適當檔案交給 Core、Path A、Path B 與五個 Host
containers。內容涵蓋：

- free5GC core NF、UPF-A/B 與 `uerouting`；
- NWDAF-A/B/C、ADRF、PyAnLF、PyMTLF 與 optional webconsole；
- gNB-A/B 與 UE1–6；
- 由同一 topology 產生的 `network/{core,path-a,path-b}.yaml` process alias 清單；
- SBI、PFCP、N2/N3/N4/N6 address、NRF/Mongo URI、PLMN、S-NSSAI、DNN、TAI、UE pool、
  callback 與 discovery fields。

`default/` 不是未修改的 upstream sample，而是本專案 public isolated three-VM topology 的
完整、reviewed、可直接執行設定；fresh clone 不執行 renderer 也能使用。`manifest.yaml`
記錄每組 baseline 的來源 repository／revision、schema／generator compatibility 與重要
topology identity。需要從 pinned free5GC baseline 匯入時，必須記錄來源 revision、license
與本專案差異；執行時不要求另有完整 `free5gc-main` checkout。

### 9.2 Config set 選擇與人工修改

未指定時，所有 setup、systemd unit、status 與 preflight 都使用 `config/default/`。Effective
config directory 的優先順序是：Make command 的 explicit `CONFIG_DIR`、
`testbed.local.yaml` 的 `configDir`、最後才是 `config/default/`。使用者
若想逐份理解與調整 native YAML，可將整組複製到 `config/local/<name>/`，修改後以同一套
`config-check` 驗證，再透過 `testbed.local.yaml` 的 `configDir` 明確選用。`config/local/`
預設 gitignored；若某個組合後續值得公開，應 review 後提升為 `config/<name>/` 的 committed
config set，而不是提交 local override。

選擇單位必須是完整 config set，不能 Core 使用 default、Path A 使用 generated、Path B
又使用另一套 local config。`vm-up`／`services-start` 必須回報 effective testbed definition、
config directory 與 manifest/hash，且不得在背景重寫 `config/default/`。例如：

```text
make vm-up TESTBED=testbed.my-lab.yaml
make services-start TESTBED=testbed.my-lab.yaml CONFIG_DIR=config/local/my-lab
```

### 9.3 選用的 config renderer

對需要成套變更 PLMN、TAI、NF address、UPF pool 或 A/B mapping 的情況，提供
`config-render.py`，但 renderer 是便利工具，不是 public default 的必要前置。它讀取：

1. read-only `config/default/` native baseline；
2. 一份符合 `testbed.yaml` schema 的明確 topology definition；
3. 僅限 provider／physical-host 欄位的 optional `testbed.local.yaml`。

輸出只寫入 `config/generated/<name>/`，維持和 default 相同的 file layout，產生三台
Guest 各自的 network alias YAML，並產生
包含 baseline hash、definition hash、generator revision、generated file list 與 effective
network identity 的 manifest。Renderer 必須用 typed YAML/object mutation 修改已宣告欄位，
不得用文字取代、regex 或任意 deep merge 猜測 component schema，也不得依賴另一份
`free5gc-main` checkout。

`config-check.py` 同時支援 default、generated 與 local sets，至少檢查：

- Vagrant/testbed network 與 component bind／advertise address 一致；
- 每個 process alias 的 guest owner、network、prefix、anchor 與 address 均由同一份
  testbed definition 推導且沒有重複；
- NRF URI、NF identity／service endpoint 與 NWDAF discovery scope 一致；
- AMF TAI、SMF AN-to-UPF mapping、UPF N3/N4／UE pool、UERANSIM gNB／UE 一致；
- A/B/C role、Internal Group、consumer TAI subscriptions 與 callback reachability 一致；
- address、port、UE pool、Linux interface name 與 NF instance ID 沒有衝突。

使用者可以選擇 renderer，或自行慢慢建立完整 local config set；兩條路最後都必須通過
相同 check。第一版不建立泛用 template language、Jinja hierarchy 或另一層 profiles/sites。

### 9.4 `testbed.yaml` 與本機 override

Root `testbed.yaml` 是 public default topology definition，描述：

- `core`、`path-a`、`path-b` 的 RAM、vCPU 與 disk budget；
- management、SBI、N2、N3-A/B、N4、N6-A/B network；
- component 到 VM／Host container 的 placement，以及 Host published ML endpoints；
- PLMN、S-NSSAI、DNN、TAI、UPF／UE pools 與生成 component config 所需的 network
  identity；
- Vagrant／VirtualBox defaults 與 two-TAI／two-UPF reachability。

需要另一組完整 topology 時，使用者可複製 `testbed.yaml` 到自選 definition file，經
Makefile 的 explicit `TESTBED=<path>` 選用；同一個 definition 同時交給 Vagrant、renderer
與 config checker，避免兩份網路 source of truth。

`testbed.local.example.yaml` 說明可覆寫欄位；`testbed.local.yaml` gitignored，只保存
physical Host 差異與 active `configDir`，包含 VirtualBox storage、Docker storage 與 Host SBI
bind address。未被程式讀取的 bridge/interface、port-forwarding 或 gateway 欄位不預先放進
example；確有 deployment contract 時才另案加入。
Local bind override 不改變 VM 所使用的 advertised address；兩者不同時，Host 必須具備
對應 route／forwarding。Local override 不直接改變 TAI、UE identity、NWDAF ownership 或
FL assertion；這些語意變更必須放進 explicit topology definition 和完整 config set。

#### `testbed.yaml` schema 草案

以下是預期 shape，不是已凍結的實驗室 IP 配置。`192.168.56.0/21` 內的分段先作 public
isolated default 候選；Phase 4 必須經 provider feasibility 與 address-conflict preflight
後才定案。

```yaml
schemaVersion: 1
name: dual-tai-default

config:
  directory: config/default

guest:
  os: ubuntu
  release: "22.04"

hostSafety:
  reserveMemoryMiB: 6144
  swapPolicy: warn
  minimumFreeSwapMiB: 1024
  minimumFreeStorageGiB: 120

machines:
  core:
    resources:
      memoryMiB: 4096
      cpus: 4
      diskGiB: 40
    interfaces:
      management: 192.168.56.10
      sbi: 192.168.57.2
      n2: 192.168.58.2
      n4: 192.168.61.2

  path-a:
    resources:
      memoryMiB: 3072
      cpus: 3
      diskGiB: 40
    interfaces:
      management: 192.168.56.11
      sbi: 192.168.57.3
      n2: 192.168.58.3
      n3-a: 192.168.59.2
      n4: 192.168.61.3
      n6-a: 192.168.62.2

  path-b:
    resources:
      memoryMiB: 3072
      cpus: 3
      diskGiB: 40
    interfaces:
      management: 192.168.56.12
      sbi: 192.168.57.4
      n2: 192.168.58.4
      n3-b: 192.168.60.2
      n4: 192.168.61.4
      n6-b: 192.168.63.2

networks:
  management:
    cidr: 192.168.56.0/24
    mode: private
  sbi:
    cidr: 192.168.57.0/24
    mode: private
  n2:
    cidr: 192.168.58.0/24
    mode: private
  n3-a:
    cidr: 192.168.59.0/24
    mode: private
  n3-b:
    cidr: 192.168.60.0/24
    mode: private
  n4:
    cidr: 192.168.61.0/24
    mode: private
  n6-a:
    cidr: 192.168.62.0/24
    mode: private
    egress: nat
  n6-b:
    cidr: 192.168.63.0/24
    mode: private
    egress: nat

mobileNetwork:
  plmn:
    mcc: "466"
    mnc: "92"
  snssai:
    sst: 1
    sd: "010203"
  dnn: internet
  internalGroupId: "00000001-466-92-01"

placement:
  core:
    - nrf
    - nssf
    - udr
    - udm
    - ausf
    - pcf
    - amf
    - smf
    - mongodb
    - adrf
    - nwdaf-c
    - nwdaf-consumer
  path-a: [upf-a, gnb-a, ue1, ue2, ue3, nwdaf-a]
  path-b: [upf-b, gnb-b, ue4, ue5, ue6, nwdaf-b]
  host-containers: [pyanlf-a, pymtlf-a, pyanlf-b, pymtlf-b, pymtlf-c]

mlRuntime:
  engine: docker-compose-v2
  networkMode: bridge
  bindAddress: 192.168.57.1       # physical Host listener candidate
  advertisedAddress: 192.168.57.1 # endpoint used by VMs and callbacks
  services:
    pyanlf-a: {image: pyanlf, publishedPort: 9093, containerPort: 9093, device: cpu}
    pyanlf-b: {image: pyanlf, publishedPort: 9094, containerPort: 9093, device: cpu}
    pymtlf-a: {image: pymtlf, publishedPort: 9092, containerPort: 9092, device: "cuda:0"}
    pymtlf-b: {image: pymtlf, publishedPort: 9091, containerPort: 9092, device: "cuda:0"}
    pymtlf-c: {image: pymtlf, publishedPort: 9292, containerPort: 9292, device: cpu}

coreServices:
  nrf:
    sbi: {network: sbi, address: 192.168.57.10, port: 8000}
  nssf:
    sbi: {network: sbi, address: 192.168.57.11, port: 8000}
  udr:
    sbi: {network: sbi, address: 192.168.57.12, port: 8000}
  udm:
    sbi: {network: sbi, address: 192.168.57.13, port: 8000}
  ausf:
    sbi: {network: sbi, address: 192.168.57.14, port: 8000}
  pcf:
    sbi: {network: sbi, address: 192.168.57.15, port: 8000}
  amf:
    sbi: {network: sbi, address: 192.168.57.16, port: 8000}
    n2: {network: n2, address: 192.168.58.10, port: 38412}
  smf:
    sbi: {network: sbi, address: 192.168.57.17, port: 8000}
    n4: {network: n4, address: 192.168.61.10, port: 8805}
  mongodb:
    endpoint: {network: sbi, address: 192.168.57.18, port: 27017}
    database: free5gc
  adrf:
    sbi: {network: sbi, address: 192.168.57.19, port: 9888}

paths:
  a:
    machine: path-a
    tai: {plmn: "46692", tac: "000001"}
    gnb:
      n2: {network: n2, address: 192.168.58.20}
      n3: {network: n3-a, address: 192.168.59.20}
    upf:
      n3: {network: n3-a, address: 192.168.59.10}
      n4: {network: n4, address: 192.168.61.20}
      n6: {network: n6-a, address: 192.168.62.10}
      eventExposure: {network: sbi, address: 192.168.57.40, port: 8088}
      gtpInterface: upfgtp-a
      uePool: 10.60.0.0/16
      pseudoDriver:
        enabled: true
        mode: hybrid
        profile: fixtures/full-core/traffic/profiles/path-a.json
        dataset:
          file: traffic.parquet
          guestDirectory: /var/lib/5g-nwdaf-infrastructure/datasets/active
          minimumReplayHeadroomMiB: 512
    ues:
      - imsi-466920000000001
      - imsi-466920000000002
      - imsi-466920000000003

  b:
    machine: path-b
    tai: {plmn: "46692", tac: "000002"}
    gnb:
      n2: {network: n2, address: 192.168.58.30}
      n3: {network: n3-b, address: 192.168.60.20}
    upf:
      n3: {network: n3-b, address: 192.168.60.10}
      n4: {network: n4, address: 192.168.61.30}
      n6: {network: n6-b, address: 192.168.63.10}
      eventExposure: {network: sbi, address: 192.168.57.50, port: 8088}
      gtpInterface: upfgtp-b
      uePool: 10.61.0.0/16
      pseudoDriver:
        enabled: true
        mode: hybrid
        profile: fixtures/full-core/traffic/profiles/path-b.json
        dataset:
          file: traffic.parquet
          guestDirectory: /var/lib/5g-nwdaf-infrastructure/datasets/active
          minimumReplayHeadroomMiB: 512
    ues:
      - imsi-466920000000004
      - imsi-466920000000005
      - imsi-466920000000006

analytics:
  nwdaf-a:
    machine: path-a
    nfInstanceId: "11111111-1111-4111-8111-111111111111"
    sbi: {network: sbi, address: 192.168.57.41, port: 8080}
    tai: "000001"
    role: fl-client
  nwdaf-b:
    machine: path-b
    nfInstanceId: "22222222-2222-4222-8222-222222222222"
    sbi: {network: sbi, address: 192.168.57.51, port: 8080}
    tai: "000002"
    role: fl-client
  nwdaf-c:
    machine: core
    nfInstanceId: "33333333-3333-4333-8333-333333333333"
    sbi: {network: sbi, address: 192.168.57.30, port: 8080}
    role: fl-server

  backends:
    pyanlf-a: {runtime: host-container, address: 192.168.57.1, port: 9093}
    pyanlf-b: {runtime: host-container, address: 192.168.57.1, port: 9094}
    pymtlf-a: {runtime: host-container, address: 192.168.57.1, port: 9092}
    pymtlf-b: {runtime: host-container, address: 192.168.57.1, port: 9091}
    pymtlf-c: {runtime: host-container, address: 192.168.57.1, port: 9292}

consumer:
  machine: core
  requesterNfType: AF
  discovery:
    nrf: nrf
    serviceName: nnwdaf-eventssubscription
    event: UE_COMMUNICATION
  target:
    internalGroupId: "00000001-466-92-01"
    paths: [a, b]
  callback:
    network: sbi
    bindAddress: 192.168.57.32
    advertisedAddress: 192.168.57.32
    port: 9090
    path: /callbacks/nwdaf
  reporting:
    method: PERIODIC
    periodSeconds: 30

security:
  tls: false
  oauth: false

operations:
  clockSkewToleranceMs: 1000
  journalMaxUseMiB: 512
```

這份 definition 的欄位只放跨 component／VM 的共同事實。像 NF retry interval、NWDAF
model policy、PyAnLF window、PyMTLF training rounds 或 log level 等 component-owned 行為，
仍保留在所選 `config/<set>/` 的 native config，不因 renderer 而全部搬進 `testbed.yaml`。
範例中的 `192.168.57.1` 與 ML published ports 是 public default candidate；Host-only
interface 尚未建立時，該 address 不存在是預期狀態。Phase 4 必須在 provider 建立 network
後驗證 Host bind、VM route、port conflict 與 firewall 行為。
PseudoDriver metadata 由 Infrastructure generator 對實際 Parquet 產生；dataset checker
重掃並驗證 traffic profile、hash、bytes、rows、UE IP、timestamp 與訓練／monitor headroom。
舊 go-upf group2 `data.md` 的 row count 差異只保留作歷史 component 記錄，不再決定新情境
active dataset identity。

#### 9.4.1 主 E2E example 與 FL closure smoke

第一版提供兩個明確命名、分開產生 identity 的 scenario。兩者都必須由 Core consumer 經
NRF 找到 A／B、由正常 Nupf Event Exposure 與 PyAnLF prediction／ground-truth matching
產生 WAPE，再由 C 自動建立 A／B training resources；smoke 也不得改用 private observation
injection 或 fabricated degradation callback。

| Setting | `full-core-cat-transition` | `fl-closure-smoke` |
| --- | ---: | ---: |
| 用途 | 公開主 example／正式 business acceptance | 快速驗證 FL control/data path 閉環 |
| UPF／PyAnLF sampling | 30 秒 | 30 秒 |
| historical warm-start | 900 秒／30 observations | 3000 秒／100 observations |
| stable live lead-in | 900 秒 | 360 秒 |
| Path A change boundary | dataset `t=1800s`，CAT1→CAT2 | dataset `t=3360s`，stable→degraded |
| Path B | main acceptance 全程 stable control | 全程 stable control |
| 後續資料 | `t=3600s` 可選 CAT2→CAT3；約 `5429s` 結束 | 600 秒 degraded tail；`t=3960s` 結束 |
| accuracy report period | 90 秒 | 90 秒 |
| 每個 report 的理論 sample capacity／minimum | 3／2 | 3／2 |
| minimum reference reports | 5 | 2 |
| decision window／required hits | 5／3 | 2／1 |
| FL fitting rounds | 2 | 2 |
| local epochs | production 18 | scenario override 2 |
| trigger 後 closure budget | 1800 秒 | 300 秒 |
| performance gate | 關閉但保留完整 final validation evidence | 關閉但保留完整 final validation evidence |
| 預期用途時間 | 第一個 FL／cutover 約 30–40 分鐘內 | 約 10–15 分鐘，依實際 training 而定 |

主 example 的 900 秒 Phase 1 只負責填滿 M1 的 30-step PyAnLF input window，延續舊
testbed warm-start 的原始目的；它不負責在 subscription 成立瞬間同時填滿 PyMTLF training
dataset。Production policy 會在 900 秒 stable live 與最快約 270 秒 degraded decisions 後
才觸發 FL，因此理想 30 秒對齊下，preparation 時約已有：

```text
30 historical + 30 stable live + 9 degraded = 69 observations
```

以目前 `seq_length=30`、`out_seq_len=1`、`validation_ratio=0.1` 與 30-position boundary
purge 計算，69 observations 約形成 8 個 training samples 與 1 個 validation sample。主
example 刻意保留這個少量資料情境；只要實際資料能滿足 training 與 final validation 的
結構需求，就不因樣本少於任意品質門檻而拒絕參與。

Smoke 則允許用 3000 秒 Phase 1 同時準備 inference 與 training history。100 observations
在相同 split contract 下約形成 36 個 training samples 與 4 個 validation samples，使測試
可快速進入兩輪 fitting、FedAvg、final validation、publication、reprovision 與 generation
cutover。較少的 reference／hit policy 與 local epochs 只降低等待時間；它不構成 production
policy、模型改善或長時間 CAT transition 已通過的證據。

Scenario audit 不再只用 `minimumReferenceReports × reportPeriod` 當作 stable lead 下限。
PyAnLF 第一個 report window 可能因 startup phase 或 ground truth 尚未齊全而不可評估，因此
minimum stable lead 另保留一個完整 report period 與一個 sampling interval。Smoke 的最低值
為 `2 × 90 + 90 + 30 = 300` 秒，實際配置 360 秒；bounded trigger 為 dataset live time
570 秒，對應絕對 dataset time 3570 秒，仍落在 3600 秒 preparation window 內。A-only
degraded tail 另包含 300 秒 closure budget，最低 510 秒，實際配置 600 秒。

`minNumSamples` 是 C 在 Model Training data availability requirement 中對「dataset builder
完成後的 training sample count」提出的最低要求，不是 raw packet／UPF notification／
observation 數量、batch size、validation count 或 retrieval 上限。第一版維持
`minNumSamples=1`，不新增先前討論過但沒有業務依據的 32-sample gate。PyTorch DataLoader
不丟棄小於 batch size 的最後一批，因此少量 samples 仍可 training；FedAvg 依 Client 實際
回報的正數 sample count 加權。

目前完整 final validation 另有模型結構造成的 evidence 下限。對 `N` 個 aggregated
observations：

```text
candidate = N - seq_length - out_seq_len + 1
purge     = seq_length + out_seq_len - 1
retained  = candidate - purge
```

使用 30-step input 與 one-step output 時，至少 31 observations 才能形成一個 sliding-window
sample；要同時留下彼此不重疊的至少一個 training 與一個 validation sample，則需 62
observations。62 是目前 split/final-validation contract 的計算結果，不是
`minNumSamples` 或 Infrastructure 人工設定的 training quality threshold。Dataset checker
必須回報每個 Path 的 actual observations、training／validation sample counts；完整 E2E
只在無法形成正數 training 或 required validation evidence 時 fail，不以 32 或 batch size
作 admission gate。

C 現有 `preparation_data_window_seconds=3600` 是 request 的最大回溯時間範圍，不要求資料
一定填滿 3600 秒，也不固定傳回筆數；ADRF 只回傳範圍內實際存在且符合 scope 的 records。
正常 FL 由 C 提供 explicit time window；A／B 的 local fallback window 應對齊 3600 秒，避免
fallback 的 1800 秒在 30 秒 sampling 下最多只有約 60 observations。`max_records_per_job`
的 100,000-record 上限仍只是防止失控下載的 safety ceiling，不是實驗預期資料量。

### 9.5 第一版不管理 experiment history

VM lifecycle 與單次 experiment 分開：三台 VM 可以 `up`／`start` 一次，再連續執行多次
實驗。第一版不定義 run-id、`runs/`、`runtime/`、自動 log collection 或 archive schema。
Log 與 service state 留在各自 execution domain，需要時以 `vagrant ssh`／`journalctl` 或
Docker logs 查看。等 VM/container-aware experiment runner 的 ownership 和需求確定後，再
獨立設計跨多次 experiment 的 identity、collection 與保存方式。

## 10. Host／Guest Scripts 與 Build Lifecycle

### 10.1 Host scripts

`scripts/host/` 在實體機執行，協調 source identity、Vagrant、三台 VM、Docker ML services
以及 guest service／subscription lifecycle；不在 Host 編譯 guest component，第一版也不把
consumer 搬出 Core VM：

- `preflight.sh`：唯讀檢查 provider、RAM、swap、VM／Docker storage free space、submodule、
  testbed files、active config set、必要網段與 port；ML slice 另檢查 Docker、GPU driver、
  NVIDIA Container Toolkit 與容器內 CUDA probe；PseudoDriver 要求已生成且通過完整 audit
  的 content-addressed set，並呼叫 config checker；不安裝 toolkit、修改 host network 或建立 VM；
- `dataset.py`／`dataset-stage.sh`：由 topology/profile 生成或重掃 A/B Parquet，plan 分發
  mapping，apply 只上傳 role-specific artifact 並要求 Guest 原子啟用；
- `config-render.py`：選用地由 default baseline 與 explicit topology definition 產生一組
  `config/generated/<name>/` native configs 和 manifest，不覆寫 committed/default files；
- `config-check.py`：唯讀驗證 default／generated／local config set 的跨 component network、
  identity、placement、A/B mapping 與所選 testbed definition 一致；
- `services-start.sh`：解析 effective `TESTBED`／`CONFIG_DIR`，先 check、stage、activate 完整
  config set 與 Path-specific dataset；各 Guest 先驗證 active config 所產生的 persistent
  Netplan aliases 已由 networkd 套用，只有 drift 時才 reconcile，再依 dependency order 透過
  Vagrant SSH／guest service interface 啟動 Core、Path A 與 Path B 的 database、NF、RAN 與
  Go NWDAF process；不隱含啟動 ML container；
- `services-stop.sh`：停止 experiment stack service/process，不等於 `vagrant halt`，
  更不等於 destructive `vagrant destroy`；
- `services-status.sh`：只彙整 core NF、UPF、gNB／UE、Go NWDAF 與 database health，並回報
  Path `MemAvailable`、UPF memory accounting 與 PseudoDriver replay state；
  VM power state 另外由 `vagrant status` 或 `make vm-status` 回報；
- `ml-start.sh`：驗證所選 config、Host-to-VM reachability、GPU runtime 與五個 endpoint，
  再以 Compose 建置／啟動 `pyanlf-a`、`pymtlf-a`、`pyanlf-b`、`pymtlf-b`、`pymtlf-c`；
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

Renderer／checker 在 Host 執行，因為它們在 VM 建立前決定 effective config set；guest
只能接收已通過 check 的完整 config set，不在 VM 內自行組合或回寫設定。

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
| `preflight` | 呼叫 `scripts/host/preflight.sh` | 是；包含多項 host／provider／network 檢查 |
| `config-check` | 呼叫 `scripts/host/config-check.py` | 是；驗證 selected testbed/config set 的跨檔一致性 |
| `config-render` | 呼叫 `scripts/host/config-render.py` | 是；選用地產生 gitignored complete config set |
| `vm-up` | 直接呼叫 multi-machine `vagrant up` | 否；先避免沒有邏輯的一行 wrapper |
| `vm-status` | 直接呼叫 `vagrant status` | 否 |
| `vm-halt` | 直接呼叫 `vagrant halt` | 否 |
| `services-start` | 呼叫 `scripts/host/services-start.sh` | 是；包含 config check／activation、跨 VM stage、readiness 與 rollback |
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

1. 依 `CONFIG_DIR` explicit argument、`testbed.local.yaml.configDir`、`config/default/` 的
   優先序解析完整 config set，並解析同一次命令使用的 `TESTBED` definition；
2. 執行 `config-check`，計算 config manifest/hash，並比對 Vagrant base interfaces 與
   role-specific process alias 清單；若 base interface／Vagrant network 不相容，停止並要求
   明確 reload／reprovision，不在 service start 時修改 VM 網卡；
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
[legacy-vm-inventory-and-removal-proposal-2026-08-06.md](../reports/legacy-vm-inventory-and-removal-proposal-2026-08-06.md)。
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
| WebConsole | `webconsole-status`／`webconsole-stop` | 顯示 unit、HTTP frontend、登入 API 與 MongoDB connectivity，或獨立停止 WebConsole。 |

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

## 16. 本機實作紀錄

### 16.1 2026-08-06 Infrastructure baseline

已在 `/home/chingje/testbed/5G_NWDAF_Infrastructure` 建立獨立 local git repository，
branch 為 `main`，目前未設定 remote。實作未修改或清除舊 `5G_Infrastructure` source
working tree，也未建立任何新 VM。

主要 commits：

| Commit | 內容 |
| --- | --- |
| `36da7ee` | repository ownership、目錄與 command surface |
| `4fc1f8e` | Core／Path A／Path B、dual TAI topology 與 Vagrant definition |
| `bb9a7c0` | 16 個 component submodule gitlinks、branch hints、license inventory |
| `49f0598` | default native config、typed renderer 與 cross-component checker |
| `3b2c87b` | guest-local build、hashed config activation 與 disabled systemd units |
| `3dc3e07` | host preflight、service lifecycle、status、observe 與 journald follow |
| `a4d4a58` | NRF discovery、雙 NWDAF subscription、callback、rollback 與 exact DELETE |
| `47d3079` | Vagrant disk budgets 與 reboot-safe guest address aliases |
| `b67cd40` | guest allocation 外保留 4 GiB physical-host RAM safety margin |
| `ac2b9db` | quick start、operation boundaries、PseudoDriver 與 non-goals 文件 |

已完成的非 privileged checks：

- 所有 committed YAML 可解析；
- default 與 renderer output 都通過同一套 config checker；
- 16 個 `components.lock.yaml` revisions 與 parent gitlinks 一致，submodule clean；
- PyAnLF-A/B 與 PyMTLF-A/B/C native config 通過各自 `load_settings()`；
- consumer 以 synthetic NRF profiles 驗證 TAI A/B distinct-NF selection；
- shell／Python syntax 與 Vagrant 2.4.3 `validate --ignore-provider` 通過。

建立此 baseline 時，UPF 固定在 remote 可取得且包含 PseudoDriver 的
`test-EES-with-pseudodriver@9a4d95c`；`gtp5g` 因該 UPF 的版本檢查固定
`v0.9.16@8d723c2`。先前 full-core 紀錄中的 go-upf `c69051b` 是無法從 remote
取得的 local commit，因此此時只可稱為可重建的新基線。UPF 後續已前進到以
`9a4d95c` 為 parent 的 Release 18 contract 修正版 `234bae0`，runtime 結果見 16.18；
仍不可宣稱已等價重現舊 full FL E2E。

2026-08-06 preflight 當下 storage 約 176 GiB free，通過 120 GiB threshold；available
RAM 約 20,026 MiB，低於 16,384 MiB guests 加 4,096 MiB host reserve，且 swap 幾乎用盡，
因此正確阻擋 `vm-up`。在釋放 host memory、選定並驗證 provider 前，不進入新 VM 建立。

### 16.2 2026-08-09 GPU capability 與 deployment 調整

Component-owned GPU 支援已分別完成、驗證並推送，repositories 仍保持獨立 commit：

| Repository | Commit | 內容 | 驗證 |
| --- | --- | --- | --- |
| PyAnLF | `9e64417fe053e03c2a616abea6f284df8acd1b38` | configurable CUDA inference、device validation 與 CUDA test coverage | Ruff；295 passed、1 個既有 live-artifact test skipped；RTX 3080/R535 smoke |
| PyMTLF | `e9c5b08725dc06835485b29ff6c264340f9805f9` | configurable CUDA training、CPU-safe FedAvg／serialization 與 CUDA test coverage | Ruff；205 passed；RTX 3080/R535 smoke |

兩者使用 Python 3.12、PyTorch 2.5.1；Linux x86_64 的 uv source 指向 PyTorch CUDA 12.1
index，`nvidia-nvjitlink-cu12==12.1.105` 只作相容 constraint。這不要求 Host 安裝完整 CUDA
toolkit，也不要求先把 R535 driver 升到 CUDA 12.8/13；Docker 實際存取 GPU 仍以 NVIDIA
Container Toolkit 為 activation gate。

`5G_NWDAF_Infrastructure` 已以 `5924a67` 將 `ML/PyAnLF`、`ML/PyMTLF` gitlinks 前進到
上述 commits，並同步更新 `components.lock.yaml`。Host endpoint definition 已在後續 topology
freeze 補上；Compose definition、ML lifecycle 與 guest ML removal 仍尚未實作。不得把
source pinning 或 component GPU smoke 誤認為混合 testbed 已完成。

本次架構決策將五個 ML backend 移至 Host containers，三台 VM 僅保留需要 network/kernel
boundary 的 5GC、UPF、UERANSIM、Go NWDAF、ADRF 與 MongoDB。Consumer containerization、
application log 改造與 per-run archive 均延後，不混入第一個 ML deployment slice。

### 16.3 2026-08-09 PseudoDriver dataset RAM audit

> 這一節保存舊 pinned go-upf `pre_data` 的歷史量測。新 full-core 情境的 active
> dataset ownership、identity 與長度已由 16.16 的 Infrastructure generator 取代。

Path RAM 重新納入 PseudoDriver warm-start。Pinned group1/group2 Parquet 分別為 21,830,425／
44,050,349 bytes；以目前 `parquet-go` streaming implementation、30 秒 window 與一個 AnyUE
subscription context 直接掃描，rows 分別為 2,720,063／5,759,921，test process peak RSS
約 22.9／22.7 MiB。Probe 在完成後已移除，未留下 component source change。

此結果只證明目前 driver 不把所有 raw rows 常駐 RAM，不代表完整 Path VM 只需增加約
23 MiB；full UPF、Parquet page cache、gtp5g、UERANSIM、Go NWDAF、journald、多 subscription
context 與 Go GC 必須一起量測。因此 Path 仍先維持 3 GiB target，但將 512 MiB
pre-replay `MemAvailable` 設為 gate；未達或出現 reclaim/OOM 時，以 512 MiB step 上調。

另發現 `pre_data/data.md` 的 group2 row count 與 pinned file 直接掃描不一致。Infrastructure
必須以 SHA-256、bytes 與 machine-verified rows 鎖定 dataset identity，文件差異另由 go-upf
repository 修正，不能以過時描述估算 RAM。

### 16.4 2026-08-09 Hybrid topology/config freeze

Infrastructure commit：`9e24cc5`。對應 Host 清點見
[hybrid-host-readiness-inventory-2026-08-09.md](../reports/5g-nwdaf-infrastructure/hybrid-host-readiness-inventory-2026-08-09.md)。

Infrastructure definition 已改為 Core 4096 MiB、Path A/B 各 3072 MiB，三台 VM 都使用
40 GiB dynamic primary logical capacity。五個 PyAnLF／PyMTLF role 從 VM placement 移到
Host containers，guest service start 不再啟動 Python units；舊 guest provisioning/unit
仍暫留作 rollback material，待 container/GPU gate 通過後才另行移除。

Host ML public candidate 使用 `192.168.57.1` 與五個不重疊的 published ports。Native config
在 container 內 bind `0.0.0.0`，Go NWDAF backend、callback 與 artifact URL 則只使用
advertised endpoint。Renderer 可從任一 explicit topology 重建同樣關係；checker 會驗證
placement、port/device mapping、native endpoints 與兩份 PseudoDriver file identity。

Preflight 已加入 Docker access、VirtualBox initialization、Host ML bind address／port、10 GiB
guest allocation 加 6 GiB Host reserve 與 swap warning policy。這一批只完成 definition、
static/non-mutating checks 和文件；沒有建立 VM、container、interface 或 route，也沒有安裝
NVIDIA Container Toolkit。

### 16.5 2026-08-09 Host ML image 與 CPU smoke

Infrastructure 新增 pinned Python 3.12 slim base、共用 PyTorch 2.5.1＋CUDA 12.1 runtime、
PyAnLF／PyMTLF 兩個 image targets，以及五個獨立 Compose services。Production definition
只讓 PyMTLF-A/B request GPU；CPU smoke override 同時移除 GPU request 並產生 disposable config，
將 A/B training device 改成 CPU。每個 service 使用非 root user、read-only root filesystem、
read-only config bind、獨立 named volume、health check、CPU/RAM/PID limit 與 bounded local logs。

Static checker 同時解析 production 與 CPU override，驗證 service/port/build target/source
revision/security/config/data volume/GPU request。PyMTLF 的四個可寫目錄——artifact、model
state、publication journal、FL workspace——全部固定在各自 named volume root；本次 smoke
也因此找出並修正原先遺漏的 publication 絕對路徑。

Bounded CPU smoke 實際建立五個 containers，全部通過 application readiness。五者都載入
`torch 2.5.1+cu121` 且在無 GPU request 下回報 CUDA unavailable；PyMTLF-A/B effective
training device 都是 `cpu`。空載即時 RSS 約為 PyAnLF 230 MiB/個、PyMTLF 283 MiB/個，合計
約 1.28 GiB。這不是 training/dataset/GPU peak，不能據此縮小 full-run budget。

兩個 images 的 virtual size 各約 5.42 GB，前六層（含 Python、uv、PyTorch/CUDA runtime）
相同；Docker 當下把 5.421 GB 報為 shared size。Smoke 結束後自身 container、network、volume
與 generated config 已移除，兩個 images 保留供下次使用；未對共用 Docker daemon 執行
global prune。完整證據見
[host-ml-container-cpu-smoke-2026-08-09.md](../reports/5g-nwdaf-infrastructure/host-ml-container-cpu-smoke-2026-08-09.md)。

這次沒有安裝 NVIDIA Container Toolkit、沒有讓 container 存取 GPU、沒有建立 VM 或 Host
network，也沒有驗證 VM-to-Host published endpoints。下一個安全工作是實作日常
`ml-start`／`ml-status`／`ml-stop` 及 observe/log integration；GPU 與 network mutation 仍需
另行授權。

### 16.6 2026-08-09 Host ML lifecycle 與觀測

Infrastructure 已實作正式 `ml-start`／`ml-status`／`ml-stop`。Production project 固定為
`5g-nwdaf-infrastructure`；所有 status、stop 與 Docker log selection 都以 exact Compose
project/service label 定位，不依賴 container IP，也不掃描或操作其他 project。

`ml-start` 依序驗證 native config 與 resolved Compose、確認 Host bind address、RAM reserve、
Docker free-space 與 swap policy、記錄完整 config hash、各 build 一次 PyAnLF/PyMTLF image，
並在 production mode 以一次性 container 執行實際 CUDA visibility probe。CUDA 不可見時不
允許 silent CPU fallback；若 service startup 失敗，只停止本 project containers，保留 image
與 named volume。`ml-stop` 同樣只停止該 project 的 running containers，不執行 `down` 或
prune。

`ml-status` 可在 running 或 stopped 狀態顯示五個 service 的 container state、application
health、native config effective device、實際 CUDA visibility、即時 memory、image ID、
component source revision、config-set 與 config hash。`observe` 已加入這張狀態表；`logs.sh`
支援 VM、ML 或 combined source、service glob、since、tail、follow/non-follow，ML log 以
`[ml:<service>]` prefix 呈現，離開 follower 不停止 process。

Disposable CPU lifecycle smoke 使用獨立 project 與 loopback config，驗證五個 services
healthy、status identity、ML log filter、stop 後五個 containers 與五個 volumes 仍保留，最後
才對自己的 smoke project 執行 `down --volumes`。空載 RSS 仍約 1.28 GiB；smoke 沒有安裝或
使用 NVIDIA toolkit，也沒有建立 VM 或 Host network。完整紀錄見
[host-ml-lifecycle-smoke-2026-08-09.md](../reports/5g-nwdaf-infrastructure/host-ml-lifecycle-smoke-2026-08-09.md)。

PyAnLF 啟動同時回報 callback ingestion default 的高理論 memory bound（8192 entries ×
4 MiB request ceiling）。Queue 不會在 startup 預先配置，且 container 具有 768 MiB hard
limit，但正式 callback burst 仍可能造成 container-local OOM 或 drop-oldest。這次不在缺乏
traffic peak evidence 時改變 queue semantics；GPU/full-stack smoke 前須量測實際 payload、
queue depth、drop counter 與 peak RSS，再決定 capacity／request limit 或提高 container RAM。

### 16.7 2026-08-09 Shared Docker GPU activation 初始決策（已由 16.9 修正）

唯讀 Host audit 確認 RTX 3080／R535 driver 正常，Docker 27.4.1、Compose 2.32.1 可用，但
尚未安裝 `nvidia-ctk`／NVIDIA container runtime，也沒有 CDI spec。共用 Docker 當時有八個
running containers 且 `live-restore=false`；其中部分使用 `restart: no`，因此不能把 daemon
restart 視為不影響他人的一般 prerequisite step。

第一版改採既有 rootful Docker 的 native CDI support。另行取得 Host package mutation 授權後，
只安裝 CDI 所需的 `nvidia-container-toolkit-base`，確認自動或手動產生的
`/var/run/cdi/nvidia.yaml`，再依序執行：

1. `nvidia-ctk cdi list` 應列出 RTX 3080 device；
2. 不改 `/etc/docker/daemon.json`，以 disposable container 的 CDI `--device` 執行
   `nvidia-smi`；
3. 將 production Compose 的 PyMTLF-A/B 從 `gpus:` 改成
   `devices: ["nvidia.com/gpu=all"]`，同步更新 static checker、CPU override 與 lifecycle
   CUDA probe；
4. 驗證現有八個 containers 的 ID、start time 與 state 未因安裝／probe 改變；
5. 依序量測 PyMTLF-A、PyMTLF-B 與 A/B 同時 training 的 peak VRAM；外部 GPU workload
   可以共存，但必須記錄其 baseline usage 與避免把 slowdown 誤判為本 testbed failure。

任何 CDI inventory、Docker device resolution 或 container CUDA probe 失敗，都應 fail closed，
不得自動改採 legacy runtime configuration 或 restart shared daemon。專用 rootless Docker 只作
第二順位 fallback；audit 顯示 rootless scripts、user namespace 與 subordinate IDs 已具備，
但仍缺 `uidmap`，且 Host 的 cgroup version 是 v1。即使補齊 package，仍不能依賴 rootless
daemon 落實目前 Compose 的 CPU／RAM limits，並會重複 image store、增加固定 Host SBI port
與 VM reachability 的網路驗證，因此不作預設部署路徑。

### 16.8 2026-08-09 Native CDI deployment definition（已由 16.9 取代）

Infrastructure production Compose 已將 PyMTLF-A/B 的 legacy `gpus:` request 改為 CDI
`devices: ["nvidia.com/gpu=all"]`；PyAnLF-A/B 與 PyMTLF-C 不要求 GPU。CPU smoke override
改為清除 `devices`，resolved Compose 已證明五個 CPU services 都不殘留 CDI request。Static
checker 同時驗證 exact CDI source／target／permissions，並禁止任何 service 保留 legacy
`gpus:`。

Production `ml-start` 在 image build 前先要求 `nvidia-ctk cdi list` 包含
`nvidia.com/gpu=all`，避免缺 Host prerequisite 時執行無用 build；image ready 後再以
`docker run --device nvidia.com/gpu=all` 執行 PyTorch CUDA visibility probe。任一 gate 失敗
都發生在 production services 建立前，且 script 不修改 daemon config。`ml-status` 新增 CDI
欄位，將 application effective device、實際 container CDI mapping 與 CUDA visibility 分開
呈現。

Static baseline／CPU Compose checks、Python/Bash syntax 與 Compose render 均通過。Disposable
CPU lifecycle regression 再次讓五個 services 全部 healthy，running／stopped status 都顯示
`CDI=none`，project-scoped stop/retention 與最終 `down --volumes` cleanup 通過；其他 Docker
projects 未納入操作範圍。這是 Host activation 前的原始定義；Docker 27.4.1 的實機結果與
後續採用方式記錄於 16.9。

### 16.9 2026-08-09 NVIDIA runtime CDI activation

Host 已安裝 `nvidia-container-toolkit-base` 1.19.1，啟用 `nvidia-cdi-refresh` 並產生
`/var/run/cdi/nvidia.yaml`。Inventory 列出 GPU index、UUID 和 `nvidia.com/gpu=all`。Docker
27.4.1 雖接受 `features.cdi=true`，但 reload 後 native CDI manager 沒有初始化；使用
`--device nvidia.com/gpu=all` 的 probe 因 `cdi` driver 無法選取而 fail closed。

為避免在 `live-restore=false` 的共用 daemon 上 restart，經使用者同意改用 NVIDIA runtime
CDI mode。`nvidia-ctk runtime configure --runtime=docker` 只註冊 named `nvidia` runtime，保留
`runc` 為 default；`systemctl reload docker` 後 daemon PID、start time、restart count 和八個
既有 container 的 ID／state 都未改變。Disposable Ubuntu probe 使用：

```text
--runtime=nvidia
NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=all
NVIDIA_DRIVER_CAPABILITIES=compute,utility
```

probe 成功回報 RTX 3080、UUID `GPU-05edb56c-f554-413d-4cf3-b92e8e85a42b`、driver
535.183.01 與 10240 MiB VRAM，且 container 已自動移除。Production Compose 因此讓
PyMTLF-A/B 使用相同 runtime 與 CDI selector；其他三個 ML services 維持預設 runtime。
CPU smoke override 明確使用 `runc` 和 `NVIDIA_VISIBLE_DEVICES=void`。`ml-start` 會先驗證
named runtime 與 CDI inventory，再用 PyMTLF image 做 PyTorch CUDA probe；一般 automation
不設定 daemon，也不 reload／restart Docker。

同一 runtime path 也以 integration repo 的 PyMTLF image 驗證通過：PyTorch
`2.5.1+cu121` 回報 CUDA available、CUDA runtime 12.1 與 RTX 3080，沒有 silent CPU
fallback。更新後的 CPU lifecycle regression 讓五個 services 全部 healthy，PyMTLF-A/B
明確顯示 `RUNTIME=runc`、`CDI=void`、CUDA unavailable，stop／retention／cleanup 亦通過。

這仍不等於 training capacity 已確認。為避免先建立假的 ADRF／dataset 路徑，GPU training
gate 延後併入 full-core validation：先執行一次有 timeout 的 A/B concurrent training，記錄
peak VRAM、OOM 與外部 GPU workload 影響；只有該最壞情境失敗時才拆成 A、B 單獨診斷。

### 16.10 2026-08-09 Guest baseline 改為 Ubuntu 22.04

建立 VM 前重新對照舊 `5G_Infrastructure`：舊 5GC、gNB 與多數 network VM 使用
`ubuntu/focal64` 20.04，較新的 `UPF-EES`／`UPF-EES2` 使用 `ubuntu/jammy64` 22.04；舊環境
並非全數 22.04。新版不延續混合 release，而是依使用者決策將 Core、Path A、Path B 統一為
Ubuntu 22.04。

Host 已快取 `ubuntu/jammy64` `20241002.0.0`，尚未快取原定的 `bento/ubuntu-24.04`。改用
Jammy 可避免額外 box download，並貼近曾實際使用的 UPF Event Exposure 環境。Canonical
`testbed.yaml` 將 box 精確固定為該版本，Vagrant 關閉 automatic box update check；Vagrant
fallback 和 operations 說明也同步改為 22.04。Core MongoDB 8.0 apt source 由 `noble` 改為
`jammy`。後續 skeleton smoke 必須記錄實際 box version、guest kernel、gtp5g build 與
UERANSIM build 結果，不能只因 VM 能 boot 就宣稱 guest 相容性完成。

### 16.11 2026-08-09 Core VM skeleton smoke

VirtualBox Host 原有 `/etc/vbox/networks.conf` 只允許 `192.168.33.0/24` 與
`192.168.56.0/24`。舊 testbed repository／指引沒有要求此檔案，且舊有效 VM networking
主要使用 bridged `public_network`；因此 `192.168.33.0/24` 視為未知 site range 保留，另由
使用者將 testbed allowlist 擴為 `192.168.56.0/21`。這只允許 VirtualBox 建立 56–63 的
host-only adapter，實際 topology 仍是八個獨立 `/24`。Host preflight 新增 generic interface
coverage check，避免 VM import 後才因 allowlist 失敗。

第一次 `core --no-provision` boot 已證明 Jammy box、NAT、SSH 與四個 host-only adapter 可用，
但同時發現 Vagrant rsync 把 Host `.venv` 傳入 guest，且預設 `/vagrant` vboxsf 直接暴露整個
Host working tree。同步在約 3.6 GiB 時中止；由於 dynamic VMDK 已膨脹到約 5.5 GiB 且不會因
guest 刪檔自動縮小，經使用者明確同意後只銷毀這台尚未 provision、無資料、無 snapshot 的
`core`，再以 cached box 乾淨重建。

修正後 guest rsync 排除整個 `ML/`、virtualenv 與常見 cache，並停用預設 `/vagrant` share。
既然 Host NVIDIA runtime、PyMTLF CUDA probe 與 CPU lifecycle 已通過，guest provisioning 和
service dispatch 也正式移除 PyAnLF／PyMTLF staging、uv environment 與舊 service cases；ML
source/runtime 只由 Host container 負責。

乾淨 `core` skeleton 的實測結果：

- Ubuntu 22.04.5、kernel 5.15.0-171、4 GiB RAM、4 vCPU；
- NAT 加 `192.168.56.10`、`192.168.57.2`、`192.168.58.2`、`192.168.61.2`；
- Host 對四個 IP ping、guest 對 Host `192.168.57.1` ping 與 Vagrant SSH 均通過；
- guest source snapshot 106 MiB，`ML/` absent，`/vagrant` absent；
- 40 GiB dynamic VMDK 實佔約 1.6 GiB，guest root used 約 1.7 GiB。

此 smoke 使用 `--no-provision`，尚未驗證 apt packages、MongoDB、Go/NF build、config
activation 或 systemd services。Host VirtualBox 6.1 與 guest additions 6.0 版本不完全一致，
但受控 source 使用 rsync 且 `/vagrant` 已停用，因此目前不把 vboxsf compatibility 當成
prerequisite；後續不得重新引入 shared-folder dependency。

### 16.12 2026-08-09 Path VM skeleton 與三 VM network smoke

在 `core` definition commit 後，以相同 pinned Jammy box 和 `--no-provision` 依序建立
`path-a`、`path-b`。兩台均套用 3072 MiB RAM、3 vCPU、40 GiB dynamic VMDK，source
snapshot 都是 106 MiB、沒有 `ML/` 或 `/vagrant` share，VMDK 實佔各約 1.6 GiB。

Path A 的 management／SBI／N2／N3-A／N4／N6-A 位址，Path B 的
management／SBI／N2／N3-B／N4／N6-B 位址均存在。Host 對所有 VM interface、每個 Path
對所屬 Host `.1`，以及 Core／Path A／Path B 在 management、SBI、N2、N4 四個共享 `/24`
的雙向 ping 都通過。Host 最終具有 `vboxnet0`–`vboxnet7`，分別承載
`192.168.56.0/24`–`192.168.63.0/24`，沒有把八個 segment 合併成一個 `/21`。

為避免把 production ML readiness 與未 provision 的 MongoDB／ADRF 混在一起，本輪沒有
啟動 production Compose。Host 在 `192.168.57.1:9091` 開啟短暫 HTTP listener，三台 VM
分別從 `192.168.57.2/.3/.4` 取得 HTTP 200，證明實際 VM-to-Host ML TCP path；listener 隨即
停止且 port 無殘留。

三台同時 running 時 Host 約有 24 GiB `MemAvailable`、175 GiB workspace free，八個既有
Docker containers 狀態未受影響。完成 bounded smoke 後三台 VM 已 graceful halt，保留 disks
供後續分階段 provisioning。這只完成 VM/network skeleton，不代表 N6 NAT、gtp5g、UERANSIM、
NF build 或 service E2E 已通過。

### 16.13 2026-08-09 Process alias 改由 topology 產生

先前 `network-setup.sh` 直接內建 Core、Path A、Path B 的 process alias IP。這能在尚未
stage config 時於開機建立位址，但使 shell script 和 `testbed.yaml` 成為兩份 network truth；
其中 Path `.42/.43`、`.52/.53` 與 Core `.31` 也已是 Guest ML 搬到 Host containers 後不再
具有 owner 的舊 alias。

調整後 `testbed.yaml` 仍是唯一人工維護的 topology source。`config-render.py` 會從 VM
interface anchors、Core service endpoints、Path gNB／UPF endpoints、NWDAF SBI 與 consumer
callback 產生 `network/core.yaml`、`network/path-a.yaml`、`network/path-b.yaml`。Public
`config/default` 提交同 schema 的完整 baseline；generated／local set 也必須包含三份檔案並
通過 `config-check.py` 的 exact derivation、subnet、address uniqueness 與 native config
一致性檢查。

VM boot 仍只由 Vagrant 建立 base interfaces。Config activation 切換 active symlink 後先
restart `5g-nwdaf-network.service`；Guest script 依 machine identity 讀取自己的 YAML，以
anchor 找到實際介面，idempotently 套用 aliases，並移除上一組由它管理但新組合不再宣告的
位址。失敗時恢復前一組 active config，不啟動 NF process。這保留 VM power 與 service
lifecycle 分離，同時允許不改 Vagrant base network 的 process address 組合隨完整 config set
切換。

本節記錄的是當時已完成的 imperative implementation；後續實機 cold-boot audit 證明
boot-time `ip address add` 會被 Vagrant 最後一次 Netplan apply 清除。新的 persistent
ownership 與遷移方案以 16.23 為準，topology-derived YAML 與 anchor resolution 決策則保留。

Default config 與一次 disposable rendered set 已通過相同 checker，三份產生結果與 committed
baseline 完全一致；Python compile 與 Guest／Host shell syntax 亦通過。因三台 VM 尚未
provision，本輪沒有宣稱實際 guest `ip address` 套用或 rollback 已完成 runtime 驗證。

### 16.14 2026-08-09 Guest service lifecycle 與 six-UE registration

三台完成 provisioning 的 VM 已共同執行 config activation 與完整 guest service lifecycle。
實機整合依序找出並修正三個 bring-up blocker：config identity 不應包含 Host／Guest absolute
root、SMF 必須明確取得 active `uerouting.yaml`、pinned UERANSIM 的六份 UE config 必須包含
`uacAic`／`uacAcc`。啟動 rollback 也改為 success path 才解除的 `EXIT` trap，並由 UE schema
failure 證明 partial stack 會被完整停止。

UERANSIM 不會建立 operator subscription data。依 Phase 3 原規劃，只從
`nwdaf-resources@d2634b84e8790a6b696e5b21ec1a0f660b683948` 選取 six-UE／single-group fixture
語意，將 scoped provisioning 收進本 repository；沒有加入整個 repository 或 runtime
dependency。Core 已有的 `mongosh` 負責 48 份 subscriber documents 與一份 Internal Group 的
validate／plan／idempotent apply／show／scoped clear，`config-check` 先驗證 PLMN、SUPI、group、
K/OPc、AMF、S-NSSAI 與 DNN。subscriber data 跨 service stop 保留，`services-start` 在啟動 UE
前自動 apply。

完整 stop/start 回歸後，23 個 guest units 全部 active；六個 UE 不需人工 restart 即完成
registration 與 PSI 1 PDU Session。Path A 取得 `10.60.0.1`–`10.60.0.3/16`，Path B 取得
`10.61.0.1`–`10.61.0.3/16`，符合 two-TAI pool boundary。完整證據與剩餘限制見
[guest-services-and-ue-registration-smoke-2026-08-09.md](../reports/5g-nwdaf-infrastructure/guest-services-and-ue-registration-smoke-2026-08-09.md)。

這完成 Phase 6 的 process、registration 與 PDU Session bring-up 子集，不宣稱 N6 traffic、
PseudoDriver replay／Event Exposure、Host ML containers 或 subscription E2E 已通過。

### 16.15 2026-08-09 Host ML 與 Guest Stack 整合 Smoke

三台 VM 的完整 guest stack 與五個 production Host ML containers 已同時啟動。Production
project 使用 config identity
`a73ef32bb621b3a20efa836f12183a95bdf0e7bd34cfb8565f8884626f5a99c0`；PyMTLF-A/B 透過
NVIDIA runtime CDI mode 使用 `cuda:0` 並回報 CUDA available，PyAnLF-A/B 與 PyMTLF-C
維持 CPU。五個 services 全部 healthy，空載／週期 sync 狀態合計 RSS 約 1.38 GiB；同時運行
三台 VM 時 Host 仍有約 22 GiB `MemAvailable`。

Core、Path A、Path B 分別從 `192.168.57.2/.3/.4` 對所屬五個 Host readiness endpoint
取得 HTTP 200。三個 NWDAF 向 NRF 註冊成功並持續對 ML backend 執行
`POST /internal/v1/sync`；PyAnLF-A/B 對各自 NWDAF 的 SMF association／training descriptor
sync 也持續取得 204，證明 hybrid boundary 的雙向 transport 與 contract 可用。

本輪沒有啟動 consumer subscription、N6 traffic、PseudoDriver replay、analytics callback、
training 或 FL。PyAnLF 仍顯示 8192 × 4 MiB callback queue 理論上限警告；它沒有預先配置
32 GiB，但正式 callback burst 必須量測 queue/drop/peak RSS。GPU 當時約 432 MiB 已使用且主要
來自另一使用者 process，本 testbed 未 training，因此這次也不是雙 client VRAM capacity
證明。

驗證後依序停止 ML、guest services 與 VM；三台皆 poweroff，running Docker containers 回到
原有八個共用 containers。另將 subscriber fixture identity 改為以 repository-relative
filenames 計算，確保同內容在不同 clone root 仍得到
`d30803f9c5904ae86bb222484170089cc4cf60ee3fe3f29e43c6487918113167`。完整證據見
[host-ml-guest-stack-integration-smoke-2026-08-09.md](../reports/5g-nwdaf-infrastructure/host-ml-guest-stack-integration-smoke-2026-08-09.md)。

### 16.16 2026-08-09 Generated PseudoDriver Dataset Tooling

新版情境不再直接使用 go-upf submodule 的 `pre_data/group1`、`group2`，也不把
`nwdaf-resources` 納入 runtime。Infrastructure 現在只提交兩份 traffic profile、Go
generator/auditor、Host lifecycle 與 Guest activation；Parquet、resolved spec、manifest
和 generator binary 都寫入 ignored `.generated/`。

UE IP 不寫死在 profile。Generator 從 `testbed.yaml` 的 Path A/B `uePool` 與 UE 數量推導
`10.60.0.1`–`.3`、`10.61.0.1`–`.3`，並把 seed model window、PyMTLF validation ratio、
PyAnLF/UPF 30 秒 period、Model Monitor 90 秒 period、reference/hit policy 一起納入 dataset
set identity。預設每 Path 4,500 秒、27,000 rows：前 3,000 秒提供 100 個歷史觀測，接著
600 秒穩定 live lead-in；Path A 再提供 900 秒 degraded tail，Path B 同期維持 stable。

`dataset-check` 會重掃 Parquet，驗證 schema、SHA-256、bytes、rows、UE IP set、timestamp
range、每 UE row count 與 `file.json` boundary。`dataset-stage` 只把 role-specific artifact
傳到所屬 Path；Guest 再驗 set ID、role、hash、bytes 與 breaking time，最後原子切換
`/var/lib/5g-nwdaf-infrastructure/datasets/active`。`services-start` 在任何 process 啟動前完成
staging，UPF 在 active dataset 缺失時拒絕啟動。

靜態驗證已完成兩次獨立 deterministic generation、tampered-Parquet negative test、default
與 rendered config check、shell/Python/Go syntax 及 staging plan；尚未啟動 VM 或執行真正
PseudoDriver subscription/replay。完整結果見
[Generated PseudoDriver Dataset Tooling](../reports/5g-nwdaf-infrastructure/generated-pseudodriver-dataset-tooling-2026-08-09.md)。

### 16.17 2026-08-09 PseudoDriver Dataset Guest Staging Smoke

以既有 VirtualBox `core`、`path-a`、`path-b` 執行短時間 staging smoke。Preflight 在
`VAGRANT_DEFAULT_PROVIDER=virtualbox` 下為 0 failures；free swap 約 1 MiB 是唯一 warning，
Host `MemAvailable` 約 29 GiB 且 workspace／Docker filesystem 約 165 GiB free。三台 VM
啟動後都沒有 active `5g-nwdaf@*.service`，consumer 也維持 inactive。

`make dataset-stage` 將 set
`3cc771b6d283ceee5927e3986dbe1920039e72ce69575389c10556a82a8be4a2` 的 role-specific
archive 分別上傳 Path A/B。Guest 驗證後，兩邊 `active` 都原子指向自己的
`datasets/sets/<set-id>`：Path A manifest 為 `10.60.0.1`–`.3`、SHA-256
`4e221ac0b2197be0dad4bbfb20b34f2849d67f8c79d3115169eeb95c613d29da`；Path B 為
`10.61.0.1`–`.3`、SHA-256
`76a887d5b37dbce0710250785756884f9bcf080695be4b0a06bb4f329fa6e0e9`。兩者均為
27,000 rows／841,634 bytes，Guest 實際 hash 與 manifest 相同，owner 為
`5g-nwdaf:5g-nwdaf`。

另把 Path A archive 暫時交給 Path B activation；Guest 以
`dataset identity does not match target machine` 拒絕，且 Path B 原 active symlink 未改變。
staging 後 Path A/B `MemAvailable` 約 2640／2627 MiB，dataset directory 各約 1 MiB且只有
一個 set。驗證後三台 VM 已 graceful halt 並回到 poweroff。

這只完成 artifact transport、Guest verification、role isolation 與 reversible activation，
沒有啟動 UPF、建立 PDU Session／subscription、讀取 Parquet 或量測 replay peak。下一個 gate
是短時間啟動 guest services 與 subscription，核對實際 PDU IP、PseudoDriver matched rows、
Event Exposure callback 及 Path RAM peak。完整證據見
[PseudoDriver Dataset Guest Staging Smoke](../reports/5g-nwdaf-infrastructure/pseudodriver-dataset-guest-staging-smoke-2026-08-09.md)。

### 16.18 2026-08-09 Nupf Contract 與 PseudoDriver Runtime Smoke

Infrastructure commit `e119dc9` 將 `NFs/upf` 從
`test-EES-with-pseudodriver@9a4d95c` 前進到
`fix/r18-nupf-event-exposure-contract@234bae0`。新 commit 以 `9a4d95c` 為直接 parent，
保留 shared Parquet PseudoDriver，並把 `ueIpAddress`、`repPeriod` 對齊 Release 18 Nupf
Event Exposure contract。`go test ./internal/ees` 與 `go build ./cmd` 通過；完整 repository
test 仍會碰到既有 root `sim_my_test.go` compile error 與需 netlink／gtp5g privilege 的測試，
不可誤報為全套 unit tests green。

兩台 Path VM 都從該 revision 重建 UPF，binary SHA-256 同為
`931ea003d6055d0f1f20d26f19cdf7f454bf12b4139f8ae6bcacfd72e27ac0e7`。23 個 Guest
units 全部 active，六個 PDU Session 分別取得 Path A `10.60.0.1`–`.3` 與 Path B
`10.61.0.1`–`.3`。Consumer 經 NRF 找到兩個不同 NWDAF 並建立兩筆 subscription；SMF
觀察到 12 次 Nsmf Event Exposure subscription POST 全部回 `201`，沒有先前的 `502`。
兩個 UPF 各維持三筆 subscription，對各自三個 PDU IP 完成 18,006-row historical
warm-start，接著開始 pacing 8,994-row Phase 2。PyAnLF-A/B 都接受至少兩輪、每輪三筆的
UPF callback 並回 `204`；consumer 最終收到 A/B 各一筆 analytics notification。

五個 Host ML containers 同時 healthy，runtime RSS 合計約 1.33 GiB；完整 stack 運行時
Host 仍有約 23.7 GiB `MemAvailable`，workspace 約 164 GiB free。UPF service cgroup 的
單次 current snapshot 約為 Path A 8.1 MiB、Path B 9.5 MiB，但 Host 使用 cgroup v1，沒有
取得可採信的 `memory.peak`，所以本輪不關閉 Path replay peak gate。驗證後 exact DELETE
兩筆 subscription，依序停止 consumer、五個 ML containers、23 個 Guest services 與三台
VM；三台均為 `poweroff`，其他八個共用 containers 持續運行。

本輪同時找出並修正 default UDM Internal Group range、consumer state CLI 執行身份，以及
已移到 Host containers 後仍殘留在 Guest stop list 的 ML units。這完成短時間
subscription／replay／analytics callback 閉環，不包含 3,000 秒 degradation、accuracy
degradation、Model Monitor coordination、A/B training、FedAvg、ADRF publication、reprovision
或 generation cutover。下一個 gate 應以 bounded timeout 驗證這段完整 FL business flow，
並同步收集 Path RAM 與 RTX 3080 VRAM peak。完整證據見
[Nupf Contract 與 PseudoDriver Runtime Smoke](../reports/5g-nwdaf-infrastructure/nupf-contract-pseudodriver-runtime-smoke-2026-08-09.md)。

### 16.19 2026-08-09 雙 E2E Scenario 與 Training Data 決策

後續不再讓單一 traffic profile 同時代表快速 plumbing test 與正式 example。主場景
`full-core-cat-transition` 延續舊 testbed 的 30 秒 slot、900 秒 historical warm-start、
15 分鐘 stable live 後 CAT1→CAT2，以及 optional CAT2→CAT3；warm-start 的唯一必要目的
是讓 PyAnLF 立即取得完整 30-step input。快速場景 `fl-closure-smoke` 才使用 3000 秒
historical burst，同時為 PyAnLF 與 PyMTLF 預備較多資料，並以較少的 reference/hit policy
和兩個 local epochs 降低等待時間。兩個場景都維持 90 秒 accuracy report period。

兩個場景都保留正常 Consumer／NRF／NWDAF／SMF／UPF／PyAnLF／ADRF 路徑、A-only changed
profile、B stable control、兩輪 sample-count-weighted FedAvg、final validation、publication、
reprovision 與 monitor cutover；smoke 不使用 fabricated degradation，也不構成主 business
acceptance。

PyMTLF preparation 不新增 32-sample minimum。C 維持 `minNumSamples=1`，其計數單位是
chronological split 後真正用於 fitting 的 training samples；batch size 32 也不是 admission
minimum。以目前 model/split contract，完整 final validation 需要至少 62 observations 才能
留下彼此不重疊的一筆 training 與一筆 validation evidence。主場景預期在第一次 trigger
附近約有 69 observations，雖只形成約 8 train／1 validation，仍是應被允許的有效 FL
輸入；smoke 的 100 observations 則約形成 36 train／4 validation。

正常 distributed FL 仍由 C 提供 3600 秒 explicit preparation window；這是最大回溯範圍，
不是要求填滿的資料量或 minimum。A／B fallback 後續需由 1800 對齊 3600 秒，dataset audit
新增 actual observation／train／validation counts。既有 100,000-record job ceiling 只保留作
下載安全界線。本節是已確認的後續設計，尚未表示兩份新 profile、config override、checker
或 runner 已實作。

### 16.20 2026-08-10 Stateless Backend Lifecycle 遷移

Infrastructure 對齊新版 stateless backend lifecycle，component pins 更新為 NWDAF
`318ac19d8b027373f4468660394da1ec3338268e`、PyAnLF
`5c305c7b69a50e9356bcfca8229f1a3cffd11a9a` 與 PyMTLF
`49b1ef474472559a487b4cf36d312265c45b0c9a`。新版 contract 已移除 full-state
`POST /internal/v1/sync`、`/health/live`、`SyncProjection` 與 `SYNCING` 狀態；Go NWDAF
只以 ready 狀態和 `processInstanceId` 追蹤 backend generation，backend replacement 時清除
舊 generation。PyAnLF／PyMTLF 在真正需要 context 時，才向所屬 NWDAF 的 thin internal
context API 查詢，不再維護一份由週期性 sync 推送的完整投影。

五個 Host ML service 的 containing-NWDAF 對應為：PyAnLF-A
`http://192.168.57.41:8090`、PyAnLF-B `http://192.168.57.51:8090`、PyMTLF-A
`http://192.168.57.41:8091`、PyMTLF-B `http://192.168.57.51:8091`、PyMTLF-C
`http://192.168.57.30:8091`。這些位址由 config renderer 根據各 `nwdafcfg-*.yaml` 的
AnLF／MTLF internal server 產生，config checker 會比對 ML 與 NWDAF 兩側設定並驗證 30 秒
request timeout，避免 topology 改動後留下獨立寫死且不一致的 endpoint。Compose image build
revision 與 `components.lock.yaml` 也同步前進到上述版本。

靜態驗證範圍包括 NWDAF `go test ./...` 通過、PyAnLF／PyMTLF Ruff 通過，以及
`nwdaf-resources` focused full-core support tests 10 passed。兩個 Python repository 的完整
suite 在目前既有 `.venv` 會於空白 FastAPI `TestClient` context enter 同樣停住，因此本次
不能宣稱完整 Python suite green，也不以修改 ML 程式規避該環境問題。先前 runtime reports
仍是當時 revision 的歷史證據，其中 periodic sync 成功不代表新版 stateless contract 已完成
runtime 驗證。

下一個 runtime gate 是重建五個 ML images，短時間驗證三台 Guest 到五個 Host container、
五個 container 回到各自 NWDAF internal context API 的雙向連線，再執行 bounded full E2E。
在這個 gate 完成前，不把舊 full-state sync smoke 當成新版 lifecycle 的驗收結果。

### 16.21 2026-08-10 Stateless Full E2E 與 ADRF-only PyAnLF Policy

stateless lifecycle 的雙向 transport gate 已通過。`fl-closure-smoke` 中三台 VM、23 個 Guest
services、五個 Host ML containers 與六個 UE 正常啟動；單一 consumer 只建立 A/B 各一筆
subscription。雙 Path 各完成 18,006 packets warm-start 與 26 個 30 秒 live windows，consumer
累計收到 56 筆 analytics notifications。Core database 最終有 420 筆 ADRF data records，
PyAnLF-A/B fallback collections 都是零，證明這次實際 observation path 使用 ADRF。

本輪沒有完成 FL closure。PyAnLF-A/B 各只產生一筆 `matched=1`、`deviation=None` 的 accuracy
measurement。兩筆 C→A/B monitor subscription 在 22:12:04 建立，第一筆 measurement 到
22:13:34 才 ready，恰好撞上快速場景的 90 秒 watchdog deadline；通知再耗時約 5 秒後，A
timeout、B 到達 C 但未被評估為 degradation，C 隨即將 A/B subscription 都視為 missing
periodic reports 並刪除。後續沒有 report 的直接原因是 monitor resources 已不存在，下一個
blocker 因而縮小到「首報 startup latency、notification latency 與 C watchdog policy 的時序
不相容」。ADRF periodic NRF heartbeat 的 HTTP 400 是另一個需獨立診斷的問題。詳細 runtime 證據見
[Stateless Full E2E Smoke](../reports/5g-nwdaf-infrastructure/stateless-full-e2e-smoke-2026-08-10.md)。

後續 E2E 將 PyAnLF 的 ADRF 路徑改為 fail-closed：default A/B config 設定
`mongodb.enabled: false`，renderer 產生的完整與快速場景繼承此值，checker 也禁止重新啟用
fallback。Core MongoDB 仍保留給 5GC NFs 與 ADRF；這項 ADRF-only policy 不涉及 PyMTLF。
三組 config、Compose render 與 PyAnLF config loader 均已驗證，PyAnLF config tests 24 passed。

依變更邊界，目前不直接修改 PyAnLF、PyMTLF 或其他 NF/ML source。為先排除首報與 deadline
相撞，使用者已核准只調整 PyMTLF-C deployment config：明確設定
`watchdog_grace_seconds: 300`。當時 30 秒 smoke report period 的 deadline 由 90 秒延長為
360 秒，完整場景則為 480 秒；三組 config/Compose、PyMTLF loader 與 48 個 config tests
均通過。

後續 bounded rerun 證明 grace 調整有效：A/B monitor subscriptions 越過舊 90 秒 deadline，
連續 report 都立即送達 C，沒有再出現 5 秒 timeout。但每個 30 秒 report window 只產生一個
group-level prediction/ground-truth pair；三個 UE 是同一 pair 內的 contributors，不是三個
matched samples。因此每輪固定為 `matched=1`，低於既有 minimum 2，所有 report 都是
`deviation=None`。

為保留既有 accuracy 語意，不把 smoke 的 minimum 降為 1。`fl-closure-smoke` report period
恢復為 90 秒，讓每個 window 正常累積約三個 group-level time-slot pairs；minimum reference
reports 2、decision window 2、required hit 1 與 local epochs 2 仍保留快速場景特性。兩個場景
的 watchdog deadline 現在都為 `90 × 2 + 300 = 480` 秒。

恢復後的 bounded rerun 已確認 A/B 第一份可評估 report 都為 `matched=2`，後續為
`matched=3`，並帶有非空 deviation；C 在第三輪 report 正常偵測 A-only degradation 並自動
啟動 FL。A/B 都從 ADRF 取得 27 筆 records、以 44 個 samples 在 GPU 完成 round 0 與 round 1
local training，C 也完成 round 0 FedAvg。

C self-origin 已經使用者確認後加入 infrastructure default、config renderer 與 checker；三組
config check、兩種 Compose check 與 PyMTLF 48 個 config tests 均通過。後續 rerun 證明原本的
origin blocker 已排除：C 能下載自己的 round 0 global artifact，A/B 在 GPU 完成兩輪 local
training，C 也完成兩輪 FedAvg。

新的 blocker 位於 final validation。GPU 支援 commit 將 `LocalTrainer._predict` 改為必須接收
`device`，但 `fl_client.py` 的 base/candidate validation 呼叫仍沿用舊簽名，A/B 因
`TypeError: LocalTrainer._predict() missing 1 required positional argument: 'device'` 回報
`NOT_AVAILABLE_ML_TRAIN`。使用者核准後，PyMTLF client 已從 configured training device
resolve 一次執行裝置，並在 base／candidate final-validation prediction 都傳入同一 device；新增
regression test 會實際執行 validation 並確認兩次 prediction 的 device。Targeted FL client、
trainer、server、dataset、scope、workspace、artifact、wire 與 config tests 均通過；完整 suite
仍受既有 FastAPI/TestClient shutdown hang 阻擋，不能宣稱全套通過。

### 16.22 Configuration Contract Hardening

在下一次 runtime 前，先針對重複出現的簡單配置錯誤完成跨層 contract hardening：

- `reportPeriodSeconds` 必須整除 sampling interval，且單一 report 的理論 sample capacity 必須
  大於等於 PyAnLF `min_matched_predictions`；
- stable lead、degraded tail、bounded trigger、preparation window 與 scenario-owned closure
  budget 必須同時成立；
- C artifact allowlist 必須包含 A、B 與 C self-origin；PyAnLF／PyMTLF public URL、callback、
  MongoDB、interoperability 與 seed model shape 必須逐欄對齊；
- topology 的 `gtpInterface` 必須實際 render 為 UPF `gtpu.ifList[].ifname`；
- generated manifest 的 baseline hash、topology hash 與 file inventory 必須仍對應目前來源；
- A/B FL fallback deadlines 必須與 C preparation／round deadlines 相同，monitor missed-report
  threshold 也改為 deployment config 明確值，不依賴 library default。

另新增 temporary negative contract smoke，刻意破壞 report capacity、C self-origin、GTP
interface、FL public URL、Consumer callback、manifest baseline hash 與 config generator hash，
七種錯誤均確認會被 checker 拒絕。修正後 smoke dataset identity 為
`2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`；完整場景 identity 為
`c3b428ea763834f34b2ff3a7e7674b5d082a2685e3825595f0b5cc33c356bb49`。下一個 gate 才是以
修正後 image 與 dataset 重跑 bounded smoke；若再需要 NF／ML source change，仍先停止回報。

### 16.23 2026-08-12 Persistent Netplan Alias Ownership 計畫

Core VM 以 `vagrant up core --no-provision` 進行 cold-boot audit，實際證實 alias 問題不是
reset 或 Makefile 本身造成，而是 Guest boot 與 Vagrant post-SSH network configuration 的
固定時序衝突：

- `5g-nwdaf-network.service` 於 `17:13:25 UTC` 成功加入 Core 的 14 個 aliases，並將清單寫入
  `/run/5g-nwdaf-infrastructure/network-aliases`；
- Vagrant 於 `17:13:31 UTC` 覆寫 `/etc/netplan/50-vagrant.yaml`，networkd 在
  `17:13:32`–`17:13:33 UTC` reconfigure 四張 host-only interfaces；
- 最終 `ip address` 只剩 `.56.10`、`.57.2`、`.58.2`、`.61.2` anchors，14 個 aliases 全部
  消失，但 oneshot 因 `RemainAfterExit=yes` 仍回報 `active (exited)`。

Host 上的 Vagrant 2.4.3 implementation 也確認 Debian／Ubuntu guest capability 只會覆寫
`50-vagrant.yaml`，接著執行 `netplan apply`，不會刪除其他 Netplan fragments。Jammy Guest
使用 Netplan `0.106.1`；隔離 `--root-dir` 測試加入
`60-5g-nwdaf-aliases.yaml` 後，`netplan get` 與生成的 networkd unit 都同時保留 Vagrant
anchor `.57.2` 和測試 aliases `.57.10`、`.57.18`。因此採用後置 Netplan fragment 是已由
目前實際版本驗證的 merge path，不是假設未來 Vagrant hook 行為。

預定調整分六個 bounded steps：

1. 將 `network-setup.sh` 改為 candidate render、隔離 generate、atomic install、targeted
   networkd reconfigure、effective address verification 與 fragment rollback；`--clear` 改為
   移除 fragment 後重新產生並套用，不再依賴 `/run` state file。
2. 調整 `config-activate.sh`，讓 active symlink 與 persistent fragment 成為同一個 rollback
   boundary；candidate config 不得在任何 NF、consumer 或 ML backend 運行時 hot switch。
3. 調整 `5g-nwdaf-network.service`／`common.sh`，移除 boot correctness 對 early oneshot 的
   依賴；保留手動 reconcile 與 service dependency，並為既有 VM 清理舊 runtime state。
4. 調整 `services-start.sh` 與 reset repair guidance：正常路徑先驗證 addresses，不再每次
   unconditional restart；drift 或 migration 狀態才 reconcile。
5. 先在 Core 驗證 config activation、同 config idempotence、alias removal、invalid candidate
   rollback、Guest reboot，以及不經 Makefile 的直接 `vagrant halt`／`vagrant up`。其中 reset
   必須能在 services-start 之前直接連上 MongoDB `.57.18`。
6. Core gate 通過後才擴至 Path A／B，驗證 SBI、N2、N3、N4、N6 aliases、三 VM cold boot、
   config-set switch 與一次 bounded guest stack smoke；最後確認三台 VM 回到預期 power state。

此重構只涉及 `5G_NWDAF_Infrastructure` 的 Guest/Host scripts、systemd unit、focused tests 與
對應 `testbed-docs`。它不變更 NF／ML source、component revisions、Vagrant adapter topology、
public IP plan 或 `nwdaf-docs`。實作必須分階段 commit；若 isolated Netplan validation、targeted
reconfigure 或 rollback 無法避免影響 management／SSH，應停下來回報，不以 full
`netplan apply` 靜默擴大變更範圍。

上述六步已於同日完成。三台 migration、cold boot、Core reset-before-services、drift recovery、
failure-injection rollback 與 23-unit bounded Guest stack smoke 全部通過；驗證後 services 與 VM
均停止。實際證據與 revision identity 見
[Persistent Netplan Alias Migration](../reports/5g-nwdaf-infrastructure/persistent-netplan-alias-migration-2026-08-12.md)。

### 16.24 2026-08-12 FL Closure Smoke 與 Netplan Regression

在 `7699574` 的 Persistent Netplan implementation 上，以 fresh rendered
`fl-closure-smoke`、confirmation-gated clean state 與三台 VM poweroff 起點完成 bounded E2E。
Cold boot 後 network unit 沒有執行，Core 14、Path A／B 各 7 個 aliases 已由 Netplan
持久恢復；reset-before-services、23-unit startup 與 Host ML startup 都不需額外 reconcile。

單一 consumer 建立兩筆 TAI-specific subscriptions。A-only degradation 在 subscription 後約
481 秒觸發唯一 federated process；A／B 各以 49 samples 完成兩輪 GPU local training，C 完成
FedAvg、final validation 與 ADRF publication。Candidate WAPE 為 `0.2464`，低於 base 的
`1.8398`；新 model `1786505512331` 被 A／B 採用，trigger 後約四秒完成 monitor generation
cutover。A 隨後產生 `matched=2` 的新 model report，證明 post-cutover inference 與 monitor
route 持續運作。

Concurrent training 的兩個 GPU processes 各約使用 400 MiB VRAM，Host 最低觀察到約 24 GiB
available RAM。五個 ML containers 沒有 ERROR、traceback 或 collection failure。完成後兩筆
subscriptions 精確 DELETE，ML／Guest services 依序停止，三台 VM 均 graceful poweroff；本輪
state 保留供後續查驗。完整 identity、timing、resource 與 teardown 證據見
[FL Closure Smoke 與 Persistent Netplan 回歸](../reports/5g-nwdaf-infrastructure/fl-closure-smoke-netplan-regression-2026-08-12.md)。

### 16.25 2026-08-12 Runtime Helper Sync 與 FL Lifecycle Regression

為避免既有 VM 沿用 provision 當時的 stale runtime scripts，Infrastructure 將 helper 安裝集中
到單一 Guest installer。全新 provision 與 `services-start` 前的 Host sync 共用同一份 allowlist；
Host 上傳 archive 後，Guest 驗證完整 SHA-256 才安裝 Consumer、config／network／dataset
helpers、subscriber projection 與 systemd definitions。同步只執行 `daemon-reload`，不啟動或
重啟 NF。三台既有 VM 已實測安裝相同 bundle identity，config activation 與 subscriber apply
也確實經過新 helper；repository tests、cold boot、28 個 aliases 與 23-unit startup 均通過。

同一輪 GPU `fl-closure-smoke` 再次完成 A-only degradation、A／B 兩輪 local training、C
FedAvg、ADRF publication、A／B adoption 與 post-cutover accuracy report。第一個 candidate
WAPE `0.2464` 優於 base `1.8398`，trigger 後約四秒完成 cutover，因此 helper sync 沒有破壞
既有 closure。

延長運行也釐清兩項 lifecycle 邊界。第一，`experiment-start` 是持續運行入口，不會在首次
closure 後自動停止；持續 degradation 約四分半後再次觸發 FL。第二次 candidate WAPE
`0.4575` 劣於 base `0.2467`，但 smoke 的 deployment enforcement=false，因此仍依 config
語意發布。bounded acceptance 應在首次 cutover 後等待一個新 generation report window 便
停止；若要無人值守，後續應以通用 closure condition／watcher 表達，不重新增加多組
scenario-specific smoke commands。

正常 teardown 已精確 DELETE 兩筆 consumer subscriptions，停止五個 containers、23 個
Guest units，並 graceful poweroff 三台 VM；但 PyMTLF-C 在 Compose shutdown 期間留下單次
monitor subscription DELETE `503`。下一個 implementation gate 是診斷並讓 teardown 在停止
ML backend 前等候 monitor delete convergence。此工作先限於 lifecycle orchestration；若確認
需要修改 PyMTLF／NWDAF source，仍必須先向使用者說明原因並取得同意。完整 timing、identity、
state 與資源證據見
[Runtime Helper Sync 與 FL Lifecycle 回歸](../reports/5g-nwdaf-infrastructure/runtime-helper-sync-and-fl-lifecycle-regression-2026-08-12.md)。
