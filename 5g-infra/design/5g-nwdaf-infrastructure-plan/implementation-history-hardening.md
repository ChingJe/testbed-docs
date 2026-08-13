# 實作紀錄：Contract Hardening 與文件整理

[返回主計畫](../5g-nwdaf-infrastructure-plan.md)

本文件保存原計畫第 16.22–16.32 節的本機實作紀錄。

## 16.22 Configuration Contract Hardening

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

## 16.23 2026-08-12 Persistent Netplan Alias Ownership 計畫

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
[Persistent Netplan Alias Migration](../../reports/5g-nwdaf-infrastructure/persistent-netplan-alias-migration-2026-08-12.md)。

## 16.24 2026-08-12 FL Closure Smoke 與 Netplan Regression

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
[FL Closure Smoke 與 Persistent Netplan 回歸](../../reports/5g-nwdaf-infrastructure/fl-closure-smoke-netplan-regression-2026-08-12.md)。

## 16.25 2026-08-12 Runtime Helper Sync 與 FL Lifecycle Regression

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

延長運行也確認 `experiment-start` 的持續 lifecycle 語意：它不會在首次 closure 後自動停止；
持續 degradation 約四分半後再次觸發 FL 是預期行為。第二次 candidate WAPE `0.4575` 劣於
base `0.2467`，但 smoke 的 deployment enforcement=false，因此仍依 config 語意發布。實驗
何時停止由使用者決定，不需要為首次 cutover 加入自動停止或 scenario-specific command。

正常 teardown 已精確 DELETE 兩筆 consumer subscriptions，停止五個 containers、23 個
Guest units，並 graceful poweroff 三台 VM；但 PyMTLF-C 在 Compose shutdown 期間留下單次
monitor subscription DELETE `503`。診斷確認原因是五個 ML containers 同時停止，使 C 在
graceful shutdown 刪除 A／B Model Monitor subscriptions 時，下游 PyAnLF 已不可用。

Infrastructure `a2da93a` 曾嘗試先同步等待 PyMTLF-C 退出，再平行停止其餘四個 containers；
repository tests 與 stopped-container no-op 路徑雖通過，但 active-runtime regression 證明此
順序不足。PyMTLF-C 收到 SIGTERM 後，NWDAF-C 已將 MTLF backend 視為 unavailable；C 約
30 秒後才執行 pending monitor DELETE，因此即使 A／B containers 仍存活，NWDAF-C 仍回覆
`503`。A／B 的 asynchronous Model Provision／Monitor registration cleanup 同時也遇到
`503`，稍後才由 NWDAF backend-loss cleanup 收斂。

Infrastructure 隨後恢復原本的 project-scoped stop，避免保留一個增加關機時間卻無法解決
問題的 ordering。使用者決定先在 Infrastructure lifecycle 採固定 grace，不為此新增
PyAnLF／PyMTLF／NWDAF runtime API：`experiment-stop` 完成 consumer exact DELETE 後，預設
保持所有 ML containers 與 Guest services 可用 40 秒，再呼叫原本的 project-scoped
`ml-stop`；`ML_CLEANUP_GRACE_SECONDS` 可設非負整數覆寫，非法值會在任何 runtime mutation
前拒絕。40 秒高於前一輪約 30 秒的 late reconciliation window；`0` 只供明確除錯使用。

後續 CPU active-runtime regression 於 `13:08:48Z` 建立 A／B 兩條 Model Monitor
subscriptions。consumer resources 於 `13:10:11Z` exact DELETE，同一秒內 A／B Model
Provision、Monitor registration 與 C 持有的兩條 monitor subscription 全部回覆 `204`；第一個
ML container 到約 `13:10:53Z` 才 shutdown。完整時段沒有 DELETE `503`、ERROR 或 traceback。
teardown 後 23 個 Guest units inactive、五個 containers exited、consumer inactive，三台 VM
均 graceful poweroff。本輪只使用 ignored CPU config，因 Host NVIDIA driver 當時不可用；這不
影響 teardown contract，也不構成 GPU runtime 回歸。完整 timing、identity、state 與資源證據見
[Runtime Helper Sync 與 FL Lifecycle 回歸](../../reports/5g-nwdaf-infrastructure/runtime-helper-sync-and-fl-lifecycle-regression-2026-08-12.md)。

## 16.26 Optional WebConsole Lazy-Build 計畫

使用者確認跳過 CPU full FL closure，下一個工作包改為 optional free5GC WebConsole。現有
Infrastructure 已固定 upstream WebConsole submodule `70d282f`，並保存指向 Core MongoDB、NRF
及 management address `192.168.56.10:5000` 的 `webuicfg.yaml`，但尚未建立 config enable
contract、Guest build artifact、systemd unit、Host lifecycle 或實際 smoke。

WebConsole 的 optional 語意同時涵蓋 build 與 runtime：

- 完整 config 的 `optionalServices.webconsole.enabled` 預設為 `false`；disabled 時不安裝
  Node／Yarn、不建置 frontend／Go server，也不啟動 process。
- enabled 時，`webconsole-start` 以 WebConsole source revision、frontend lock 與 build helper
  identity 計算 artifact identity。artifact 缺少或 identity 不符才在 Core 安裝固定 toolchain 並
  lazy build；相符時直接重用，不因每次 experiment start 重建。
- config 後續改回 disabled 只停止／略過 process，不自動刪除既有 artifact、toolchain 或 cache；
  任何空間回收另作明確 cleanup，不隱含在 lifecycle。
- WebConsole 僅監聽 VirtualBox host-only management address，不對實驗室 LAN 或 Internet 暴露。
  鎖定的 upstream revision 因 boolean validator regression 無法接受
  `billingServer.enable: false`，因此暫時保留 `true`，但 FTP control listener 固定只綁 Core
  loopback `127.0.0.1:2121`，不對 Host 或實驗室網路暴露；不修改 upstream submodule。
  Billing transfer、TLS、certificate、OAuth 與 subscriber write path 不在第一版驗收範圍。

實作分為五個 bounded gates：

1. 在 default manifest、renderer 與 checker 加入 boolean enable flag，並逐欄驗證 WebConsole
   MongoDB database／URI、NRF URI、HTTP management endpoint、billing compatibility settings 及
   config identity；negative contract smoke 必須拒絕 endpoint 或 billing drift。
2. 新增 Core-only lazy-build helper。它只在 enabled start 路徑安裝 Node.js 20、Corepack／Yarn，
   build frontend 與 Go server，原子發布 runtime directory 及 identity；啟動階段不接受 stale
   artifact，也不修改 upstream WebConsole submodule。
3. 新增獨立 `webconsole-start`／`status`／`stop` 與 Core systemd unit。start 必須要求 Core VM、
   MongoDB、NRF、active config 與 port ready；stop 不得停止 5GC。status 顯示 config enable、unit、
   HTTP 與 artifact identity。
4. 將 optional domain 納入 `experiment-start` rollback、aggregate status、cleanup-safe stop、logs 與
   Make help；disabled path 必須維持既有 experiment behavior，不能引入 Node／Yarn 或 build。
5. 先跑 repository tests 與 disabled-path regression，再用 enabled local config 做 bounded Core
   smoke：frontend HTTP、`admin/free5gc` login token、authenticated subscriber list 可讀、FTP
   listener 只綁 loopback、獨立 stop 後其他 Guest services 仍 active。若需要修改 WebConsole、
   NRF 或其他 NF source，依既定邊界先停止並回報。

本工作包只修改 `5G_NWDAF_Infrastructure` 與 `testbed-docs`。現有 VM 的第一次 enabled smoke
會在 Core 安裝 guest packages 並產生 build artifact；執行此 privileged Guest mutation 前另行
回報實際命令與資源狀態。

2026-08-12 實作與 bounded smoke 已完成。disabled regression 證明不接觸 VM、toolchain 或
artifact；enabled config 完成首次 Node.js 20／Yarn 4.1 lazy build、HTTP frontend、登入、六筆
subscriber read、Core-loopback FTP listener、獨立 stop、23 個 Guest services 保持 active，以及
第二次啟動 artifact reuse。upstream 每次 startup 會重建其 `admin` tenant/user 並重設為
`admin/free5gc`，Host lifecycle 已明確警告；config-owned subscriber records 不受此動作取代。
三台 VM 最後均 graceful poweroff。詳細證據見
[Optional WebConsole Lazy-Build Smoke](../../reports/5g-nwdaf-infrastructure/optional-webconsole-lazy-build-smoke-2026-08-12.md)。

## 16.27 PLMN 單一來源與 Mobile Identity 衍生計畫

使用者確認下一個工作包收斂目前只套用一半的 PLMN customization。現況 renderer 已將
`mobileNetwork.plmn` 寫入大部分 NF、gNB、UE MCC/MNC、NWDAF 與 Consumer，但 Path TAI
另存 `plmn`、六個 UE 另存完整 SUPI、Mobile Network 與 Consumer 各自另存完整 Internal
Group ID，subscriber/group fixtures 也仍是 baseline `466/92`。checker 會阻止混合 config
啟動，因此不會 silent 使用錯誤 PLMN，但只修改 canonical 欄位尚無法產生可執行 config。

本工作包採以下 contract：

- `mobileNetwork.plmn.mcc/mnc` 是唯一 MCC/MNC 來源；MCC 必須為 3 digits、MNC 必須為 2 或
  3 digits。
- Path TAI 只保存 TAC，不再重複 PLMN。
- Path UE 只保存正整數 subscriber number。renderer 以 15-digit IMSI 總長計算 MSIN width，
  將 number 補零後產生 `imsi-<MCC><MNC><MSIN>`；因此二位與三位 MNC 都不會產生 16-digit
  IMSI，default number 1–6 仍得到現有六個 SUPI。
- Internal Group 只保存 8-hex Group Service Identifier 與 2–20-hex、偶數長度 Local Group
  ID；renderer 依 TS 23.003／TS 29.571 組成 `<service>-<MCC>-<MNC>-<local>`，Mobile Network、
  UDM、Consumer 與 fixture 不再各自保存完整 group string。
- GPSI/MSISDN 不是 PLMN，不從 MCC/MNC 衍生。現有六筆 GPSI 保留為獨立 fixture identity；
  文件與 topology comments 明確避免把表面相同的數字前綴誤當 PLMN contract。
- renderer 在每個完整 config set 內重寫 subscriber `servingPlmnId`、SUPI list、S-NSSAI、DNN，
  以及 group ID/member list；authentication、GPSI、AMBR、PDU/QoS defaults 保持 baseline。

實作與驗證只修改 `5G_NWDAF_Infrastructure` 與 `testbed-docs`，不修改任何 NF／ML／RAN
submodule。`configlib.py` 提供共用 PLMN、SUPI、Internal Group ID derivation，renderer、checker
與 dataset UE count 共用同一 topology contract。repository tests 新增非預設二位 MNC
`001/01`、三位 MNC `001/001` positive cases，以及 derived SUPI／group／TAI drift negative cases；
全部使用 temporary topology/config，不啟動 VM 或 process。完成後更新 config/operations 文件並
以既有 `466/92` regression 證明 committed default identities 不變。

2026-08-12 實作與 isolated regression 已完成。topology 現在只保存 canonical MCC/MNC、
Internal Group 的 service/local portions、Path TAC 與 subscriber numbers；renderer 同步重寫
NRF、AUSF、NSSF、AMF、SMF、ADRF、NWDAF-A/B、UERANSIM、UE routing、UDM、Consumer 與
subscriber/group fixtures。checker 對同一批欄位逐項反向核對，並拒絕 topology 重新加入完整
SUPI、TAI PLMN 或完整 Group ID 等第二來源。

repository regression 分別以 `001/01` 與 `001/001` 只替換 canonical PLMN，完整 render/check
皆通過；第一筆 SUPI 分別為 `imsi-001010000000001` 與 `imsi-001001000000001`，Group ID 分別為
`00000001-001-01-01` 與 `00000001-001-001-01`。六筆既有 GPSI 在兩案均維持不變。derived
SUPI、UE routing、NSSF PLMN、SMF TAI 與 Consumer Group 的獨立 drift 均被 negative tests
拒絕。完整 repository tests 除 sandbox 內 Vagrant home 權限外全部通過；`vagrant validate`
已在正常 home 權限下獨立通過，沒有啟動 VM 或 process。證據見
[PLMN Single-Source Config Regression](../../reports/5g-nwdaf-infrastructure/plmn-single-source-config-regression-2026-08-12.md)。

## 16.28 2026-08-12 Submodule HTTPS Transport

為移除公開環境對個人 SSH key 的預設依賴，Infrastructure 已將 9 個
`Intelligent-Systems-Lab` submodule URL 從 `git@github.com:...` 改為
`https://github.com/...`；原本 7 個 public upstream submodules 亦維持 HTTPS。branch hint、parent
gitlink 與 `components.lock.yaml` revision 均未變更，且沒有使用 `submodule update --remote`。

本機執行 `git submodule sync --recursive` 後，16 個既有 submodule 的 `origin` 全部為 HTTPS；
逐一執行 remote `HEAD` read 及 pinned `git submodule update --init --recursive` 均成功，working
trees 仍位於原本的 16 個 revisions。現階段 9 個 team repositories 仍為 private，因此這項結果
證明的是 authenticated HTTPS transport；只有各 component visibility 與 license blocker 解決後，
才能另作無 credential 的 fresh recursive checkout acceptance。

## 16.29 2026-08-12 Guest Provisioning Version Policy

公開準備、license、remote、CI 與 anonymous checkout 依使用者決定延後；N6 topology 與現有
`egress` 宣告保留，不納入本工作包。下一個 bounded change 只處理 Guest provisioning 使用的
Go toolchain 與 MongoDB packages，避免未來重新建立 VM 時 silent 取得不可追溯版本。

Infrastructure 將新增單一 `provisioning.lock.yaml`，保存 Go archive version／URL／official
SHA-256，以及 MongoDB preferred packages、相容 `8.0` series、repository 與 signing-key
fingerprint。Guest scripts 不再保存第二份版本常數；Host repository test 驗證 lock schema、版本、
URL、checksum 與 package coverage。provision 完成後，Guest 保存 requested／resolved identity、
lock hash、OS／kernel 與 drift reason，供後續稽核與 evidence 使用。

失敗政策區分 compatibility drift 與 artifact integrity：

- Go archive checksum、解壓後 binary version、MongoDB signing-key fingerprint、architecture 或
  必要 package 缺失不符時 fail closed；這些不能降級為 warning。
- MongoDB 優先使用目前已通過 E2E 的 exact package versions；preferred version 不可取得或現有
  VM 已是不同 patch 時，只要所有必要 packages 仍位於 `8.0.x` compatible series，就保留／解析
  實際版本、顯示 warning 並將 `drift: true` 寫入 manifest，不自動升降級既有 database。
- MongoDB 跨出 compatible series、混合 incompatible component series 或無法解析 version 時
  停止，不 silent fallback。

驗證順序為 repository static/negative contract、官方 Go artifact checksum、MongoDB package
availability、現有 Core VM installed identity、disposable Ubuntu 22.04 clean-install smoke，以及
same-version existing-VM regression；真正 fresh Core provisioning 可併入下一次 VM rebuild gate。
現有 Core database 不為測試移除。若需啟動已關閉 VM、建立 disposable container／VM 或執行
package mutation，先回報當下資源與 exact action。

Repository 文件整體重構另作單一 atomic 工作包：建立新的 `docs/` canonical hierarchy、更新
所有入口與連結後，直接移除被取代的過期文件，不保留舊 `OPERATIONS.md` 或其他 stale
compatibility copies。本工作包只維護必要的版本 contract 與本計畫，不提前進行部分搬移。

2026-08-12 bounded implementation 與驗證已完成。Infrastructure 新增
`provisioning.lock.yaml`、共用 lock validator/resolver、Guest manifest、repository negative
regression 與 disposable Ubuntu 22.04 clean-install smoke。現有 Core 的 Go 1.26.2、MongoDB
server 8.0.28、mongosh 2.9.2、database tools 100.17.0 和 signing key 全部符合 preferred
identity，resolver 回報 installed/no-drift；沒有為 audit 修改既有 package、hold 或 runtime
service。乾淨容器亦成功以 exact versions 安裝全部九個 MongoDB packages，並驗證 official Go
archive checksum／binary identity 與 MongoDB key fingerprint。詳細證據見
[Guest Provisioning Version Lock](../../reports/5g-nwdaf-infrastructure/guest-provisioning-version-lock-2026-08-12.md)。

## 16.30 Single Testbed Definition

目前 `testbed.local.yaml` 並不是 `testbed.yaml` 的一般 overlay，只由不同入口零散讀取
provider、Host bind address、storage expectation 與 default config directory。檔名卻暗示它能
覆寫完整 testbed；未支援欄位即使存在也會被忽略。實際部署從未建立此檔，最初 provisioning
以 `VAGRANT_DEFAULT_PROVIDER=virtualbox` 選擇 provider，後續既有 VM 則由 `.vagrant`
metadata 辨識。這個隱藏的 fallback 結構不適合作為公開操作 contract。

下一個 bounded change 將移除 local layer，建立以下單一來源規則：

- `TESTBED` 選擇一份完整且自足的 testbed definition；未指定時使用 committed
  `testbed.yaml`。VM resources、network、Host ML endpoint、NF／RAN／NWDAF topology、mobile
  identity、host safety threshold 與 default config directory 全由該檔案提供。
- `CONFIG_DIR` 只負責選擇這次要啟用的完整 generated/native config set；顯式參數優先，未指定
  時只 fallback 至 selected testbed 的 `config.directory`。不再存在第三個 hidden default。
- reference runtime 只支援 VirtualBox，因此 provider 是程式能力而非使用者 topology 選項；
  Vagrantfile 與 preflight 直接採 VirtualBox，並拒絕衝突的 provider environment。
- VirtualBox machine folder 與 Docker data root 從各 runtime 實際查詢後執行 free-space gate，
  不保存重複的 expected path。這些 discovery result 是 Host 狀態，不是可重現 topology。
- Host ML bind address 直接使用 selected testbed 的 `mlRuntime.bindAddress`；
  `advertisedAddress` 仍代表 VM／NF 看見的 endpoint，兩者繼續由 config checker 驗證可用性。

欄位處理如下：

| 舊 local 欄位 | 新行為 |
|---|---|
| `provider.name` | 移除；reference implementation 固定 VirtualBox |
| `provider.expectedVmStorage` | 移除；由 `VBoxManage` discovery 並檢查實際 filesystem |
| `host.mlBindAddress` | 移除；使用 selected testbed 的 `mlRuntime.bindAddress` |
| `host.dockerDataRoot` | 移除；由 `docker info` discovery 並檢查實際 filesystem |
| `config.directory` | 移除；使用顯式 `CONFIG_DIR` 或 selected testbed 的 `config.directory` |

實作範圍包括刪除 `testbed.local.example.yaml` 與 `.gitignore` ignore rule，移除
`configlib.py` 的 local settings loader，收斂 Vagrantfile／preflight／config／ML helpers 的
resolution path，並清除 README、OPERATIONS 與 script comments 中所有 local-layer 指引。
若 working tree 存在舊 `testbed.local.yaml`，preflight 應 fail with migration guidance，避免使用者
誤以為其中設定仍生效；本機目前沒有此檔，因此不需要刪除使用者資產。

替代環境使用完整檔案，例如 `TESTBED=testbed.my-lab.yaml`，而不是 partial override。renderer
將 selected testbed path 與 canonical hash 寫入 generated config manifest；checker 維持以同一
selected definition 驗證 topology hash，避免建立 config 後切換 testbed。所有同一 lifecycle 的
commands 必須傳入相同 `TESTBED` 與 `CONFIG_DIR`。

驗證包括 local reference elimination、selected-testbed／explicit-config precedence、替代完整
testbed positive case、stale local file negative case、config render/check、dataset、ML Compose、
repository test 與 `vagrant validate`。此變更只修改 Host orchestration 與文件，不修改 config
內容、N6、submodule、既有 VM definition 或 Guest runtime；不需重建或啟動 VM。

2026-08-13 bounded implementation 與驗證已完成。`testbed.local.example.yaml`、ignore rule、
local config/bind/storage/provider resolution 均已移除；VirtualBox 與 Docker storage 由 runtime
discovery，Host ML bind 與 default config 只由 selected testbed 提供。repository regression
證明 alternate `TESTBED`、explicit `CONFIG_DIR` precedence、stale local rejection、non-VirtualBox
provider rejection 與正常 Vagrant definition。實際無 local/provider environment 的 Host preflight
為 0 failures、1 個既有 low-swap warning，並正確找到 VirtualBox、Docker、Host SBI address、
config、component locks 與 dataset。詳細證據見
[Single Testbed Definition Regression](../../reports/5g-nwdaf-infrastructure/single-testbed-definition-regression-2026-08-13.md)。

## 16.31 Atomic Repository Documentation Restructure

2026-08-13 read-only audit 確認 Infrastructure `main` 位於 `10400a5`、working tree clean，
`testbed-docs` `main` 位於 `fbac1f7`、working tree clean；parent repository 尚未設定 remote。
目前 parent-owned 說明分散在 11 份 Markdown、共 1,064 行：root `README.md`、
`OPERATIONS.md`、`COMPONENTS.md`，四個分類 README、config／Host／Guest scripts README，
以及 Consumer README。它們沒有形成可巡覽的文件 hierarchy，且只有 root README 三個
cross-document links。

盤點確認以下內容問題：

- root README 同時承擔 quick start、架構、操作、歷史 smoke、容量觀察與 validation ledger，
  使新使用者無法快速分辨 current contract 與過去 evidence。
- `OPERATIONS.md` 同時描述安裝、config、VM build、WebConsole、reset、dataset、GPU、
  subscription、logs 與 non-goals；相同資訊又散落在 config／scripts README。
- `OPERATIONS.md` 仍說 production integration 未驗證 concurrent training，`COMPONENTS.md`
  仍說目前 UPF lock 尚未建立 FL closure；但相同 component revisions 已有 business E2E、
  bounded smoke 與 runtime-helper regression 證明 training、FedAvg、publication、reprovision
  與 cutover。這些句子已是 stale boundary。
- `NFs/`、`ML/`、`RAN/`、`kernel/` 四份 parent README 合計只有 17 行，只增加入口數量；
  config、Host scripts、Guest scripts 與 Consumer README 的有效內容可由 topic guides 完整吸收。
- `.gitmodules` 的 16 個 URL 已全為 HTTPS，但 `components.lock.yaml` 內 9 個 team remote
  metadata 仍是 SSH。preflight 目前只比對 installed submodule HEAD 與 lock commit，沒有驗證
  lock path set、`.gitmodules` URL／branch hint 或 parent gitlink，因此先前未發現這項 drift。
- 現有 user docs 沒有獨立的 prerequisite/install guide、完整 Make command reference、
  troubleshooting guide、link checker 或 command-coverage regression。

本工作包採一次性最終 layout：

```text
README.md
docs/
├── README.md
├── architecture.md
├── installation.md
├── configuration.md
├── operations.md
├── commands.md
├── components.md
├── validation.md
└── troubleshooting.md
```

公開 repository 文件延續既有英文；`testbed-docs` 繼續保存中文計畫、逐次驗證 evidence 與
歷史時間線。兩者責任不同：Infrastructure `docs/` 只描述目前使用者 contract，不複製逐次
runtime logs；`testbed-docs` 的 dated reports 不搬入公開 repository，也不因過時就改寫歷史結果。

各文件責任如下：

- root `README.md`：專案目的、三 VM＋五 container 摘要、可直接執行的最短標準實驗流程、
  quick stop、文件導航與目前 license；移除長篇 validation chronology 與深入操作細節。
- `docs/README.md`：依「第一次安裝、建立實驗、執行、診斷、開發／驗證」提供唯一導航表。
- `architecture.md`：placement、network planes、Host/Guest boundary、六個 lifecycle domains、
  Consumer/NRF subscription、ADRF/FL data flow 與 retained-state ownership。另設「Reference
  experiment flow」，依序描述 UE registration／PDU Session、Consumer 經 NRF 找到 A/B、Nupf
  Event Exposure、PyAnLF analytics、C Model Monitor、A/B federated training、FedAvg、ADRF
  publication、reprovision 與 generation cutover。N6 只按目前 topology contract 描述為保留的
  private plane 與 UPF alias；PseudoDriver 明確描述為可重現 stimulus。
- `installation.md`：Linux／VirtualBox／Vagrant／Docker／Python prerequisites、private HTTPS
  submodule authentication、VirtualBox allowlist、CPU/GPU paths、resource gates、submodule init、
  first validation 與 VM provisioning。Docker/VirtualBox storage 由 runtime discovery，不再引入
  Host-local settings file。
- `configuration.md`：single `TESTBED` contract、`CONFIG_DIR` precedence、default/generated/local
  sets、scenario ownership、PLMN／SUPI／Group derivation、WebConsole/device policy、dataset generate／
  audit／stage、manifest hashes 與 config activation。
- `operations.md`：第一節即為「Standard experiment workflow」，完整列出 config create、dataset
  generate、read-only validate、VM up、首次／後續 clean-run state handling、aggregate start、status／
  logs、aggregate stop 與 optional VM halt，並說明每一步成功後應看到什麼。後續再描述 VM、Guest
  services、optional WebConsole、Host ML、Consumer/subscriptions 的分域 start/status/stop、40-second
  cleanup grace、subscriber projection、guarded reset、observation/logs 與 destructive boundary。
- `commands.md`：Makefile 所有 targets 的完整分級表。primary、advanced、developer/internal alias
  分開，每個 target 記錄參數、state mutation、prerequisite 與保留內容；正常使用者不需直接操作
  Host／Guest helper scripts。
- `components.md`：parent gitlink ownership、`.gitmodules` transport、`components.lock.yaml` metadata、
  branch/tag hint、UPF/gtp5g compatibility、Guest `provisioning.lock.yaml`、private visibility 與 license
  blocker。revision／URL／version 數值以 lock files 為唯一來源，不在 prose 重抄完整表格。
- `validation.md`：只記錄目前已通過且仍適用的 static test、Host preflight、clean-install smoke、
  existing-VM provisioning、business E2E、bounded closure 與 cleanup regression。它是 verified
  capability summary，不放未完成項目、未來 roadmap，也不把舊 smoke limitation 誤寫為 current
  capability。
- `troubleshooting.md`：resource gate／low swap、dirty 或 mismatched submodule、stale
  `testbed.local.yaml`、config/hash/dataset mismatch、missing Netplan alias、gtp5g kernel drift、Docker
  permission、GPU CDI/runtime、WebConsole billing workaround、subscription cleanup retry 與 safe log
  collection。

Atomic migration 直接刪除被新 hierarchy 完整取代的十份文件：`OPERATIONS.md`、
`COMPONENTS.md`、`NFs/README.md`、`ML/README.md`、`RAN/README.md`、`kernel/README.md`、
`config/README.md`、`scripts/host/README.md`、`scripts/guest/README.md` 與
`tools/nwdaf-consumer/README.md`。不保留 redirect、compatibility copy 或 archive；submodule 內由
各 upstream 擁有的 README 不在 parent change scope。root README 原地重寫，所有 old-path links
在同一 change 內切到 `docs/`。

在撰寫文件前，將 `components.lock.yaml` 的 9 個 team remotes 改為與 `.gitmodules` 相同的
HTTPS URL，不改 branch hints、gitlinks 或 submodule working trees。既有 preflight 繼續負責
installed HEAD 與 readable lock commit 的必要比對；不為 metadata 或文件增加額外使用者命令與
獨立 Host scripts。

文件驗證直接納入本次 review，而不擴張 command surface：確認十份 canonical Markdown 都存在、
十個舊路徑已移除、relative links 指向有效檔案、Makefile targets 全部在 `docs/commands.md` 有
說明，並搜尋過時 local override、舊 backend sync contract 與已被 E2E 推翻的 limitation。歷史
report 不套用 current-doc rule。

實作與驗證順序：

1. 修正 component remote metadata，確認不改 branch、gitlink 或 submodule worktree。
2. 建立全部九份 target documents 與精簡 root README；內容以 testbed、Makefile、scripts、locks、
   scenarios 和 current component source 為依據。
3. 在同一 working tree 刪除十份舊文件、更新所有 links；不留下半遷移入口。
4. 執行 `make help-all`、完整 `make test`、read-only Host preflight 與
   `vagrant validate`。不執行 container lifecycle、VM up/provision 或 E2E。
5. 在 `testbed-docs` 新增 dated restructure report、更新 report index 與本節完成結果。
6. 依 repository boundary review。Infrastructure 可分為一個 component-metadata fix commit 與一個
   atomic documentation commit；`testbed-docs` 另作 documentation commit。使用者確認前不 push。

本工作包只修改 `5G_NWDAF_Infrastructure` 與 `testbed-docs`，不修改 NF／ML／RAN／gtp5g／
WebConsole source，不變更 gitlinks、testbed/config/scenario、N6、VM、container 或 runtime state。
公開文件只陳述現在可使用、可操作或已有 evidence 的內容，不建立未完成清單。

2026-08-13 atomic implementation 與驗證已完成。Infrastructure 現在只保留 root README 與
`docs/` 九份 canonical guides；root、operations、architecture 分別提供最短流程、詳細 operator
流程與 full-core FL 概念流程。十份舊文件已直接移除。`components.lock.yaml` 的九個 team
remotes 已與 `.gitmodules` 對齊 HTTPS，沒有新增使用者命令或維護用 Host script。

本次 review 確認 10 份 canonical Markdown、全部相對連結與現有 Make targets 的 command
coverage。完整 repository test 與 Vagrant validation 通過；實際 Host
preflight 為 0 failures、1 個既有 low-swap warning，沒有啟動 VM、container 或 service。
詳細證據見
[Atomic Repository Documentation Restructure](../../reports/5g-nwdaf-infrastructure/atomic-repository-documentation-restructure-2026-08-13.md)。

## 16.32 Operator Command And Test Layout Cleanup

2026-08-13 audit 確認 Infrastructure 現有 55 個 Make targets，其中真正顯示於 `help`、
`help-advanced` 與 `help-dev` 的使用者入口已相對精簡，但 Makefile 與 `docs/commands.md` 仍公開
多個只供 repository regression 使用的 smoke target，以及多組指向同一實作的 alias。另一方面，
九個 test-only runner/helper 與正式 Host lifecycle scripts 混放在 `scripts/host/`，使檔案位置看不出
production operation 與 regression 的差別。

本輪清理採以下原則：

- 使用者 Make surface 只保留可操作 environment、建立 input、觀察狀態或執行完整測試的命令。
- 細項 regression 由 `make test` 統一執行，不各自暴露 Make target。
- container lifecycle test 因耗時、會建立 disposable containers，保留獨立且明確的
  `make test-containers`，不混入 Host-only `make test`。
- production validation 不因整理而刪除：`config-check.py`、`preflight.sh`、
  `ml-compose-check.py` 與 `gtp5g-preflight.sh` 仍由 start／validate 流程在必要時直接呼叫。
- 不把 test code 併成單一大型腳本，也不增加 wrapper；只把既有 test-only files 移至清楚的
  `tests/` hierarchy 並修正引用。
- `fl-closure-smoke` 是可執行的 bounded experiment example，不是 repository test，scenario 與
  traffic profiles 保留原名及位置。

### Make command surface

移除以下只供內部 regression 使用的 targets；對應檢查仍由 `make test` 執行：

- `config-contract-smoke`
- `network-config-smoke`
- `dataset-smoke`
- `ml-compose-check`

移除以下重複 alias，只保留 `make test-containers`：

- `ml-cpu-smoke`
- `ml-lifecycle-smoke`

同時收斂只為暴露內部 implementation name 而存在的 targets：

| 移除 target | 保留的使用者入口 | 內部行為 |
|---|---|---|
| `config-check` | `config-validate` | Make 直接呼叫 `scripts/host/config-check.py` |
| `config-render` | `config-create` | renderer script 仍由 `config-create` 與 tests 使用 |
| `dataset-check` | `dataset-validate` | Make 直接呼叫 `dataset.py ... check` |
| `dataset-stage` | `dataset-load` | Make 直接呼叫 `dataset-stage.sh apply` |
| `dataset-stage-plan` | `dataset-show`／`dataset-load` | staging plan 留在 helper 內供診斷，不另作 Make command |
| `subscriber-data-validate` | `subscriber-data-show` | `show` 已包含 fixture validation 與 expected/actual comparison |
| `subscriber-data-plan` | `subscriber-data-show` | `show` 已顯示 apply/clear scope |
| `experiment-reset-plan` | `reset-show` | Make 直接執行 reset helper 的 `plan` action |
| `experiment-reset` | `reset` | `reset` 直接執行 guarded apply |
| `experiment-reset-verify` | `reset` | apply 後仍自動 verify，不提供拆開的正常入口 |
| `preflight` | `experiment-validate` | preflight script 仍由 aggregate validation/start 使用 |

`observe` 是持續狀態觀察而非測試，保留並補進 advanced help；`experiment-status` 繼續提供單次
snapshot，`logs` 提供即時 log channel。VM、services、ML、WebConsole、subscriptions、subscriber
data 與 reset 的正常操作 targets 全部保留。

整理後 Makefile 預計只留下 38 個入口：四個 help、四個 aggregate experiment、兩個 config、
四個 dataset、三個 VM、各三個 services／ML／WebConsole／subscriptions、三個 subscriber-data、
兩個 reset、`observe`、`logs`、`test` 與 `test-containers`。`docs/commands.md` 只記錄這些使用者可
直接使用的入口，不再把內部 Python/shell implementation 描述成額外命令。

### Test layout

新增單一 `tests/` 分類，但不新增測試種類：

```text
tests/
├── repository.sh
├── config-contract.py
├── dataset-determinism.sh
├── mobile-identity.py
├── network-config.py
├── provisioning-lock.py
├── testbed-definition.py
├── ml-container-lifecycle.sh
└── support/
    └── ml-cpu-config.py
```

既有檔案以 `git mv` 語意一對一搬移並去掉 `smoke` 命名；runner 仍執行相同 assertions：

- config contract test 保留，因它會攔截 report capacity、endpoint、callback、PLMN、ADRF、
  WebConsole 與 manifest 等實際曾發生的簡單 config mismatch。
- mobile identity test 保留，確保二位／三位 MNC 下 PLMN、TAI、SUPI 與 Internal Group 一起生成。
- network config test 保留，覆蓋 persistent Netplan alias、stale address 與 collision regression。
- dataset determinism test 保留，證明兩次生成相同且篡改 Parquet 會被拒絕。
- provisioning lock 與 single-testbed definition tests 保留，分別保護 Guest dependency contract 與
  已移除 local overlay 的 resolution contract。
- ML container lifecycle test 保留為唯一獨立 container test，helper 移至 `tests/support/`。

`scripts/host/provisioning-install-smoke.sh` 沒有 Make target、沒有被 runner 或任何其他腳本引用，
且其一次性 Ubuntu 22.04 clean-install 結果已保存在 dated report。它會重新下載並安裝完整 Go／
MongoDB package set，成本高且不是日常 regression，因此直接刪除，不搬入 `tests/`。

### Runtime script boundary

整理後 `scripts/host/` 只保留下列 production responsibilities：

- aggregate experiment validate/start/status/stop；
- VM 以外的 services、ML、WebConsole 與 subscriptions lifecycle；
- config render/check 與 shared config library；
- dataset generate/audit/stage 與 shared dataset library；
- subscriber projection、guarded reset、Host preflight、gtp5g preflight；
- guest helper sync、observation、logs 與共用 shell/Python helpers。

`scripts/guest/` 不調整；其中 setup、activation、systemd runtime 與 scoped data operations 都是 Guest
實際行為，不是多餘 tests。`tools/nwdaf-consumer/consumer.py` 亦是 runtime component，不移入
tests。

### Implementation and validation

實作採一個 bounded Infrastructure change，不修改 submodule、gitlink、config、scenario、VM 或
runtime state：

1. 先建立 `tests/` 並搬移九個 runner/helper；修正 `ROOT`、imports 與 runner paths。
2. 刪除未引用的 provisioning install smoke。
3. 精簡 Makefile targets，讓 user-facing aliases 直接呼叫既有 implementation；確認 aggregate
   scripts 不依賴被移除的 Make names。
4. 更新 README、`docs/commands.md`、validation 與 troubleshooting，只呈現最終 command surface；
   不為舊 target 保留 compatibility alias。
5. 執行 shell/Python syntax、搜尋舊 target/path、`make help-all`、完整 `make test`、read-only
   `make experiment-validate CONFIG_DIR=config/default`。`make test-containers` 會建立 disposable
   containers，另列為需使用者決定是否執行的 runtime validation，不自動加入本輪 static test。
6. 在 `testbed-docs` 新增 dated result report 並依 repository boundary review；使用者確認前不
   commit 或 push。

2026-08-13 bounded implementation 與 read-only validation 已完成。Make targets 已由 55 個收斂
為 38 個；細項 regression 與 implementation-name aliases 已移除，`observe` 則保留並加入
advanced help。九個既有 test-only files 已整理至 `tests/`，未新增測試種類；沒有 caller 且會執行
完整外部 package installation 的 `provisioning-install-smoke.sh` 已刪除。

新 layout 下完整 `make test` 通過，read-only
`make experiment-validate CONFIG_DIR=config/default` 亦以 0 failures、1 個既有 low-swap warning
通過，並確認 preflight、Compose、GPU CDI/runtime 與 Vagrant validation 仍由正式 aggregate 流程
執行。後續 `make test-containers` 亦完成五個 CPU services 的 health、non-root identity、device、
logs、status、stop、retention 與 scoped cleanup regression；測試 project 的 containers、volumes、
network 與 generated config 已清除，沒有建立 VM 或改變 production services。詳細結果見
[Operator Command And Test Layout Cleanup](../../reports/5g-nwdaf-infrastructure/operator-command-and-test-layout-cleanup-2026-08-13.md)。
