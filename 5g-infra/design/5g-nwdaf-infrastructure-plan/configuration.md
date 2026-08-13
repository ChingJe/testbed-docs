# 設定檔、Testbed Definition 與 Scenario

[返回主計畫](../5g-nwdaf-infrastructure-plan.md)

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
