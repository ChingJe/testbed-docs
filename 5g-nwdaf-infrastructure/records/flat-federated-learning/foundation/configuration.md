# 設定、Experiment Definition 與 Dataset Contract

[返回基礎建置計畫](plan.md)

## 9. 設定責任邊界

第一版將使用者可調整的輸入分成三層，再產生 component 真正讀取的完整
native config set 與 PseudoDriver dataset：

```text
testbed.yaml
  └─ VM 資源、網路、PLMN／TAI／UE pool、placement、endpoint 與 Host policy

experiments/<definition>/scenario.yaml
  └─ sampling、monitor policy、FL training policy 與時間預算

experiments/<definition>/traffic/*.json
  └─ raw row 間隔、warm start、traffic transition 與 A／B traffic values

config/default + config-create
  └─ config/local/<name>/ 完整 NF／RAN／ML／Consumer config set

config set + experiment definition + testbed
  └─ .generated/datasets/<dataset-set-id>/ Path A／B artifacts
```

每個對使用者開放的欄位都必須在 Infrastructure repository 的正式文件說明型別、
單位、合法值、source of truth、影響的 runtime consumers 與交叉限制。不開放直接修改的
欄位則必須明確標示為 generated／derived，並指出它從哪一個使用者輸入展開。

### 9.1 `testbed.yaml`

Root `testbed.yaml` 是一份完整 topology definition，同時供 Vagrant、config renderer、
dataset resolver 與診斷工具使用。它負責：

- Core、Path A、Path B 的 RAM、vCPU 與 dynamic disk ceiling；
- management、SBI、N2、N3-A／B、N4、N6-A／B networks 與 machine interfaces；
- PLMN、S-NSSAI、DNN、Internal Group、TAI、UE pool 與 subscriber ordinals；
- NF／RAN／MongoDB／ADRF／Consumer 的 placement、bind address、advertised address 與 port；
- Host ML containers 的 CPU／GPU device policy 與 published endpoints；
- optional WebConsole 和 Infrastructure-owned operational timeouts；
- RAM、swap 與 storage 的建議安全門檻。

`466/92`、目前 subnet、TAI、resource values 與 ports 都是 committed reference，不是程式寫死的
實驗室常數。需要另一組 topology 時，複製整份 `testbed.yaml` 到使用者自選路徑，
並在每一個相關 Make command 以 `TESTBED=<path>` 明確選用。不再使用
`testbed.local.yaml` overlay，也不合併兩份不完整 topology。

### 9.2 Experiment definitions

內建參考實驗與使用者實驗分開保存：

```text
experiments/
├── examples/
│   ├── full-core-cat-transition/
│   │   ├── scenario.yaml
│   │   └── traffic/
│   │       ├── path-a.json
│   │       └── path-b.json
│   └── fl-closure-smoke/
│       ├── scenario.yaml
│       └── traffic/
│           ├── path-a.json
│           └── path-b.json
└── local/
    └── .gitkeep
```

- `experiments/examples/` 是 committed、reviewed 且可直接執行的 reference definitions。
- `experiments/local/` 預設 gitignored，供使用者複製後調整。
- `fixtures/` 不保存使用者實驗；程式測試所需的固定輸入放在 `tests/fixtures/`。
- generated Parquet 仍只放在 `.generated/datasets/`，不放入 experiment definition。

Scenario schema v2 以 scenario 所在目錄作為 traffic profile path 的相對基準：

```yaml
schemaVersion: 2
name: full-core-cat-transition
kind: business-acceptance
warmStartMode: inference-only
samplingIntervalSeconds: 30
trafficProfiles:
  a: traffic/path-a.json
  b: traffic/path-b.json
```

Scenario 與 profile 必須位於 repository 內；resolver 拒絕 absolute path、repository escape
與錯誤的 Path A／B identity。Renderer、checker 與 dataset generator 共用同一個 resolver，
不各自解釋路徑。

### 9.3 `config-create`

`FROM` 只有一種語意：必填的 scenario YAML path。不提供 short name、不自動選擇預設
example，也不保留舊 `fixtures/full-core` 相容層。

```sh
make config-create \
  NAME=my-experiment \
  FROM=experiments/examples/full-core-cat-transition/scenario.yaml \
  DEVICE=cpu \
  WEBCONSOLE=false
```

自訂實驗使用相同入口：

```sh
make config-create \
  NAME=my-experiment \
  FROM=experiments/local/my-experiment/scenario.yaml \
  DEVICE=gpu
```

- `NAME` 只命名 `config/local/<NAME>/` 輸出。
- `FROM` 選擇 experiment definition。
- Scenario 內的 `name` 是實驗契約名稱，也是 reset confirmation identity。
- `DEVICE` 是明確 `cpu` 或 `gpu`，runtime 不做 silent fallback。
- `WEBCONSOLE` 決定 aggregate start 是否包含 optional WebConsole。

Renderer 以 `config/default/` 作為 native baseline，把 testbed 與 scenario-owned 欄位展開到
`config/local/<NAME>/`，並寫入 manifest、config identity、scenario definition/hash、profile provenance
和 generated file list。它不覆寫既有 output，除非使用者明確要求可覆蓋行為。

### 9.4 Native config ownership

`config/default/` 與 `config/local/<NAME>/` 都保存 component 真正讀取的完整 native config，
包含 core NF、UPF-A/B、UERANSIM、NWDAF-A/B/C、ADRF、PyAnLF、PyMTLF、Consumer、
WebConsole、subscriber fixtures 與 role-specific network aliases。

文件必須為每份 native config 列出：

- 屬於哪個 component，在哪台 VM 或 container 使用；
- 哪些欄位由 `testbed.yaml` 產生；
- 哪些欄位由 scenario 產生；
- 哪些 component-native 欄位可在 local set 進階調整；
- 哪些欄位不能單獨修改，否則會與其他 component 不一致。

例如 `scenario.samplingIntervalSeconds` 會同時展開為：

```text
smfcfg.yaml       urrPeriod
upfcfg-a/b.yaml   ees.periodSec
consumer.yaml     reporting.periodSeconds
pyanlf-a/b.yaml   sampling and ground-truth check intervals
```

使用者可調整 local native config，但 validation 會如實列出它和 canonical input 的差異；
該差異不自動禁止實驗啟動。

### 9.5 診斷與執行分離

`config-validate`、`dataset-validate` 與 `experiment-validate` 是唯讀診斷工具。它們可回報並以
non-zero exit status 讓 CI 辨識：

- topology／native config／scenario 不一致；
- sampling、monitor、warm-start、training 與 closure capacity 風險；
- component lock、dirty worktree、port、provider 或 Host resource 問題；
- dataset 實際內容與契約不符。

這些診斷不是 runtime 啟動授權。`experiment-start`、`services-start`、`ml-start`、
`webconsole-start` 與 subscriber apply 不先執行完整 validation gate，也不因設定只是
不一致、不建議或資源低於 reference threshold 就拒絕執行。

執行動作仍在以下實際無法繼續的邊界停止：

- 必要檔案不存在或 YAML／JSON 無法解析；
- VM、Docker、bind address、GPU 或 service 實際無法使用；
- lifecycle 已在運行而容易造成重複資源；
- dataset 檔案缺少、hash 錯誤或 staging 不完整；
- process／container／subscription 實際啟動失敗；
- reset 等破壞性操作無法確定 exact scope。

其中 dataset resolution 再分成：

- **structural requirements**：可否解析並生成 deterministic artifact；失敗時
  `dataset-generate` 無法繼續。
- **advisory diagnostics**：warm start、training sample、stable lead、degraded tail、trigger
  與 closure budget 是否符合 reference experiment；不阻擋 dataset generation 或 start。

### 9.6 Dataset 語彙與顯示

Infrastructure 文件與 `dataset-show` 必須分開以下層級：

```text
Traffic profile
  ↓ generator
Parquet rows（raw interval，目前 1 秒）
  ↓ PseudoDriver 依 subscription period 聚合
Usage observations（目前 30 秒）
  ↓ PyAnLF prediction／ground-truth matching
Accuracy reports（目前 90 秒）
  ↓ PyMTLF policy／dataset builder
Training and validation samples
```

`windowSeconds=1` 只是 Parquet raw row 的時間解析度。若 UPF Event Exposure subscription
period 是 30 秒，PseudoDriver 把同一 UE 在 dataset timestamp `0..29`、`30..59` 的 UL／DL
bytes 分別加總成 30 秒 UsageMeasures；若 period 是 5 秒，則以五個 1 秒 rows 為一組。

現有 traffic profile 欄位名稱暫不改 schema，但對外顯示必須使用準確語意：

- `breakingTimeSeconds`：warm-start／live replay boundary，不是 degradation time；
- `stableWindows * windowSeconds`：traffic transition timestamp；
- `degradedWindows * windowSeconds`：post-transition duration；
- `postBoundaryMode`：決定 Path A 改為 degraded 或 Path B 繼續 stable。

`dataset-show` 預設輸出人類可讀的 dataset set identity、Path A／B rows／UEs／hash，
以及 raw interval、warm-start boundary、traffic transition、sampling、monitor report、earliest／
bounded trigger 與 closure budget。Manifest 仍保存在 artifact 目錄供程式與進階查詢使用。

### 9.7 Reference scenarios

| Setting | `full-core-cat-transition` | `fl-closure-smoke` |
| --- | ---: | ---: |
| 用途 | 主 example／business acceptance | 快速驗證同一 FL control/data path |
| warm-start responsibility | inference-only | inference-and-training |
| raw row interval | 1 秒 | 1 秒 |
| UPF／PyAnLF sampling | 30 秒 | 30 秒 |
| historical warm-start | 900 秒／30 observations | 3000 秒／100 observations |
| stable live lead-in | 900 秒 | 360 秒 |
| Path A transition | dataset `t=1800s` | dataset `t=3360s` |
| Path B | 全程 stable control | 全程 stable control |
| accuracy report period | 90 秒 | 90 秒 |
| report capacity／minimum match | 3／2 | 3／2 |
| minimum references | 5 | 2 |
| decision window／required hits | 5／3 | 2／1 |
| fitting rounds／local epochs | 2／18 | 2／2 |
| closure budget | 1800 秒 | 300 秒 |

主 example 的 900 秒 Phase 1 只負責填滿 30-step PyAnLF input window。實時回放從 dataset
`t=900s` 開始，Path A 在 `t=1800s` 才發生 traffic transition，因此有 900 秒 stable
live lead-in。最快再經過 `3 * 90 = 270` 秒 degradation decisions，所以 earliest trigger
是 live replay 後 1170 秒；1290 秒 bounded trigger 另保留一個 report period 與 sampling boundary。

以 `seq_length=30`、`out_seq_len=1` 和 `validation_ratio=0.1` 計算，69 observations 約形成
8 個 training samples 與 1 個 validation sample。`minNumSamples=1` 指 dataset builder 完成後的
training sample count，不是 raw rows、UPF notifications、observations、batch size 或 retrieval 上限。

Smoke 用 3000 秒 historical burst 同時準備 inference 與 training history，並降低 reference、hit
與 local epoch 以縮短等待；它仍走正常 WAPE trigger、ADRF retrieval、A／B local training、
FedAvg、publication、reprovision 與 cutover，但不代表主 example 的 production timing 或長時間
traffic transition acceptance。

### 9.8 文件交付

Infrastructure repository 最終將 `docs/configuration.md` 作為設定入口，並拆分：

```text
docs/configuration/
├── testbed-reference.md
├── scenario-reference.md
├── traffic-profile-reference.md
├── native-config-reference.md
└── dataset-reference.md
```

每份 reference 使用同一格式說明 field path、意義、單位、合法值、source of truth、
rendered consumers、交叉限制與修改範例。Root README、commands、operations 和 troubleshooting
同步改為新 `experiments/` 路徑與 non-gating validation 語意，不殘留舊 `fixtures/`、short-name
`FROM` 或「start 必須先通過 validation」的說明。

### 9.9 第一版不管理 experiment history

VM lifecycle 與單次 experiment 分開：三台 VM 可以 `up`／`start` 一次，再連續執行多次
實驗。第一版不定義 run-id、`runs/`、自動 log collection 或 archive schema。Log 與 service
state 留在各自 execution domain，必要時以 `experiment-status`、`make logs`、Guest journal 或
Docker logs 查看。等跨多次 experiment 的 identity、collection 與保存需求確定後，再獨立設計。
