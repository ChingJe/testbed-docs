# Runtime Helper Sync 與 FL Lifecycle 回歸（2026-08-12）

## 結論

本輪從三台既有 VM poweroff 與 scoped clean state 起跑，驗證 Host checkout 的 runtime
helpers 可在不重新 provision VM 的情況下安全同步到三台 Guest。同步後以 GPU 版
`fl-closure-smoke` 完成 23 個 Guest units、五個 Host ML containers、兩筆 analytics
subscriptions、A-only degradation、兩輪 federated training、ADRF publication、A／B
reprovision 與 monitor generation cutover。第一個 closure 及 post-cutover accuracy route
均通過。

本輪同時確認 `experiment-start` 是持續運行的通用 lifecycle，不是「首次 closure 後自動
停止」的 bounded runner。持續的 Path A degradation 在約四分半後再次觸發第二個 FL
process，符合實驗保持 active 時持續監測與再次訓練的語意，不是 lifecycle 問題。第二個
candidate 品質較差，但 smoke 明確關閉 deployment policy enforcement，因此仍被發布。
原始 teardown 在 PyMTLF-C 留下一次 monitor subscription DELETE `503`，暴露五個 ML
containers 同時停止時的短暫下游不可用。後續 active-runtime regression 證明只調整 container
停止順序無效；Infrastructure 加入 consumer exact DELETE 後的固定 40 秒 grace，再次實測時
所有相關 cleanup 均在 container shutdown 前回覆 `204`，沒有重現 `503`。

本輪沒有修改 NF／ML source 或 component revision。PseudoDriver 只提供可重現的 traffic
stimulus，本結果不代表真實 application traffic benchmark，也不取代
`full-core-cat-transition` 的 business timing 驗收。

## Runtime identity

- Infrastructure：`1f2307b010b4dacb38f6267dc991c781f94a64c1`
- Guest runtime helper bundle SHA-256：
  `765dd1f50eed33d5aefa6f5b30cb35bf82b25411f2b01ccd146f9d5261099d16`
- NWDAF：`318ac19d8b027373f4468660394da1ec3338268e`
- PyAnLF：`08798f15c3693027e00bc60dd53f74ebaa26c3a1`
- PyMTLF：`7e8ab7f23bf5d6398eb1cd5f053dd8bda9439a87`
- ADRF：`905f0599f68fe389bba14ed56db0ef9abeab5ccd`
- SMF：`128b0ec6157238efe4203e2060415728599ada04`
- go-upf：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- UERANSIM：`2a3ef81f189ca95d5c1996a28ed7af9734f5cfb4`
- gtp5g：`8d723c29fc0de3eeeff3e9a91132838579e8ee1b`
- Scenario：`fl-closure-smoke`（GPU）
- Manifest config SHA-256：
  `2f2f700e4617174a44b767315a7c96a436e47d880d1110cbb4369de2ecea38b6`
- Effective service／Compose config：`fl-closure-smoke:edd6dceed361`
- Dataset：`2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`
- VM OS／provider：Ubuntu 22.04／VirtualBox

Repository test 的 shell、Python、negative config contract、Netplan renderer、deterministic
dataset、Compose contract 與 Vagrant validate 全部通過。Runtime preflight 為零 failure；兩個
warning 是 Host free swap 為零，以及 `provider.expectedVmStorage` 尚未固定。起跑時 Host 約
有 24 GiB available RAM、219 GiB filesystem free；RTX 3080 為 10 GiB，啟動前只使用
21 MiB VRAM。

## Existing VM helper sync

三台 VM 以 `vagrant up --no-provision` cold boot，Vagrant rsync 完成後由 Host 建立只包含
runtime helpers、Consumer、subscriber projection 與 systemd definitions 的 archive。每台
Guest 先驗證 archive SHA-256，再經共用 installer 安裝至 `/usr/local/libexec` 與
`/etc/systemd/system`，只執行 `daemon-reload`，不啟動或重啟 process。

Core、Path A、Path B 最終記錄相同 bundle identity；三台的 `config-activate` 與
`subscriber-data.js` 逐檔 SHA-256 也一致。同步完成後 stack target 與 consumer 都維持
inactive。後續 config activation、subscriber fixture apply 與 service startup 實際使用已安裝
helper，因此本輪同時驗證「既有 VM 不 reprovision 也不沿用 stale helper」。

Config staging 後，Core 14 個、Path A／B 各 7 個 aliases 都直接通過 `NETWORK VERIFY`，沒有
執行 reconcile。兩台 Path 的 gtp5g vermagic 均對應 running kernel `5.15.0-187-generic`。

## Scoped clean state 與 startup

reset 前保留 111 筆 ADRF data、1 筆 model record、1 個 model artifact，以及五個 ML state
volumes 內容。confirmation-gated reset 只清除上述內容與 NRF ADRF URI residue；containers、
images、network、named volumes 與 VM 都保留。reset verify 顯示 ADRF collections、model
storage、NRF ADRF state 與五個 volume contents 全部為空。

後續 23 個 Guest units 全部 active，五個 ML containers 全部 healthy。PyMTLF-A／B 使用
`cuda:0`、NVIDIA runtime 與 `nvidia.com/gpu=all` CDI selector；PyAnLF-A／B 與 PyMTLF-C
使用 CPU。單一 consumer 於 `08:35:40Z` 經 NRF 只建立兩筆 subscriptions：

| Path | NF instance | TAC | Subscription |
| --- | --- | --- | --- |
| A | `11111111-1111-4111-8111-111111111111` | `000001` | `65c7554c-1bfb-496d-ae85-ff632917af6a` |
| B | `22222222-2222-4222-8222-222222222222` | `000002` | `a7190cce-5f10-43ed-b4d8-6ee6dcd9e947` |

A／B 都成功從 C provision seed model `1`。初始 model activation 各有一次預期的 stale
revision terminal cancellation，沒有 retry storm。第一個 stable report 為 `matched=2`，後續
為 3；A deviation 約 `2.4305`，B 約 `2.1991`，C 均正常評估且不觸發。

## 第一個 FL closure

`08:43:42Z` 的 report 首次涵蓋 Path A degradation：

- A：model `1`、`matched=3`、deviation `5.9568`，C 判定 triggered；
- B：model `1`、`matched=3`、deviation `2.1992`，同輪未觸發另一 process；
- trigger 距兩筆 subscriptions 約 481 秒。

C 建立 process `a06d7673-b7b3-4026-ad54-50814b832543`。A／B 都從 ADRF 取得 45 筆
records；兩輪 local training 各使用 49 samples，final validation 各使用 5 samples。C 於
`08:43:45Z` 依序完成 round 0、round 1 FedAvg，final validation 為：

- base WAPE：`1.8397703958`；
- candidate WAPE：`0.2463687766`；
- `gate_would_accept=true`；deployment enforcement 為 `false`。

C 發布 model `1786524225676`，publication ID 為
`ca50eae5-1c16-40a6-b3fb-67dd7ba4322b`。A／B 都從 ADRF 下載並 activate 同一 model；C 於
`08:43:46Z` 完成兩個 required scopes adoption 與 cutover，trigger 到 cutover 約四秒。

第一個新 generation report 中，A／B 都為 model `1786524225676`、`matched=2`；C 在
`08:45:16Z` 對兩者均記錄 `evaluated=[True] triggered=[False]`。這證明新 model 已進入
production inference、accuracy delivery 與 monitor route，不只是完成 publication。

## 延長運行與第二次 trigger

`experiment-start` 不會在上述 acceptance point 自動停止。Path A degradation stimulus 繼續
存在，`08:48:16Z` 的下一個 decision window 又觸發 process
`82b113e9-779f-4841-a05a-d9aa50f47bf3`。A／B 各從 ADRF 取得 72 筆 records、兩輪各訓練
53 samples，C 再次完成兩輪 FedAvg。

第二次 final validation 的 base WAPE 為 `0.2466968301`，candidate WAPE 為
`0.4575428550`，因此 `gate_would_accept=false`。但 smoke 的 deployment policy
enforcement 明確設為 `false`，C 仍發布 model `1786524497258`，A／B 也完成 adoption 與
cutover。這是 example config 的既有語意；若測試目標是驗證「品質 gate 阻止 regression」，
必須使用 enforcement=true 的使用者 config，不能把本結果解讀為 production deployment
policy 驗收。

因此第二次 trigger 是持續運行同一實驗的預期結果；實驗何時停止仍由使用者決定，不需要讓
通用 `experiment-start` 在首次 cutover 後自動結束。

## Teardown、保留 state 與後續項目

consumer 最終收到 46 次 notifications。兩個 exact subscription locations 均於
`08:48:38Z` DELETE，consumer 狀態為 stopped 且 `lastError=null`。五個 ML containers、23
個 Guest units 都停止，三台 VM 最後 graceful poweroff；Host 回到約 26 GiB available RAM，
GPU 為 21 MiB idle。本專案沒有 running container。

第一次 FL concurrent closure 時觀察到 aggregate VRAM 約 829 MiB、Host available RAM 約
20.7 GiB，沒有觸發 RAM hard gate 或 OOM。swap 全程為零 free，仍是共用 Host 的風險，不能
以本次短跑通過推論適合長時間運行。

teardown 過程中，PyMTLF-C 於 `08:49:08Z` 記錄一次 monitor subscription DELETE `503`。
時間點在兩筆 consumer subscriptions 已刪除、Compose 正在停止五個 ML containers 期間；
初步判斷是非同步 monitor reconciliation 與 shutdown 相撞。它不影響先前兩次 closure 或
exact consumer DELETE，但代表當時的並行 shutdown 不是 error-free teardown。

後續 Infrastructure commit `a2da93a` 曾把 `ml-stop` 改為先等待 PyMTLF-C 退出，再停止其餘
project containers。Repository tests 與 stopped-container no-op 路徑通過，但 active-runtime
regression 失敗：兩條 Model Monitor subscriptions 於 `12:47:16Z` active；兩筆 consumer
subscriptions 於 `12:47:57Z` exact DELETE 後，C 收到 SIGTERM。C 到 `12:48:27Z` 才執行
pending monitor DELETE，此時 A／B containers 仍到約 `12:48:28Z` 才退出，但 NWDAF-C 已因
MTLF backend unavailable 回覆 `503`。NWDAF-C 隨後執行 backend-loss cleanup，A／B 的部分
deletes 於 `12:48:29Z` 回覆 `204`。

因此根因不是單純「A／B 比 C 早停止」，而是 cleanup 尚未收斂就對任一相關 backend 送出
SIGTERM。Infrastructure 已恢復原本的 project-scoped `ml-stop`，避免保留無效 ordering。

### 固定 cleanup grace 修正與 active-runtime regression

使用者選擇先以 deployment lifecycle 的固定 grace 解決，不新增 ML／NWDAF runtime API。
`experiment-stop` 現在於 consumer exact DELETE 成功後，保持全部 ML containers 與 Guest
services 可用 40 秒，再執行原有 `ml-stop` 與 `services-stop`。等待可由非負整數
`ML_CLEANUP_GRACE_SECONDS` 覆寫；非法值會在查詢或修改 runtime state 前以 exit 2 拒絕。
預設 40 秒高於前次觀察到約 30 秒的 late reconciliation window。

修正後以同一 `fl-closure-smoke` contract 產生 ignored CPU config
`fl-cleanup-grace:06df80cc579d`。使用 CPU 是因本輪 Host `nvidia-smi` 無法連線 driver；測試
目標只涵蓋 resource cleanup 與 teardown ordering，沒有修改提交的 GPU 預設、lock 或任何
NF／PyAnLF／PyMTLF source。Repository tests、config validation 與 CPU runtime preflight 都
通過；起跑時約 22 GiB available RAM、218 GiB filesystem free，低 swap 仍只作既有 warning。

實測證據如下：

- PyMTLF-C 於 `13:08:48Z` 建立 A／B monitor subscriptions
  `9bd59817-642a-4201-9e71-0b4ab9d4b7e6` 與
  `bad7030e-23ed-46d1-9fd0-f85a497af988`；
- `experiment-stop` 於 `13:10:08Z` 開始，A／B consumer subscriptions 於
  `13:10:11Z` exact DELETE 且 consumer `lastError=null`；
- A／B 的 Model Provision subscription、Model Monitor registration，以及 C 持有的兩條
  internal Model Monitor subscription 均於 `13:10:11Z` 回覆 `204`；
- PyMTLF-C 到 `13:10:53Z` 才完成 shutdown，Core NWDAF-C 到 `13:11:16Z` 才收到 stop，
  因此 cleanup 明確發生在第一個 backend unavailable 之前；
- 本次時段沒有 DELETE `503`、ERROR 或 traceback；最終 23 個 Guest units inactive、五個
  containers exited、consumer inactive，三台 VM graceful poweroff。

這個結果驗證目前 testbed 的固定等待方案。它不是 protocol-level convergence proof；若未來
cleanup window 可能超過本地觀察值，可提高環境變數，或另案設計可輪詢的 drain contract。

本輪保留 150 筆 ADRF data records、2 筆 model records、2 個 model artifacts，以及五個
ML state volume contents，供後續查驗。下一次實驗仍應先使用 confirmation-gated scoped
reset；不需要刪除 containers、images、network 或 named volumes。
