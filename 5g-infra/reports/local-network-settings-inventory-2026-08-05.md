# 5G Infrastructure Local Network Settings Inventory

日期：2026-08-05

狀態：第一版靜態盤點完成；host 網路已唯讀確認；VM guest runtime 尚未驗證

## 1. 目的

本文件保存 `/home/chingje/testbed/5G_Infrastructure` 在實驗室共用網路環境中
使用的 site-specific 設定，供未來清理現有 working tree、重新 clone upstream
並重建 testbed 時使用。

這不是新版 NWDAF 架構文件，也不代表現有舊 testbed 已可正常啟動。新版 NWDAF
方向與 full-core federated learning 實驗應以 `nwdaf-docs/` 為準；本文件只回答：

- 這台 host 使用什麼實體介面與 IP；
- 舊 testbed 對 upstream 網段做了哪些位移；
- 五台主要 VM、NF、UERAN、UE pool 與 service endpoint 如何對應；
- 哪些差異可以由 `.agent/setup.sh` 重建；
- clean rebuild 前還有哪些本地資產必須另行保存。

## 2. 盤點方法與限制

本次唯讀檢查：

- `5G_Infrastructure` 的 git status、tracked diff、untracked inventory 與
  submodule revisions；
- Vagrantfile、component `setup.sh`／`provision.sh`；
- `config/5GC/*.yaml`、`config/UERAN/*.yaml`、consumer JSON 與 Daisy nodes；
- host 的實際 interface、address、route、Vagrant global status 與 MongoDB
  listener／systemd unit。

本次沒有：

- 啟動、重建或登入任何 VM；
- ping 實驗室 gateway 或掃描 IP 衝突；
- 修改 infra、更新 submodule、切 branch 或 fetch／pull；
- 啟動或修復 MongoDB；
- 驗證 VM guest 內的 interface numbering、route、iptables、NF bind 或端到端封包。

`vagrant global-status` 是 Vagrant cache。它顯示五台主要 VM 都已登記且為
`poweroff`，但未執行 `--prune`，所以只能作現況參考。

## 3. 結論摘要

1. Host 現在確實使用 `enp2s0`，而不是舊註記中的 `eno1`、`enp6s0` 或
   `.agent/setup.sh` 訊息中的 `wlp85s0f0`。
2. Host 有線位址為 `140.113.110.77/24`，default gateway 為
   `140.113.110.254`；`wlp0s20f3` 目前 DOWN。
3. 舊 testbed 採 upstream 私有網段「第三段 +20」策略；目前 Vagrant 與主要
   5GC／UERAN config 的位移大致一致。
4. 主要舊拓撲是 `5GC`、`UPF-EES`、`UPF-EES2`、`gNB`、`gNB2` 五台 VM；
   I-UPF、PSA-UPF、MEC 系列雖也被機械式 patch，但既有操作文件標示為不使用。
5. `.agent/setup.sh` 只能重放一部分差異，不能當作完整 backup 或 rebuild tool。
6. `mongod-27018` unit 雖然 enabled，但自 2026-05-18 起為 failed，27018
   目前沒有 listener；27017 則正在監聽。舊文件宣稱 27018 正常運作已不符合現況。
7. 目前 dirty tree 不只包含網路設定。`nwdafcfg.yaml`、submodule revisions、
   ADRF／Daisy 整合與多個 untracked scripts 是另一類應用／實驗資產，不能與
   site-specific network patch 混為一談。
8. 此舊拓撲的兩個 gNB config 都是 `tac: 1`，不等於新版 Phase 7 計畫中的
   two-TAI profile。後續重建應重新設計 profile，不宜原封不動套用舊設定。

## 4. Host 現況

### 4.1 實體網路

| 項目 | 2026-08-05 唯讀結果 |
| --- | --- |
| 主要有線介面 | `enp2s0`，UP |
| Host IPv4 | `140.113.110.77/24` |
| Default route | `via 140.113.110.254 dev enp2s0` |
| Wi-Fi | `wlp0s20f3`，DOWN |
| Container networks | host 上另有多個 Docker／CNI bridge；不是本舊 Vagrant profile 的設定來源 |

Vagrantfile 的 `public_network` 全部 bridge 到 `enp2s0`。這是目前最重要的
machine-specific 設定之一；重新 clone 後 upstream 若仍使用其他 bridge 名稱，
必須以當時 host 實際 interface 再確認，不能盲目套字串取代。

### 4.2 MongoDB

舊 testbed 的預期設計是讓 VM 經 VirtualBox NAT host address
`10.0.2.2:27018` 使用 host MongoDB，避免與共用的 27017 衝突。

目前實際狀態：

| 項目 | 結果 |
| --- | --- |
| systemd unit | `mongod-27018.service`，enabled |
| unit runtime | failed since 2026-05-18 20:54:01 CST |
| 27018 listener | 無 |
| 27017 listener | `0.0.0.0:27017` 與 `[::]:27017` 存在 |
| dbpath | `/var/lib/mongodb-27018` |
| bind | unit 使用 `--bind_ip 0.0.0.0` |
| log | `/var/lib/mongodb-27018/mongod.log` 為零 bytes，這次無法由 log 判斷失敗原因 |

因此目前所有指向 `mongodb://10.0.2.2:27018` 的 config 只能視為「預期設定」，
不能視為可用 dependency。後續修復需另開診斷工作，不應在 clean rebuild 時順便
假設 service 正常。

## 5. Site-specific IP Shift

舊 upstream 網段以第三段 `+20` 避免與實驗室其他 testbed 衝突：

| Upstream | Local | 舊拓撲用途 |
| --- | --- | --- |
| `192.168.103.0/24` | `192.168.123.0/24` | Path 1 N3 |
| `192.168.105.0/24` | `192.168.125.0/24` | 共用 N2／N4 與部分 NF bind |
| `192.168.106.0/24` | `192.168.126.0/24` | Path 1 額外／N6-side network |
| `192.168.107.0/24` | `192.168.127.0/24` | NWDAF／ML／Daisy／UPF EES service network |
| `192.168.108.0/24` | `192.168.128.0/24` | Path 2 N3 |
| `192.168.109.0/24` | `192.168.129.0/24` | Path 2 額外／N6-side network |
| `192.168.110.0/24` | `192.168.130.0/24` | 舊 N3IWF 設定，主要五 VM profile 未使用 |
| `192.168.200.0/24` | `192.168.220.0/24` | 舊 I-UPF／MEC／DN network |

「用途」是依現有 Vagrant／config 與舊文件整理；由於 VM 都是 poweroff，尚未在
guest 內驗證每張 NIC 的實際介面名稱及資料平面行為。

## 6. 五台主要 VM 舊拓撲

### 6.1 VM 與 interface address

| VM | Vagrant bridged address | Config role | Guest default route script |
| --- | --- | --- | --- |
| `5GC` | `192.168.125.5`, `192.168.127.5` | AMF／SMF N2/N4；NWDAF/ML/Daisy service | `192.168.125.1 dev enp0s8` |
| `gNB` | `192.168.123.20`, `192.168.125.20` | Path 1 N3；N2/radio-link | `192.168.123.1 dev enp0s8` |
| `gNB2` | `192.168.128.21`, `192.168.125.21` | Path 2 N3；N2/radio-link | `192.168.128.1 dev enp0s8` |
| `UPF-EES` | `192.168.123.10`, `192.168.125.10`, `192.168.126.10`, `192.168.127.10` | Path 1 N3、N4、額外/N6-side、EES | `10.0.2.2 dev enp0s3` |
| `UPF-EES2` | `192.168.128.11`, `192.168.125.11`, `192.168.129.11`, `192.168.127.11` | Path 2 N3、N4、額外/N6-side、EES | `10.0.2.2 dev enp0s3` |

UPF setup 另對 `enp0s3` 加入 POSTROUTING MASQUERADE。這個「default route
改走 VirtualBox NAT」不是單純 `+20` 位移，而且 `.agent/setup.sh` 不會從乾淨
upstream 重建它，是 clean rebuild 時必須獨立保存的差異。

### 6.2 5GC／UERAN 對應

| Path | UE | gNB | UPF | Slice | UE pool | EES replay |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | IMSI `...001`–`...003` | N2 `192.168.125.20`, N3 `192.168.123.20` | N4 `192.168.125.10`, N3 `192.168.123.10` | `sst=1, sd=010203` | `10.10.0.0/24` | `group1`, 30 s |
| 2 | IMSI `...004`–`...006` | N2 `192.168.125.21`, N3 `192.168.128.21` | N4 `192.168.125.11`, N3 `192.168.128.11` | `sst=1, sd=112233` | `10.100.0.0/24` | `group2`, 30 s |

補充：

- 兩個 gNB 的 `tac` 目前都為 `1`。
- `smfcfg.yaml` 的 user-plane graph 是 `gNB1 -> UPF-EES` 與
  `gNB2 -> UPF-EES2`。
- `uerouting.yaml` 將兩組 SUPI 分別固定到兩條 path，並保留舊 MEC destination
  `192.168.220.25/32`、`192.168.220.35/32`。
- 既有操作文件說明主要 profile 不啟動 MEC／I-UPF／PSA-UPF，因此上述
  specific destination 需要在未來重整時重新確認是否仍有用途。
- UPF EES `periodSec`、NWDAF `samplingInterval` 與 consumer `rep_period` 目前
  都是 30 秒；SMF `urrPeriod` 仍為 5 秒，兩者不是同一個設定層級。

## 7. Service Endpoints

| Service | 舊 profile endpoint | 備註 |
| --- | --- | --- |
| 5GC web console forwarding | host `:5000` -> guest `:5000` | 唯一啟用的 Vagrant forwarded port |
| NWDAF | `192.168.127.5:8080` | consumer、SMF/UPF callback target |
| ML service | `192.168.127.5:9090` | 舊 inference backend |
| Daisy API | `192.168.127.5:9887` | 舊 MTLF integration |
| Daisy master | `192.168.127.5:8887` | `daisy/nodes.yaml` |
| Daisy clients | `192.168.127.5:10087`, `:10088` | 同一舊 5GC VM |
| UPF-EES 1 | `192.168.127.10:8088` | SMF `nupfEeApiRoot` |
| UPF-EES 2 | `192.168.127.11:8088` | SMF `nupfEeApiRoot` |
| ADRF | guest loopback `127.0.0.1:9888` | untracked `adrfcfg.yaml`；使用 host Mongo 27018 |
| MongoDB from VM | `10.0.2.2:27018` | 目前不可用，見 4.2 |

這些是舊單一 containing-NWDAF／Daisy profile 的 endpoint，不應直接拿來規劃
新版 A／B／C 三 NWDAF topology。

## 8. `.agent/setup.sh` 可重建與不可重建範圍

`.agent/setup.sh` 位於 git exclude 範圍，不會隨 repository clone 回來。本次檔案
SHA-256：

```text
80cf2468ccf74f83d23b2d98bbc4b59c29d27a95b1d3a2c3010cd52bd1d1765b
```

### 8.1 目前可機械重放

- Vagrantfile：`bridge: "enp6s0"` 改為 `enp2s0`；
- Vagrantfile、`config/**/*.yaml`、component `setup.sh`、Daisy nodes 與 consumer
  JSON：已知私有網段第三段 `+20`；
- `config/**/*.yaml`：Mongo URL 改為 `mongodb://10.0.2.2:27018`；
- 5GC SMF 與兩個 UPF submodule 複製進 guest 後，移除殘留 `.git` file。

### 8.2 目前不能重建

- UPF-EES／UPF-EES2 default route 從 N6-side gateway 改成
  `10.0.2.2 dev enp0s3`；
- 5GC Vagrant 增加 ADRF synced folder；
- ADRF／Daisy setup 與 run scripts；
- untracked `config/5GC/adrfcfg.yaml`、`daisy/daisyconfig.json`；
- gNB／UPF trace scripts；
- `nwdafcfg.yaml` 中非網路的 MTLF、WAPE、ADRF、model bundle 與 training 行為；
- submodule commit selection；
- runtime state、generated output、PID 與 log。

### 8.3 已確認的文件／腳本不一致

- `.agent/setup.sh` 的輸出訊息寫著 `bridge -> wlp85s0f0`，實際 sed 目標是
  `enp2s0`。
- 舊 `environment.md` 一度同時提到 `eno1` 與 `enp2s0`；本次已以實機結果修正
  canonical 說明。
- script 只辨識 upstream bridge 字串 `enp6s0`。未來 upstream 若改名，這個取代
  會靜默失效。
- script 的 IP mapping 是固定清單；新增網段不會自動被發現。
- 因此它是 local patch helper，不是完整 migration artifact，也不是設定管理系統。

## 9. Git 與本地資產現況

盤點時 `5G_Infrastructure`：

- branch：`main-NWDAF-enabled-closed-loop`；
- HEAD：`11b7e61`；
- local tracking ref 顯示 behind origin 9 commits，但本次未 fetch，僅作參考；
- tracked changed files：60；
- untracked entries：16。

目前 submodule／nested repository identity：

| Path | Revision | Parent 狀態 |
| --- | --- | --- |
| `5GC/smf-nwdaf-ext` | `8668372` | 符合 parent recorded revision |
| `NWDAF/NWDAF` | `ef36f6c` | 與 parent recorded revision 不同 |
| `NWDAF/NWDAF-ML-Service` | `2ad1296` | 與 parent recorded revision 不同 |
| `daisy/daisy` | `8fccdaa` | 與 parent recorded revision 不同 |
| `go-upf-ess/go-upf` | `95dc04f` | 與 parent recorded revision 不同 |
| nested `adrf/` | `75b871b`, `main` | parent 視為 untracked directory；nested repo clean |

高價值 untracked／excluded 候選：

- `.agent/setup.sh`、`.agent/clean_logs.sh`；
- `config/5GC/adrfcfg.yaml`；
- `daisy/daisyconfig.json`；
- `NWDAF/run_script/run_adrf.sh`、`run_daisy.sh`；
- gNB／UPF 的 run／clean trace scripts；
- nested `adrf/` repository。

`5GC/out/`、PID、log 與其他 runtime evidence 也需列入 inventory，但是否長期保存
應另行分類，不能與可重建設定放在同一層。

## 10. 建議的 Clean Rebuild 保存層級

### A. 必須先保存：site-specific machine profile

- host interface、IPv4、gateway；
- Vagrant bridge 選擇；
- 私有網段 mapping；
- 每台 VM 的 bridged NIC address；
- guest default route 與 NAT／iptables 行為；
- host MongoDB port／dbpath／unit，以及修復後的可用性證據。

### B. 必須另行 review：舊實驗行為

- two-path slice／UE group mapping；
- EES period、replay dataset 與 trace scripts；
- 舊 NWDAF／ML／Daisy／ADRF integration；
- submodule revisions 與自建 run scripts。

這些可能提供遷移線索，但不能直接套到新版三 NWDAF 架構。

### C. 可重建或可歸檔：runtime artifacts

- PID；
- build output；
- 可由 source 重新產生的 binary；
- 已有正式報告保存結論的重複 log。

刪除前仍需逐項列出，不能只按目錄名稱批次移除。

## 11. 重建前尚待完成的驗證

1. 診斷並決定是否保留 `mongod-27018` 方案。
2. 在 VM 啟動後記錄五台 guest 的 `ip -br addr`、`ip route`、interface order 與
   iptables；目前 `enp0s3`、`enp0s8`、`enp0s10` 只由 script 推定。
3. 確認 `192.168.123/125/126/127/128/129/220.1` gateway 的實驗室實際提供者與
   reachability。
4. 進行 IP conflict 檢查，確認 `+20` 網段仍可用。
5. 驗證 N2、N3、N4、EES、MongoDB 與 UE pool 的連通性。
6. 確認未使用 I-UPF／PSA-UPF／MEC 設定是否要歸檔，而不是遷移。
7. 依 `nwdaf-docs` 的新版 full-core profile，重新決定 VM／process layout、two TAI、
   A／B／C NWDAF address 與所需 editable repositories。
8. 產生完整 tracked patch、untracked archive 與 recovery manifest，交由使用者確認
   後才可移除舊 `5G_Infrastructure/`。

## 12. 本次未做的動作

本次只新增／修正 `testbed-docs` 文件。沒有修改 `5G_Infrastructure`，沒有啟動
MongoDB 或 VM，也沒有清除任何舊檔案。
