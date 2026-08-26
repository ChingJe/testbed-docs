# Production Flat Config Migration Runtime Acceptance — 2026-08-26

## 結果

Phase 1 已在 pinned 新版 NWDAF／PyMTLF 上完成 configuration implementation、host／container checks
與既有 three-NWDAF production Flat full-core runtime acceptance。A／B 仍由 Consumer／Model
Provision／Model Monitor chain 形成兩個 participant scopes，C 完成兩輪 sample-count-weighted FedAvg、
validation、ADRF publication、兩個 scope adoption、generation cutover 與 post-cutover accuracy evidence。

本輪 business result 為 `complete`，FL failure 為 `not-seen`。標準 teardown 也已完成；兩筆 Consumer
resources 為 `deleted`、Model Monitor `active=0`、五個 ML containers 與 23 個 Guest services 均停止。
VM 保持 running，dataset、database、artifact、container、image 與 volume objects 保留。

Phase 1 的 implementation／runtime gates 已通過，後續 review 亦完成並建立 Infrastructure commit
`072748b`，因此 Phase 1 的獨立 commit gate 已關閉。

## Source 與 runtime identity

- Parent repository branch：`feat/r18-hierarchical-federated-learning`
- Parent starting checkpoint／runtime HEAD：`83118837e18ebf49c5fa97cd7f0d52c5af65a7f3`
- Runtime working tree：上述 checkpoint 加上未提交的 Phase 1 config／host／test diff；submodules clean
- Reviewed Phase 1 completion commit：`072748b`
- NWDAF：`6aed268d6528f8be6c729cbd45b59d067e5e80dc`
- PyMTLF：`36166f04320ae70674604659786ba73935371426`
- PyAnLF：`6a4d94ad3cc6f66dac55ea921772d731e4b71371`
- ADRF：`905f0599f68fe389bba14ed56db0ef9abeab5ccd`
- UPF：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- UERANSIM：`2a3ef81f189ca95d5c1996a28ed7af9734f5cfb4`
- gtp5g：`8d723c29fc0de3eeeff3e9a91132838579e8ee1b`
- Config：`config/default`
- Runtime config SHA-256：`0c6856661f55e1657c786a0926670ffbb7f92e20347db37c336c472c689fe3f6`
- Dataset set：`23697bf00ae0560c9f07f8ae451ebb91797943092317aea8cafdb37435c2fd59`
- PyAnLF image ID：`83f9374aa845`
- PyMTLF image ID：`4dbc36ee43a3`
- 三台 Guest 的 rebuilt NWDAF binary SHA-256：
  `975ca47872334e38a7f3936deadc11b6a8331eb4f76bfb203662046b67d8da18`

## Config migration

Phase 1 保留既有 topology 與 autonomous business ownership，只更新新版 schema 所需 contract：

- PyMTLF-A／B：`runtime.mode: federated`、Client engine、
  `training_data.collection_trigger: consumer_subscription`；移除 client-local `epochs`；
- PyMTLF-C：`runtime.mode: federated`、Server engine、`orchestration.mode: flat`、
  `participant_source: monitor_scopes`、degradation trigger enabled、private trigger disabled；
- 原本 intended training effort 移至 Server-owned `client_training.epochs: 18`；
- 沒有加入 static topology、第四個 Client、Branch／Leaf placement 或 private collection。

Renderer、strict config checks、dataset summary、ML status／compose role detection 與 regression tests 同步更新。
Disposable CPU container smoke 另使用 test-only Compose override 與 readiness helper，允許 component artifacts
已準備完成但 standalone test 沒有 NWDAF 時的明確 `503 unavailable`；production health contract 沒有放寬。
Aggregate `ml-start` failure 也新增明確 rollback，避免 Compose `--wait` 失敗後留下 running containers。

## 前置檢查與 scoped reset

`make experiment-validate CONFIG_DIR=config/default` 通過全部 gates。主機有 62 GiB RAM、24 cores 與約
212 GiB workspace free；VirtualBox、Docker overlay2、ports、host bind、NVIDIA runtime／CDI、dataset、
locks 與 Vagrant 都通過。唯一警告是 2 GiB swap 已全滿；啟動前仍有約 18 GiB `MemAvailable`，因此
本次 bounded run 繼續並監看資源。

經使用者明確同意後，reset 只清除本 scenario 的 retained runtime state：399 筆 ADRF data records、
1 筆 model record、1 個 model artifact、1 筆 ADRF NRF URI-list state，以及五個 ML volume 的內容。
VM、container／volume／image objects、dataset 與 subscriber fixtures 均保留；舊
`5G_Infrastructure` 完全未動。

## Stale binary discovery 與修復

第一次 `make experiment-start CONFIG_DIR=config/default` 正確失敗並 rollback，原因不是 pinned source，
而是三台 Guest 內安裝的 NWDAF binaries 仍為 2026-08-14 舊 artifact，SHA-256 為
`0e298928...`。Guest synced source 已與 Host pinned revision 相符，但舊 binary 的
`/internal/v1/nwdaf-context` 缺少新版 `processInstanceId` 與 `mlAnalyticsCapabilities`，因此 PyMTLF
federated readiness 拒絕啟動。

本輪只在 Core／Path A／Path B 由已同步的 pinned NWDAF source targeted rebuild NWDAF binary；沒有
重建其他 NF、RAN、kernel 或 VM。三台新 binary 均為
`975ca47872334e38a7f3936deadc11b6a8331eb4f76bfb203662046b67d8da18`。第二次 scoped reset 清掉
失敗啟動產生的 partial seed state 後，aggregate startup 完整成功。

## 啟動與資料流

- 23／23 Guest services active；
- UE1–UE6 的 Registration 與 PDU Session 全部 successful；
- 5／5 ML containers healthy；
- PyMTLF-A／B 使用 `cuda:0`、NVIDIA runtime 與 CDI，PyMTLF-C 使用 CPU；
- A/B readiness 各自驗證 only Client capability，C readiness 驗證 only Server capability；
- Consumer 建立兩個不同 provider／TAC／Location 的 subscriptions：
  - A：`619eb047-1ce4-44c9-9e79-da8d8fa0d2a6`，TAC `000001`；
  - B：`d9a06f22-a579-4976-9691-37a3b8884452`，TAC `000002`；
- teardown 前共保留 114 筆 callbacks，A／B 各 57 筆。

本 scenario 使用 go-upf PseudoDriver 提供可重現 stimulus。Registration、PDU Session、serving-SMF
resolution、UPF selection 與 Event Exposure control path 是實際 full-core 流程，但結果不是 real
application traffic benchmark。

## Federated learning closure

主要時間線均為 UTC：

| 事件 | 時間 | 證據 |
| --- | --- | --- |
| Coordinator start | 13:47:52.148 | config identity 相符 |
| Model monitors active | 13:48:32 | initial created=2、active=2 |
| Degradation trigger | 14:08:02.415 | `evaluated=True triggered=true` |
| Federated process | 14:08:02.416 | `c699a484-ee36-4b1e-8c1d-d9abb3e41ff2`，scopes=2 |
| Preparation | 14:08:02.644 | participants `111...`、`222...` |
| Rounds 0、1 | 14:08:06.197 | A／B 都完成兩輪與 final validation |
| Validation | 14:08:06.332 | base WAPE `1.9099532573`、candidate WAPE `0.4482478052` |
| Publication | 14:08:06.538 | model `1787753286332`，`CUTOVER_PENDING`，required scopes=2 |
| Adoption | 14:08:06.724 | scopes=2、`complete=True` |
| Cutover | 14:08:06.724 | family `ue-communication-default` |
| Post-cutover accuracy | 14:17:06.851 | `evaluated=true triggered=false` |

兩個 monitor-derived scopes 都使用 Internal Group `00000001-466-92-01`，但 A 綁定 consumer
`11111111-1111-4111-8111-111111111111`／TAC `000001`，B 綁定
`22222222-2222-4222-8222-222222222222`／TAC `000002`。A 每輪使用 7 samples，B 每輪使用 5
samples，證明 Server 收到可供 sample-count-weighted FedAvg 使用的不同 participant weights。

Validation gate 本 scenario 設為觀測而非強制拒絕：`gate_would_accept=True`、`enforced=False`。這不影響
publication／cutover closure，但解讀結果時不可宣稱本輪測了 enforced rejection path。

## 資源與 teardown

Business acceptance 時的 container memory 約為：PyAnLF-A 290.1 MiB、PyAnLF-B 284.6 MiB、
PyMTLF-A 725.1 MiB、PyMTLF-B 713.8 MiB、PyMTLF-C 298.1 MiB。A／B 各使用約 392 MiB GPU
memory；主機當時仍有約 18.4 GiB `MemAvailable`，未觀察到 OOM 或 service failure。

`make experiment-stop` 精確刪除兩個 Consumer resources，等待 40 秒讓 backend cleanup 完成，再停止
containers 與 Guest services。A／B 的 event subscription、最新 generation 的 Model Monitor
subscription 均回 `204`，model runtime 已 unload／release。停止後狀態為：

- Core、Path A、Path B VM：running；
- 23 個 Guest services：inactive；
- 五個 ML containers：exited；
- Consumer：inactive，兩筆 resource 都是 `deleted`；
- Model Monitor：created=4、active=0；
- retained FL result：`complete`，failure `not-seen`。

## 已知非阻塞問題

PyAnLF-A／B 在 startup reconciliation 各遇到四次 Model Provision create `503`，時間為約
13:48:15–22。這段期間 PyMTLF-C health 已回 200，但 NWDAF-C public Model Provision endpoint 尚未能
完成下游 subscription；13:48:30 C 的 internal PyMTLF subscriptions 回 201 後，A／B retry 自行恢復，
並於 13:48:32 完成 monitor registration。後續 training、publication、cutover 與 cleanup 均正常。

這證明目前 generic process／HTTP readiness 早於 Model Provision business readiness 約 15 秒。它不推翻
本輪 bounded acceptance，但後續 phase 應把 provider business readiness 或 startup ordering 做成可觀測
gate，避免以 retry traceback 作為正常啟動路徑。

## Verification

- pinned PyMTLF native config load：A／B／C 全部通過；
- `make test-containers`：通過完整 disposable CPU container lifecycle，並清除 test containers／volumes；
- `make test`：最終完整通過，包括 config contract、execution policy、dataset determinism、Compose checks
  與 `vagrant validate`；
- `make experiment-validate CONFIG_DIR=config/default`：通過，只有已記錄的 swap warning；
- runtime full-core production Flat acceptance 與 standard teardown：通過。

本輪仍使用 FedAvg；FedProx replacement、static Flat 四 Client、static HFL Root／Branches／Leaves 與
controlled topology comparison 都不屬於 Phase 1。
