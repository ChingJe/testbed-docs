# Fresh User Full-core Black-box Acceptance

日期：2026-08-13

## 結論

一個不繼承既有對話脈絡、且被限制只能閱讀 `5G_NWDAF_Infrastructure` repository 的 subagent，
成功只依新版 README、`docs/`、Makefile 與 repository 內程式，從完全不存在的三台 VM 與空白
runtime state 建立環境並完成一輪 `full-core-cat-transition` business E2E。

六個 UE registration/PDU Sessions、兩個 distinct NWDAF subscriptions、雙路 analytics callbacks、
Path A degradation、A/B local training、C FedAvg、ADRF publication、A/B reprovision、generation
cutover 與 post-cutover evaluated accuracy 全部取得明確 evidence。正常 `experiment-stop` 與
`vm-halt` 亦完成。

本次驗收同時發現觀測介面落差，以及一條 internal Model Monitor cleanup 在 shutdown 前收到
`503` 後缺乏成功 retry evidence。Consumer 的兩份 subscription 已確定刪除；第二條 internal
resource 是否留在 server side 則無法由現有 evidence 確認，因此 E2E business closure 通過，
teardown cleanup 不列為完全通過。

## Black-box boundary

驗收前刻意清除：

- `core`、`path-a`、`path-b` 三台 Vagrant VM；
- production Compose project 的五個 stopped containers、五個 volumes 與 network；
- `5g-nwdaf-infrastructure/pyanlf:local` 與 `pymtlf:local` image tags；
- `config/local`、`config/generated` 的 ignored sets；
- `.generated` datasets 與 tool artifacts。

保留 repository source、commits、16 個 initialized/pinned submodules、committed `config/default`、
fixtures、Ubuntu Vagrant box、共享 Docker build cache，以及 Host VirtualBox、Docker、NVIDIA/CDI 與
network settings。清除後三台 VM 均為 `not_created`，production Docker project 不存在，parent
working tree clean。

subagent 使用 `fork_turns=none`，不得閱讀 `testbed-docs`、`nwdaf-docs` 或其他 workspace
repository，不得修改 tracked file、submodule revision/branch 或降低 resource gate，也不得把
`fl-closure-smoke` 代替 full-core acceptance。遇到問題只能使用 repository 現行 commands、docs、
logs 與 read-only source diagnosis。

## User-visible workflow

subagent 自行從 root README 推導並使用：

```sh
make config-create NAME=blackbox-full-core-20260813 \
  FROM=full-core-cat-transition DEVICE=gpu WEBCONSOLE=false
make dataset-generate CONFIG_DIR=config/local/blackbox-full-core-20260813
make dataset-validate CONFIG_DIR=config/local/blackbox-full-core-20260813
make dataset-show CONFIG_DIR=config/local/blackbox-full-core-20260813
make config-validate CONFIG_DIR=config/local/blackbox-full-core-20260813
make experiment-validate CONFIG_DIR=config/local/blackbox-full-core-20260813
make vm-status
make vm-up
make experiment-start CONFIG_DIR=config/local/blackbox-full-core-20260813
make experiment-status CONFIG_DIR=config/local/blackbox-full-core-20260813
make services-status
make subscriptions-status
make experiment-stop
make vm-halt
```

驗收過程沒有修改 tracked files、呼叫被隱藏的 internal Make targets、縮短 scenario timing 或採用
未記載的破壞性 workaround。dataset tool 第一次執行曾受到 agent sandbox Go-cache permission
限制；在正常 Host 權限下重跑同一公開 Make command 即成功，故不列為 repository defect。

## Timeline

時間為 Asia/Shanghai；component logs 的 UTC 已換算。

| 階段 | 時間 | 結果 |
|---|---|---|
| 清除後基線 | 01:45:32 | parent clean、submodules at pins、三台 VM not_created |
| Config/dataset/preflight | 01:45:45–01:46:01 | 成功，只有 low-swap warning |
| Fresh VM create/provision | 01:46:18–01:57:13 | 10m55s，三台 running |
| Aggregate experiment start | 02:00:48–02:03:47 | 2m59s，無 rollback |
| Path A degradation trigger | 02:23:32 | `evaluated=True triggered=True` |
| FL training/publication/cutover | 02:23:32–02:23:36 | 約 4 秒 |
| Post-cutover acceptance | 02:25:06 | new-generation accuracy `evaluated=True` |
| Aggregate stop | 02:29:56–02:31:49 | Consumer resources deleted；internal cleanup 見下節 |
| VM halt | 02:31:49–02:32:19 | 三台 graceful poweroff |
| Final audit | 02:34:38 | Git clean、containers exited、retained state intact |

完整使用者流程從 config 準備至 VM halt 約 47 分鐘。主要等待來自 business scenario 的 900 秒
breaking time 與 1170/1290 秒 trigger boundary，而不是 provisioning 或 FL calculation。

## Runtime identity

- Config validation hash：`bc5c6aaf…40768`
- Effective config hash：`fb6f6988…a8280`
- Dataset set：`c3b428ea…6bb49`
- Dataset rows：Path A/B 各 32,580
- FL process：`95f61a79-5231-4a89-92a5-f6d25537ef90`
- Published federated model：`1786559016077`

## 5GC and subscription evidence

三台 VM running 後，23 個 Guest experiment units 全部 active。

- UE1–3 journal 均包含 `Initial Registration is successful` 與
  `PDU Session establishment is successful PSI[1]`，取得 `10.60.0.1`–`10.60.0.3`、TAC 1。
- UE4–6 journal同樣成功，取得 `10.61.0.1`–`10.61.0.3`、TAC 2。
- Consumer 經 NRF 選到兩個不同 provider：
  - A：NF instance `11111111-1111-4111-8111-111111111111`、TAC `000001`、API root
    `192.168.57.41`；
  - B：NF instance `22222222-2222-4222-8222-222222222222`、TAC `000002`、API root
    `192.168.57.51`。
- 兩個 subscription `Location` 不同，notification count 持續增加；相鄰 status snapshots 分別顯示
  A 與 B correlation，證明雙路 callback。
- NWDAF-C 建立兩個不同 correlation 的 Model Monitor registrations，兩路 accuracy descriptors
  持續回應 `204`。

## Full-core FL closure evidence

02:23:32 Path A accuracy policy 記錄 `evaluated=True triggered=True`；C 建立單一 process，包含 A/B
兩個 scopes 與 participants。A/B preparation records 分別為 117、114。

A/B 均完成 round 0 與 round 1 local training，每端每輪使用 6 samples，final validation各 1
sample。C 依 training sample count 聚合，本輪 A/B FedAvg 權重各為 `6/12 = 0.5`。兩輪 aggregate
均成功。

Final validation 將 base WAPE `1.8392504554` 降至 candidate WAPE `0.4306710779`，
`gate_would_accept=True`。C 將 model `1786559016077` 發布至 ADRF，`required_scopes=2`。

PyAnLF-A/B 均收到 Model Provision notification、回應 `204`，記錄 model committed、generation 1
與各自 exact analytics subscription activated。C 依序記錄兩個 scopes adopted，接著記錄
`Federated model cutover complete`。

Post-cutover accuracy 第一筆於 02:24:36 因 window 未成熟為 `evaluated=False`；02:25:06 與
02:26:36 均使用新 generation 得到 `evaluated=True triggered=False`，完成 business acceptance。

## Resource observations

- Available RAM：起始約 27,265 MiB；VM 建立後 preflight 約 16,589 MiB；active 最終 snapshot
  約 14,259 MiB。
- 五個 ML containers active RSS 約 2.32 GiB。
- Workspace free space由約 237 GiB 降至 223 GiB，fresh build觀察差約 14 GiB。
- PyMTLF-A/B 使用 RTX 3080、`cuda:0`、NVIDIA runtime/CDI，CUDA visibility為 true；PyAnLF-A/B
  與 PyMTLF-C 使用 CPU。
- 兩個 ML images 各顯示約 5.42 GB virtual size並共享 layers。
- 唯一 preflight warning 是 free swap約 2 MiB；RAM與storage hard gates均通過且沒有調低。

## Documentation and observability findings

新版 README 的 golden path、config/dataset identity、GPU policy、resource gates、lifecycle boundary、
full-core/smoke用途區分與正常 stop順序足以讓無背景使用者完成E2E。

以下敘述或介面仍不完整：

1. 文件將六 UE registration/PDU Sessions列為 status success signal，但 `services-status` 和
   `experiment-status` 實際只列23個unit active；使用者必須自行查六份UE journal。
2. `scripts/host/logs.sh --source vm --service consumer` 固定查
   `5g-nwdaf@consumer.service`，無法匹配特殊的 `5g-nwdaf-consumer.service`。
3. Subscription status只顯示最後一次 callback correlation，單一 snapshot無法同時證明A/B雙路；
   本次需比較不同時間點。
4. 沒有一個 full-core acceptance watcher/summary；使用者需自行知道 degradation、training、FedAvg、
   publication、provision、cutover和post-cutover accuracy的跨component log signatures。
5. Host輸出使用Asia/Shanghai，component logs主要為UTC；文件沒有提醒時區差異。

## Teardown cleanup finding

`experiment-stop` 對兩筆 Consumer subscriptions執行 exact DELETE並明確成功，callback process亦
停止。預設40秒 asynchronous cleanup grace完整執行，沒有被縮短。

PyMTLF-C 清理cutover後的兩條 internal Model Monitor registrations時：

- registration `dc9…` 首次remote DELETE回`503`，1秒後retry得到`404`並記錄removed；
- registration `48cc…` 在02:31:00 shutdown時remote DELETE回`503`，隨即application shutdown，
  沒有後續retry或removed evidence。

第二個`503`不代表resource一定仍存在，但現有流程沒有在停止前證明cleanup收斂，也沒有在40秒
grace結束後作verification。因此應將它視為potential retained server-side Model Monitor resource，
而非已確認成功清理。這是teardown orchestration/runtime contract的新問題；不影響本次business
closure，但需在後續修正或驗證。

### Follow-up orchestration adjustment

同日後續調整只修改`5G_NWDAF_Infrastructure`，未修改PyAnLF、PyMTLF或其他submodule。
`experiment-stop`不再固定等待40秒：停止前先從PyMTLF-C既有`active`／`removed` logs重建目前
active的internal Model Monitor subscription IDs，接著以單一持續log follower涵蓋Consumer
DELETE後的新事件；所有預期IDs都出現`removed`時立即繼續關閉。預設最長等待210秒，逾時或
log follower失效會列出未完成IDs並warning後繼續teardown，避免持續`503`永久阻塞。

這項機制刻意不拿PyAnLF public registration ID與PyMTLF backend ID互相比對，因NWDAF gateway
會使用不同resource identity。Parser、active-state reconstruction與雙ID convergence wait已加入
既有repository test，完整`make test`通過；production runtime行為需在下一次實驗stop再次驗證。

## Final state

- VM：Core、Path A、Path B全部`poweroff`，未destroy。
- ML：五個containers全部`Exited (143)`；五個named volumes、Compose network與兩個images保留。
- Consumer：兩筆exact resources已刪除，callback stopped。
- Internal Model Monitor：一條明確removed；另一條結果不確定。
- Git：parent clean；16個submodules pins/branches未變，無dirty marker。
- Ignored local config、generated dataset、VM/runtime與persistent experiment state依現行文件保留。
