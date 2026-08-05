# `5G_NWDAF_Infrastructure` 建置與遷移計畫

日期：2026-08-05

狀態：Draft；repository 名稱與第一版方向暫定，尚未授權建立 remote repository 或實作部署

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
- 三台精簡 VM 的角色與網路邊界；
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
- host 目前只剩約 56 GiB 可用磁碟、swap 幾乎用滿，且 VirtualBox kernel driver
  尚不可用，不適合直接複製舊 VM 方式重建。

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
8. **generated state 與 source 分離**：VM disk、binary cache、Python environment、
   MongoDB、ADRF data、log 與 pcap 都不是 submodule。
9. **第一版不支援憑證**：不提供 certificate、TLS 或 OAuth 管理；環境明確限定
   在隔離實驗網路以 HTTP 執行，不暗示 production security readiness。
10. **先保存、再清理舊 VM**：由於本機空間不足，新 VM 建立前先盤點、備份並在使用者
    確認 exact targets 後移除舊 local VM；舊 `5G_Infrastructure` repository、本地腳本
    與 migration inventory 仍保留到新環境通過 fresh-clone E2E。

## 4. Repository 責任與邊界

`5G_NWDAF_Infrastructure` 負責：

- component revision 組合；
- build orchestration 與 artifact identity；
- VM topology、network 與 single-project Vagrant orchestration；
- host-side preflight／start／stop／status；
- guest-side OS setup、role-local build、network 與 service lifecycle；
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
- `scripts/host/` 在實體機協調 Vagrant，`scripts/guest/` 由 Vagrant 在 VM 內執行；
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

## 8. 三 VM Compact Topology

### 8.1 角色分配

| VM | 主要元件 | 邊界 |
| --- | --- | --- |
| `core` | NRF、AMF、SMF、UDR、UDM、AUSF、NSSF、PCF、NWDAF-C、PyMTLF-C、ADRF、MongoDB；optional webconsole | shared control、model owner、FL coordinator、persistent services |
| `path-a` | UPF-A、gNB-A、UE1–3、NWDAF-A、PyAnLF-A、PyMTLF-A | TAI `000001` analytics／FL client path |
| `path-b` | UPF-B、gNB-B、UE4–6、NWDAF-B、PyAnLF-B、PyMTLF-B | TAI `000002` analytics／FL client path |

初始 logical mapping 延續已驗證 scenario：

- PLMN `466/92`；
- S-NSSAI `sst=1, sd=010203`；
- DNN `internet`；
- Path A：TAI `000001`、UE pool `10.60.0.0/16`；
- Path B：TAI `000002`、UE pool `10.61.0.0/16`；
- 同一 Internal Group 的六個 UE 分散在兩個 TAI；
- A 可切換到 degraded traffic profile，B 保持 stable control profile。

### 8.2 Network planes

第一版 topology 至少明確區分：

- management/provisioning：Vagrant SSH、artifact deployment、health collection；
- SBI/service：core NF、NWDAF、ML backend、ADRF、MongoDB 與 callback；
- N2：兩個 gNB 到 AMF；
- N3-A／N3-B：各 gNB 到對應 UPF；
- N4：SMF 到兩個 UPF；
- N6-A／N6-B：UE data network／egress；
- UE pools：兩條 path 不重疊。

Public default 應優先使用 isolated private／host-only network 與受控 NAT，不把所有
plane bridge 到實驗室實體 LAN。只有 gitignored `testbed.local.yaml` 可以選擇實體
bridge、provider-specific storage expectation 或 lab gateway，且必須通過衝突檢查後
才套用。

### 8.3 初始資源預算

資源配置先作為量測起點，不作固定承諾：

| VM | 初始 RAM | 初始 vCPU | 動態磁碟上限 |
| --- | ---: | ---: | ---: |
| `core` | 6 GiB | 6 | 35 GiB |
| `path-a` | 5 GiB | 4 | 25 GiB |
| `path-b` | 5 GiB | 4 | 25 GiB |
| 合計 | 16 GiB | 14 | 85 GiB |

正式建立 VM 前必須重新量測 host available RAM、swap、free disk 與現有 VM 使用量。
動態磁碟上限不等於立即占用，但現有約 56 GiB free space 沒有足夠安全餘裕同時保存
舊 VM、guest build cache 與新三 VM。已決定在新 VM 建立前先完成舊 local VM 的保存
與清理；若新舊 VM 位於同一 filesystem，清理後的暫定目標是至少 120 GiB free space，
最終 threshold 仍由 provider location、動態 disk policy 與 preflight 計算。

清理不等於直接刪目錄。執行前必須先確認每台舊 VM 的 Vagrant project、provider name、
UUID、storage path、disk/snapshot size 與 state，保存 guest-only config／script／data／log
或整台 VM 的可恢復副本，驗證 backup manifest 後，再列出 exact removal targets 讓
使用者確認。預設優先由原 Vagrant project 執行 destroy；只有 Vagrant metadata 已失效
時才考慮 provider-specific unregister/delete。不得以 broad path 或 glob 清除。

Virtualization provider 尚未定案。VirtualBox 目前缺少可用 `/dev/vboxdrv`；在選擇
Vagrant + VirtualBox、Vagrant + libvirt 或其他 provider 前，先做只影響 host
prerequisite 的 feasibility check。Topology 與 config 不應綁死 provider-specific
介面名稱。

### 8.4 單一 multi-machine Vagrant project

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
provider metadata，必須 gitignore；它不是虛擬磁碟。若採 VirtualBox，真正的 VM
disk 位於 VirtualBox global machine folder，box cache 則位於 Vagrant 自己的 cache。
若採 libvirt，disk 位於選定 storage pool。兩者都在 repository 外。

`testbed.local.yaml` 可以記錄預期 provider 與 VM storage location，讓 preflight 回報
實際位置和 free space；它不應在 `vagrant up` 時靜默修改 provider 的 global machine
folder。任何 global storage 變更都屬於獨立 host setup，必須先確認對既有 VM 的影響。

## 9. 設定檔與 Testbed 定義

### 9.1 `config/default/`：可直接執行的 baseline

`config/default/` 保存 component 真正讀取的完整 native config，延續 free5GC-style
flat filenames；只有 UERANSIM 等本來就適合分組的檔案使用子目錄。Guest scripts 再依
placement 將同一 config set 的適當檔案交給 Core、Path A、Path B。內容涵蓋：

- free5GC core NF、UPF-A/B 與 `uerouting`；
- NWDAF-A/B/C、ADRF、PyAnLF、PyMTLF 與 optional webconsole；
- gNB-A/B 與 UE1–6；
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

輸出只寫入 `config/generated/<name>/`，維持和 default 相同的 file layout，並產生
包含 baseline hash、definition hash、generator revision、generated file list 與 effective
network identity 的 manifest。Renderer 必須用 typed YAML/object mutation 修改已宣告欄位，
不得用文字取代、regex 或任意 deep merge 猜測 component schema，也不得依賴另一份
`free5gc-main` checkout。

`config-check.py` 同時支援 default、generated 與 local sets，至少檢查：

- Vagrant/testbed network 與 component bind／advertise address 一致；
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
- component 到 VM 的 placement；
- PLMN、S-NSSAI、DNN、TAI、UPF／UE pools 與生成 component config 所需的 network
  identity；
- Vagrant/provider-neutral defaults 與 two-TAI／two-UPF reachability。

需要另一組完整 topology 時，使用者可複製 `testbed.yaml` 到自選 definition file，經
Makefile 的 explicit `TESTBED=<path>` 選用；同一個 definition 同時交給 Vagrant、renderer
與 config checker，避免兩份網路 source of truth。

`testbed.local.example.yaml` 說明可覆寫欄位；`testbed.local.yaml` gitignored，只保存
physical host／provider 差異與 active `configDir`，例如 VirtualBox/libvirt、physical
bridge/interface、expected VM storage、host port forwarding 與 optional outbound gateway。
Local override 不直接改變 TAI、UE identity、NWDAF ownership 或 FL assertion；這些語意
變更必須放進 explicit topology definition 和完整 config set。

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
  release: "24.04"

machines:
  core:
    resources:
      memoryMiB: 6144
      cpus: 6
      diskGiB: 35
    interfaces:
      management: 192.168.56.10
      sbi: 192.168.57.2
      n2: 192.168.58.2
      n4: 192.168.61.2

  path-a:
    resources:
      memoryMiB: 5120
      cpus: 4
      diskGiB: 25
    interfaces:
      management: 192.168.56.11
      sbi: 192.168.57.3
      n2: 192.168.58.3
      n3-a: 192.168.59.2
      n4: 192.168.61.3
      n6-a: 192.168.62.2

  path-b:
    resources:
      memoryMiB: 5120
      cpus: 4
      diskGiB: 25
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
    - pymtlf-c
    - nwdaf-consumer
  path-a: [upf-a, gnb-a, ue1, ue2, ue3, nwdaf-a, pyanlf-a, pymtlf-a]
  path-b: [upf-b, gnb-b, ue4, ue5, ue6, nwdaf-b, pyanlf-b, pymtlf-b]

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
    pyanlf-a: {machine: path-a, address: 192.168.57.42, port: 9093}
    pyanlf-b: {machine: path-b, address: 192.168.57.52, port: 9093}
    pymtlf-a: {machine: path-a, address: 192.168.57.43, port: 9092}
    pymtlf-b: {machine: path-b, address: 192.168.57.53, port: 9092}
    pymtlf-c: {machine: core, address: 192.168.57.31, port: 9292}

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

### 9.5 第一版不管理 experiment history

VM lifecycle 與單次 experiment 分開：三台 VM 可以 `up`／`start` 一次，再連續執行多次
實驗。第一版不定義 run-id、`runs/`、`runtime/`、自動 log collection 或 archive schema。
Log 與 service state 先留在各 guest，需要時以 `vagrant ssh`、`journalctl` 或 component
自身工具查看。等 VM-aware experiment runner 的 ownership 和需求確定後，再獨立設計
跨多次 experiment 的 identity、collection 與保存方式。

## 10. Host／Guest Scripts 與 Build Lifecycle

### 10.1 Host scripts

`scripts/host/` 在實體機執行，只協調 source identity、Vagrant、三台 VM，以及 service／
subscription lifecycle，不編譯 guest component，也不在 Host 執行 consumer：

- `preflight.sh`：唯讀檢查 provider、RAM、swap、VM storage free space、submodule、
  testbed files、active config set、必要網段與 port，並呼叫 config checker；不修改 host
  network 或建立 VM；
- `config-render.py`：選用地由 default baseline 與 explicit topology definition 產生一組
  `config/generated/<name>/` native configs 和 manifest，不覆寫 committed/default files；
- `config-check.py`：唯讀驗證 default／generated／local config set 的跨 component network、
  identity、placement、A/B mapping 與所選 testbed definition 一致；
- `services-start.sh`：解析 effective `TESTBED`／`CONFIG_DIR`，先 check、stage、activate 完整
  config set，再依 dependency order 透過 Vagrant SSH／guest service interface 啟動 Core、
  Path A 與 Path B 的 database、NF、RAN、NWDAF 與 ML process；
- `services-stop.sh`：停止 experiment stack service/process，不等於 `vagrant halt`，
  更不等於 destructive `vagrant destroy`；
- `services-status.sh`：只彙整 core NF、UPF、gNB／UE、NWDAF／ML 與 database health；
  VM power state 另外由 `vagrant status` 或 `make vm-status` 回報；
- `subscriptions-start.sh`：透過 Vagrant SSH 在 Core 啟動單一 consumer callback，確認
  A/B callback reachability，要求 consumer 經 NRF discovery 找到兩個不同的 NWDAF，
  建立兩筆 subscription；部分成功時負責 rollback；
- `subscriptions-status.sh`：彙整 Core consumer／callback 狀態、discovered A/B NF
  identities、兩筆 subscription identities 與最近 notification；
- `subscriptions-stop.sh`：要求 Core consumer 依保存的 exact `Location` 退訂兩側，成功後
  停止 callback；失敗時保留可重試的 state，不等於停止 5GC services；
- `observe.sh`：週期性顯示三台 VM、service health、discovered A/B、subscription 與最近
  callback 的 compact terminal view，不解析完整 business log；
- `logs.sh`：透過 Vagrant SSH 並行 follow 各 guest 的 journald，為每行加上 VM／unit
  prefix，支援依 VM、service 與 since time 過濾；結束 follow 不停止任何 guest process。

Renderer／checker 在 Host 執行，因為它們在 VM 建立前決定 effective config set；guest
只能接收已通過 check 的完整 config set，不在 VM 內自行組合或回寫設定。

### 10.2 Guest scripts

`scripts/guest/` 由 Vagrant 上傳／執行：

- `common.sh`：三台 VM 共用的 OS package、Go、`uv`、runtime user、directory 與基本
  observability；
- `config-activate.sh`：驗證 staged config identity 與 guest role，在 services／consumer
  都停止時原子切換 `/etc/5g-nwdaf-infrastructure/active` symlink，不修改
  committed/shared source；
- `core.sh`：Core 的 MongoDB、core NF、ADRF、NWDAF-C、PyMTLF-C 與 optional
  webconsole setup／build；
- `path.sh A|B`：Path A/B 的 kernel headers、network、gtp5g、UPF、UERANSIM、NWDAF、
  PyAnLF 與 PyMTLF setup／build。

概念流程：

```text
core   -> common.sh -> core.sh
path-a -> common.sh -> path.sh A
path-b -> common.sh -> path.sh B
```

Provisioning 與 deployment 仍是不同 lifecycle 語意，但不各占一個頂層目錄；scripts
內部以 idempotent setup／build／start／stop action 區分。Guest 不自行 clone branch、
不編輯 submodule source，也不把 generated data copy 回 source tree。

### 10.3 Guest-local build responsibility

Host 只將 parent 已固定的 clean source snapshot 與 config 交給對應 VM。三台 VM 分工：

| VM | Guest-local 建置／環境 |
| --- | --- |
| `core` | NRF、AMF、SMF、UDR、UDM、AUSF、NSSF、PCF、NWDAF-C、ADRF、PyMTLF-C；optional webconsole |
| `path-a` | gtp5g、UPF-A、UERANSIM、NWDAF-A、PyAnLF-A、PyMTLF-A |
| `path-b` | gtp5g、UPF-B、UERANSIM、NWDAF-B、PyAnLF-B、PyMTLF-B |

UERANSIM 在 `path-a`／`path-b` 各編譯一次，產生各 VM 自己的 `nr-gnb`／`nr-ue`。
這避免 host C++ toolchain、額外 builder 與 host/guest library compatibility 門檻；A/B
差異仍來自 config，不是不同 source revision。每台記錄 source commit、guest OS、
compiler/CMake 與 binary hash，可有 keyed local cache，但 cache miss 必須可 clean build。

`gtp5g` 必須配合各 path guest kernel 建置。先測 pinned upstream 原版；只有實際 guest
kernel 需要時，才把 reviewed patch 放在 `kernel/` 內靠近 gtp5g，不預先建立泛用
`patches/`。Python component 以各自 lockfile 和 `uv sync --frozen` 建立 role-local
environment。建置後可清除 apt、Go 與 compiler cache，以控制動態磁碟占用。

### 10.4 Operations

VM power、5G/NWDAF services 與 NWDAF subscriptions 是三組獨立操作：

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

make subscriptions-start
make subscriptions-status
make subscriptions-stop

make observe
make logs
```

`make vm-up` 對應 multi-machine `vagrant up`：第一次建立 VM 時執行 guest setup/build，
之後只負責讓既有 VM 開機；它不啟動 experiment stack。`make services-start` 才按順序
啟動 MongoDB、NRF、其他 core NF、Path A/B UPF 與 SMF、ADRF/NWDAF-C、A/B
NWDAF/ML，最後才啟動 gNB/UE。

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
| `subscriptions-start` | 呼叫 `scripts/host/subscriptions-start.sh` | 是；包含 callback、NRF discovery、雙訂閱與 rollback |
| `subscriptions-status` | 呼叫 `scripts/host/subscriptions-status.sh` | 是；包含 discovery／subscription state 與 notification status |
| `subscriptions-stop` | 呼叫 `scripts/host/subscriptions-stop.sh` | 是；包含精確退訂與 failure-state preservation |
| `observe` | 呼叫 `scripts/host/observe.sh` | 是；週期性組合 VM／service／subscription 摘要 |
| `logs` | 呼叫 `scripts/host/logs.sh` | 是；跨 VM journald follow、prefix 與 filter |

如果後續 `vm-up` 需要 provider 選擇、storage preparation 或額外錯誤處理，再抽成
`scripts/host/vm-up.sh`；第一版不為了名稱對稱先建立空殼 wrapper。Guest scripts 則只處理
VM 內 provisioning／build／systemd unit 安裝，不作為 Host 的使用者入口。

#### Effective config activation

`services-start` 不讓 process 直接讀 Host `config/`，而是執行以下流程：

1. 依 `CONFIG_DIR` explicit argument、`testbed.local.yaml.configDir`、`config/default/` 的
   優先序解析完整 config set，並解析同一次命令使用的 `TESTBED` definition；
2. 執行 `config-check`，計算 config manifest/hash，並比對 running VM 內 provisioning 時
   保存的 testbed/network identity；若 VM network 不相容，停止並要求明確 reload／
   reprovision，不在 service start 時偷偷改 guest network；
3. requested config identity 與 active config 相同時可 idempotent reuse；兩者不同時，必須
   確認三台 VM 上的 experiment services 與 Core consumer/subscriptions 都已停止，不得
   hot switch；
4. 透過 Vagrant upload／SSH 將各 guest role 需要的 files 與完整 manifest stage 到
   `/etc/5g-nwdaf-infrastructure/config-sets/<config-name>-<short-hash>/`；
5. 三台 Guest 先各自驗證 staged files；全部 prepare 成功後，Host 才要求
   `config-activate.sh` 原子更新 `/etc/5g-nwdaf-infrastructure/active` symlink。任一 guest
   activation 失敗時，Host 將已切換的 guest 回復到先前 link，且不啟動 process，避免
   half-activated set；
6. systemd units 使用固定 path，例如
   `ExecStart=... --config /etc/5g-nwdaf-infrastructure/active/nrfcfg.yaml`，再由既定
   dependency order 啟動；unit file 不因 config set 改變而重寫；
7. `services-status`／`observe` 回報三台 VM 的 active config hash，三者不同或與 Host
   selected config 不同時視為錯誤。

因此只改 component 參數且 VM network identity 不變時，可以先 `services-stop`，再以另一個
`CONFIG_DIR` 重新 `services-start`，不必重建 VM。若變更 management/SBI/N2/N3/N4/N6
介面或 Vagrant network，則必須先讓 VM 套用相同 `TESTBED` definition。後續
`subscriptions-start` 只讀 Core 已 active 的 `consumer.yaml`，不接受另一組臨時 config，
避免 services 與 subscriptions 使用不同組合。

`destroy`、database clear、VM removal 與舊環境清理必須是獨立的 destructive command，
具備 exact target、dry-run／confirmation 與 recovery 說明，不能隱含在 `vm-up`、
`services-start` 或 scenario retry 中。

### 10.5 Process supervision 與跨 VM 啟動順序

舊環境主要依賴使用者逐台 SSH、直接執行 binary，或由 `run.sh` 以 background process／
PID list 管理。新環境改成：每個長時間執行的 component 在所屬 guest 內有一個 systemd
service unit，但 install 後保持 disabled，不隨 VM boot 自動啟動。

概念 service 包含：

- Core：MongoDB、NRF、NSSF、UDR、UDM、AUSF、PCF、AMF、SMF、ADRF、NWDAF-C、
  PyMTLF-C，以及 optional webconsole；
- Path A/B：UPF、gNB、UE instances、NWDAF、PyAnLF、PyMTLF；
- UE 可使用 systemd template unit，例如 `ueransim-ue@1.service`，讓 A 啟動 UE1–3、
  B 啟動 UE4–6。

Unit 至少明確指定 `User`、`WorkingDirectory`、`ExecStart`、config path、restart policy
與 journald output。Guest `core.sh`／`path.sh` 負責安裝 unit file 和 `daemon-reload`，但
不執行 `systemctl enable`，也不在 setup 結尾啟動 experiment service。

Systemd 只能管理同一台 VM，無法保證另一台 VM 的 NF 已 ready，因此跨 VM 順序由
host `services-start.sh` 負責。它不是一次平行執行所有 unit，而是分 stage 啟動並等待
readiness，例如：

```text
1. Core MongoDB -> wait database ready
2. Core NRF -> wait NRF SBI ready
3. NSSF/UDR/UDM/AUSF/PCF/AMF -> wait health and NRF registration
4. Path A/B UPF + Core SMF -> wait PFCP association and SMF registration
5. ADRF/NWDAF-C/PyMTLF-C -> wait analytics/model services ready
6. Path A/B NWDAF/PyAnLF/PyMTLF -> wait role registration and backend health
7. gNB-A/B -> wait NG setup
8. UE1–6 -> wait registration, PDU Session and expected UE pools
```

可安全平行的 A/B stage 可以平行執行，但每一 stage 必須有 bounded timeout、明確失敗
訊息與當下 service status。`services-stop.sh` 依相反 dependency order 停止，避免先停
database／core 而留下仍在寫入或 retry 的 UE、NWDAF 與 UPF process。

`services-status.sh` 同時檢查 systemd active state 與必要的 application-level health；
只看到 process active 不代表 NRF registration、PFCP、PDU Session 或 NWDAF backend 已
ready。Log 先留在 guest journald，可用 `vagrant ssh <vm> -c 'journalctl -u <unit>'` 查看。

需要單步除錯時仍可維持原本操作感：先停止該 unit，再 SSH 進 VM 手動執行 binary；
結束後再交回 systemd。自動化是取代重複操作，不限制人工診斷。

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

三組可獨立啟停的生命週期因此明確分開；traffic／degradation 等實驗動作在訂閱成立後
另外執行：

```text
time -------------------------------------------------------------->
VM:             vm-up ====================================== vm-halt
Services:              services-start =============== services-stop
Subscriptions:                        start ===== stop
Actions:                                  traffic / degradation
```

`subscriptions-start.sh` 必須先確認 `services-status`、A/B NWDAF readiness 與 Core callback
address 可達，再依序：

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
  registration、PFCP、UE registration／PDU Session、consumer discovery mapping、A/B
  subscription identity、callback count 與最後通知時間；
- 「剛才發生什麼」：由 `make logs` 即時 follow guest journald，查看原始 process log。

所有由本 repo 啟動的長時間 process，包括單一 consumer，都應將 stdout／stderr 交給
systemd journald。`logs.sh` 不要求 component 另外寫一份固定路徑 log，也不把 log copy
回 repository；它透過 multi-machine Vagrant SSH 對選定 unit 執行可中止、可限制起始時間的
`journalctl --follow`，並輸出例如：

```text
[core/nrf] ...
[core/nwdaf-c] ...
[path-a/nwdaf] ...
[path-a/pyanlf] ...
[path-b/nwdaf] ...
[core/nwdaf-consumer] ...
```

預設 view 只 follow 與目前 analytics／FL 路徑直接相關的 units，避免把所有 NF debug log
混成不可讀輸出；使用者可用 VM／service filter 看單一 process，或明確要求 full stack。
Ctrl-C 只結束 Host 上的 log follow SSH sessions，不停止 consumer、subscription、NF 或 VM。

為讓三台 VM log 可比較，guest setup 必須啟用同一可靠 time source，`preflight`／
`services-status` 回報相對 Host 的 clock skew；超過容許值時不得把跨 VM log ordering 當成
可靠證據。Journald retention 採明確且受磁碟上限約束的 guest policy，只提供近期診斷；
它不是實驗結果保存機制。若日後需要可回放或長期稽核，再另立 session／collection 設計，
不在第一版 live observation 中暗中加入。

## 11. 實作階段與 Gate

### Phase 0 — 決策與文件（目前階段）

Deliverables：

- 本計畫；
- 舊 site/network inventory；
- repository 名稱、owner、visibility、license 與 virtualization provider 待決清單。

Exit gate：使用者確認計畫、第一版 component scope 與預計建立的 repositories／branches。

### Phase 0.5 — 舊 VM 保存與本機清理

此階段位於新 repository bootstrap 與新 VM 建立之前，只清理 provider VM／disk，不刪除
舊 `5G_Infrastructure` source repository。

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

- 建立 `5G_NWDAF_Infrastructure` remote 與初始 `main`；
- 建立 `feat/tai-aligned-compact-testbed`；
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
- 確認匿名 HTTPS clone readiness。

Exit gate：`git submodule status --recursive` clean，所有 required component 可取得、
revision 可追溯且 license 無未解阻塞。

### Phase 3 — Component Config 與必要工具移植

Deliverables：

- 匯入／整理 `config/default/` 下可直接執行的完整 core、path-a、path-b 與單一 consumer
  native config set，附來源與 topology manifest；
- 定義 `testbed.yaml` topology schema、`testbed.local.yaml` 允許的 host-only override 與
  active `configDir`，並驗證它們和 component config 的 network／placement 一致；
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

### Phase 4 — 三 VM Skeleton 與 Network

Deliverables：

- provider 選型與 prerequisite check；
- root multi-machine `Vagrantfile` 內的 `core`、`path-a`、`path-b` VM；
- management、SBI、N2、N3-A/B、N4、N6-A/B plane；
- `testbed.local.yaml` override 與 public isolated default；
- `.vagrant/` metadata、provider VM disk 與 Vagrant box cache 的位置／free-space 回報；
- route、forwarding、NAT、MTU、port reachability smoke tests。

Exit gate：三 VM 可由 fresh testbed definition 重建；每個 plane 只具有設計允許的 reachability，
且不需要實驗室實體 bridge。

### Phase 5 — Core Guest Setup 與 Build

Deliverables：

- reproducible guest-local build/artifact manifest；
- `common.sh`／`core.sh` 與 Core 的 NF、MongoDB、ADRF、NWDAF-C、PyMTLF-C；
- optional webconsole setup；
- start order、health、status 與 bounded cleanup。

Exit gate：required core NF 與 analytics/storage service 健康，NRF registration identity
符合本次 manifest，restart 不產生非預期 duplicate state。

### Phase 6 — Path A／B Guest Setup 與 Build

Deliverables：

- `path-a`／`path-b` 由同一 pinned source 各自完成的 guest-local UERANSIM build；
- guest-specific gtp5g build；
- `common.sh`／`path.sh A|B` 與 role-local source/config placement；
- UPF-A/B、gNB-A/B、six UEs；
- NWDAF-A/B、PyAnLF-A/B、PyMTLF-A/B；
- TAI-aware UPF selection 與 subscriber/group test data provisioning。
- 三台 VM clock-skew check、`observe.sh` compact status 與可過濾的跨 VM journald live
  follow。

Exit gate：

- six UEs 都建立 current-run AMF／SMF registration；
- TAI `000001` UE 只取得 Path A pool，TAI `000002` UE 只取得 Path B pool；
- two UPFs 形成正確 PFCP association，兩條 N3/N6 path 可被獨立觀察；
- A／B／C NWDAF role 與 scope discovery 正確。

### Phase 7 — Full Scenario E2E

Deliverables：

- 在新 repository 內實作最小 VM-aware full-core analytics／FL E2E；可參考
  `nwdaf-resources` assertion，但不把該 repo 變成 runtime dependency；
- 由單一 Core consumer 透過 NRF 找出兩個不同的 Events Subscription providers，再對
  NWDAF-A/B 建立同 group、不同 TAI 的 `UE_COMMUNICATION` subscriptions，保存各 path
  subscription start，並以兩側都成功作為共同 action barrier；
- stable A/B baseline、A-only degradation、automatic FL trigger；
- two-round FedAvg、validation、ADRF publication、reprovision 與 monitor cutover；
- 驗證成功／失敗條件；per-run record 與 archive format 留待後續設計。

Exit gate：既有 Phase 7 business assertions 全部通過，summary 同時包含 VM/process
identity、source revision、config hash、binary hash、sample count、model identity、ADRF
transaction 與 monitor route；不得把 PseudoDriver 宣稱為真實 application benchmark。

### Phase 8 — Public Release Readiness

Deliverables：

- 從全新 checkout 執行的 quick start；
- prerequisite／resource sizing／troubleshooting／architecture 文件；
- `testbed.local.example.yaml`，不含實驗室 IP、SSH key、private path 或 production secret；
- CI smoke、issue template、contribution policy、versioned release manifest；
- certificate／TLS／OAuth 明確列為第一版 non-goal。

Exit gate：非原開發機使用者能依公開文件完成至少 core smoke 與一個 bounded scenario；
release artifact 與 component commits 可追溯。

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
| Host | 此機器能否安全承載？ | RAM、swap、disk、provider、kernel、port、interface preflight |
| VM | 三個 role 能否重建？ | one-project Vagrant identity、idempotent guest setup、network reachability、restart |
| Core | 真實 5GC control path 是否成立？ | NRF registration、auth、registration、policy、PDU Session |
| Path | TAI 是否選到正確 UPF？ | A/B UE pool、PFCP、N3/N6、serving-SMF evidence |
| Observation | 進行中能否定位目前狀態與錯誤？ | compact health、clock skew、prefixed/filterable live journal |
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
| `5G_NWDAF_Infrastructure` | 新增 integration、submodule、component config、testbed definition、multi-machine Vagrant 與 host/guest scripts |
| `nwdaf-resources` | 不作 submodule/runtime dependency；只有後續確認要去除已移植重複工具時才另行修改 |
| 各 component repo | 只在確實發現 component-owned 缺口時另開 branch 修改 |
| 舊 `5G_Infrastructure` | 預設唯讀；只作 inventory／migration source，不直接清理或重構 |

不得把跨 repository 的變更混成單一 commit。每個 component change 先在自己的 branch
驗證，再更新 integration repo 的 gitlink。

## 14. 尚待決策

進入 Phase 1 前至少確認：

1. GitHub owner／organization、remote URL 與初始 public/private visibility；
2. repository 名稱是否正式採用 `5G_NWDAF_Infrastructure`；
3. open-source license 與第三方 component/license compatibility；
4. 第一個 Vagrant provider，以及此 host 的 VirtualBox 修復或 libvirt feasibility；
5. 舊 VM 採 raw directory copy、修復 provider 後 export 或選擇性 guest data backup，
   以及外部備份位置與保留期限；
6. guest source 採 provider shared folder、受控 source snapshot upload 或其他同步方式；
7. webconsole 是否列入 v0.1 release gate，或只先固定 submodule／文件；
8. public default 是否允許 outbound NAT，以及 scenario data network 的隔離政策；
9. 哪些 `nwdaf-resources` 小工具確實屬於 infrastructure ownership，應移植、重寫或保留
   在原 repo。

這些決策不影響先完成 repository skeleton 的大部分文件，但會影響 host mutation、
provider dependency、release scope 或 public security expectation，因此不能靜默假設。

## 15. 下一個待確認的實作範圍

若本計畫獲確認，下一個工作包先執行 Phase 0.5 的唯讀 inventory 與 backup／removal
proposal，不立即刪除 VM：

1. 解析 Vagrant 與 provider 中所有現存 VM 的 exact identity／storage；
2. 盤點 guest-only 高價值資產與 provider 目前是否能 export；
3. 提出 backup 方法、所需外部空間、recovery 說明與 exact removal list；
4. 由使用者確認 backup 與逐台 removal targets；
5. 另開執行步驟清理並驗證 reclaimed space，之後才進入 Phase 1 repository bootstrap。

VM destroy／unregister/delete、Phase 1 repository bootstrap、Phase 2 submodule pinning、
工具移植、provider 安裝、新 VM 建立與剩餘 legacy cleanup 都是後續獨立授權範圍；新
repository 不以加入 `nwdaf-resources` submodule 作為其中一步。
