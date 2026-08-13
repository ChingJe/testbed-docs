# 實作紀錄：基礎建置與 E2E

[返回主計畫](../5g-nwdaf-infrastructure-plan.md)

本文件保存原計畫第 16.1–16.21 節的本機實作紀錄。

## 16.1 2026-08-06 Infrastructure baseline

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

## 16.2 2026-08-09 GPU capability 與 deployment 調整

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

## 16.3 2026-08-09 PseudoDriver dataset RAM audit

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

## 16.4 2026-08-09 Hybrid topology/config freeze

Infrastructure commit：`9e24cc5`。對應 Host 清點見
[hybrid-host-readiness-inventory-2026-08-09.md](../../reports/5g-nwdaf-infrastructure/hybrid-host-readiness-inventory-2026-08-09.md)。

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

## 16.5 2026-08-09 Host ML image 與 CPU smoke

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
[host-ml-container-cpu-smoke-2026-08-09.md](../../reports/5g-nwdaf-infrastructure/host-ml-container-cpu-smoke-2026-08-09.md)。

這次沒有安裝 NVIDIA Container Toolkit、沒有讓 container 存取 GPU、沒有建立 VM 或 Host
network，也沒有驗證 VM-to-Host published endpoints。下一個安全工作是實作日常
`ml-start`／`ml-status`／`ml-stop` 及 observe/log integration；GPU 與 network mutation 仍需
另行授權。

## 16.6 2026-08-09 Host ML lifecycle 與觀測

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
[host-ml-lifecycle-smoke-2026-08-09.md](../../reports/5g-nwdaf-infrastructure/host-ml-lifecycle-smoke-2026-08-09.md)。

PyAnLF 啟動同時回報 callback ingestion default 的高理論 memory bound（8192 entries ×
4 MiB request ceiling）。Queue 不會在 startup 預先配置，且 container 具有 768 MiB hard
limit，但正式 callback burst 仍可能造成 container-local OOM 或 drop-oldest。這次不在缺乏
traffic peak evidence 時改變 queue semantics；GPU/full-stack smoke 前須量測實際 payload、
queue depth、drop counter 與 peak RSS，再決定 capacity／request limit 或提高 container RAM。

## 16.7 2026-08-09 Shared Docker GPU activation 初始決策（已由 16.9 修正）

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

## 16.8 2026-08-09 Native CDI deployment definition（已由 16.9 取代）

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

## 16.9 2026-08-09 NVIDIA runtime CDI activation

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

## 16.10 2026-08-09 Guest baseline 改為 Ubuntu 22.04

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

## 16.11 2026-08-09 Core VM skeleton smoke

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

## 16.12 2026-08-09 Path VM skeleton 與三 VM network smoke

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

## 16.13 2026-08-09 Process alias 改由 topology 產生

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

## 16.14 2026-08-09 Guest service lifecycle 與 six-UE registration

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
[guest-services-and-ue-registration-smoke-2026-08-09.md](../../reports/5g-nwdaf-infrastructure/guest-services-and-ue-registration-smoke-2026-08-09.md)。

這完成 Phase 6 的 process、registration 與 PDU Session bring-up 子集，不宣稱 N6 traffic、
PseudoDriver replay／Event Exposure、Host ML containers 或 subscription E2E 已通過。

## 16.15 2026-08-09 Host ML 與 Guest Stack 整合 Smoke

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
[host-ml-guest-stack-integration-smoke-2026-08-09.md](../../reports/5g-nwdaf-infrastructure/host-ml-guest-stack-integration-smoke-2026-08-09.md)。

## 16.16 2026-08-09 Generated PseudoDriver Dataset Tooling

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
[Generated PseudoDriver Dataset Tooling](../../reports/5g-nwdaf-infrastructure/generated-pseudodriver-dataset-tooling-2026-08-09.md)。

## 16.17 2026-08-09 PseudoDriver Dataset Guest Staging Smoke

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
[PseudoDriver Dataset Guest Staging Smoke](../../reports/5g-nwdaf-infrastructure/pseudodriver-dataset-guest-staging-smoke-2026-08-09.md)。

## 16.18 2026-08-09 Nupf Contract 與 PseudoDriver Runtime Smoke

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
[Nupf Contract 與 PseudoDriver Runtime Smoke](../../reports/5g-nwdaf-infrastructure/nupf-contract-pseudodriver-runtime-smoke-2026-08-09.md)。

## 16.19 2026-08-09 雙 E2E Scenario 與 Training Data 決策

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

## 16.20 2026-08-10 Stateless Backend Lifecycle 遷移

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

## 16.21 2026-08-10 Stateless Full E2E 與 ADRF-only PyAnLF Policy

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
[Stateless Full E2E Smoke](../../reports/5g-nwdaf-infrastructure/stateless-full-e2e-smoke-2026-08-10.md)。

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
