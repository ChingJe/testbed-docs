# 5G Infrastructure — Environment & Setup

> 2026-08-05 已重新盤點此機器的 local network settings。完整結果、限制與
> clean rebuild 保存清單請見
> [`local-network-settings-inventory-2026-08-05.md`](../reports/local-network-settings-inventory-2026-08-05.md)。
> 本頁描述舊 testbed profile，不代表新版 NWDAF 架構。

## IP Shift +20

實驗室有其他人在跑相同的 testbed，為了避免 IP 衝突，需要將所有 subnet 的第三段 +20。

### 對應表

| 原本 | 改成 |
|------|------|
| `192.168.103.x` | `192.168.123.x` |
| `192.168.105.x` | `192.168.125.x` |
| `192.168.106.x` | `192.168.126.x` |
| `192.168.107.x` | `192.168.127.x` |
| `192.168.108.x` | `192.168.128.x` |
| `192.168.109.x` | `192.168.129.x` |
| `192.168.110.x` | `192.168.130.x` |
| `192.168.200.x` | `192.168.220.x` |

### 需要修改的檔案

**Vagrantfile（11 個）：**
- `5GC/Vagrantfile`
- `I-UPF/Vagrantfile`
- `I-UPF2/Vagrantfile`
- `PSA-UPF/Vagrantfile`
- `PSA-UPF2/Vagrantfile`
- `gNB/Vagrantfile`
- `gNB2/Vagrantfile`
- `MEC/Vagrantfile`
- `MEC2/Vagrantfile`
- `UPF-EES/Vagrantfile`
- `UPF-EES2/Vagrantfile`

**config 設定檔：**
- `config/5GC/` 底下所有 yaml（amfcfg、smfcfg、upfcfg、uerouting 等）
- `config/UERAN/` 底下所有 yaml（gnb、ue 等）
- `setup.sh`（各元件目錄下，含 gateway route 設定）

### 注意事項

- 改完 Vagrantfile 後，config yaml 裡對應的 IP 也要一起改，否則 NF 之間無法連線
- `setup.sh` 裡的 `ip route add default via <gateway>` 也要對應更新
- bridge 介面：2026-08-05 實機確認為 `enp2s0`；Wi-Fi `wlp0s20f3` 目前 DOWN

## MongoDB

- 舊 profile 預期自己的 MongoDB 跑在本機 **port 27018**，避免和 27017 衝突
- systemd service `mongod-27018` 目前 enabled，但 2026-08-05 唯讀檢查為
  **failed**，27018 沒有 listener；不可再假設它已正常啟動
- 27017 目前有 listener；本次未確認其 owner、資料或 authentication policy
- dbpath：`/var/lib/mongodb-27018`
- VM 透過 VirtualBox NAT 連本機，host IP 在 VM 內永遠是 `10.0.2.2`

### 需要改 MongoDB URL 的 config 檔（改為 `mongodb://10.0.2.2:27018`）

| 檔案 | key |
|------|-----|
| `config/5GC/nrfcfg.yaml` | `MongoDBUrl` |
| `config/5GC/webuicfg.yaml` | `mongodb.url` |
| `config/5GC/udrcfg.yaml` | `mongodb.url` |
| `config/5GC/pcfcfg.yaml` | `mongodb.url` |
| `config/5GC/chfcfg.yaml` | `mongodb.url` |
| `config/5GC/nwdafcfg.yaml` | `mongodb.url` |

### 注意：git pull 會覆蓋這些檔案

`config/5GC/*.yaml` 和所有 `Vagrantfile` 都被 git 追蹤，pull 後本地修改會消失。
**pull 完後執行 `.agent/setup.sh` 重新套用所有本地設定。**

## .agent/ 目錄

存放輔助腳本，**不被 git 追蹤**（透過 git exclude 排除）。

- `.agent/setup.sh` — git pull 後重新套用本地設定（bridge、IP shift、MongoDB URL）
- `.agent/clean_logs.sh` — 清除 5GC / UPF VM 上累積的 log

操作流程與 troubleshooting 文件已移至 [`ops/`](./)：
- [`workflow.md`](workflow.md)
- [`troubleshooting.md`](troubleshooting.md)

## .agent/setup.sh

`git pull` 後執行此腳本，自動套用以下所有本地設定：

1. **bridge 介面**：所有 Vagrantfile 的 `enp6s0` → `enp2s0`
2. **IP shift +20**：所有 Vagrantfile 和 `config/` yaml 的 subnet 第三段 +20
3. **MongoDB URL**：所有 yaml 的 MongoDB URL → `mongodb://10.0.2.2:27018`

**冪等性**：已 shift 過的 IP 不會再被 shift，可以安全重複執行。
但 `git pull` 會還原所有修改，所以每次 pull 後都要跑一次。

> `.agent/setup.sh` 只重放固定的 bridge、IP mapping、Mongo URL 與 submodule
> copy cleanup。它不會重建 UPF 的 NAT default route、ADRF synced folder、
> untracked config／run scripts、submodule revisions 或 NWDAF／Daisy 行為變更。
> 不可將它視為完整 backup。

## 網路環境

| 環境 | bridge 介面 | 用途 |
|------|------------|------|
| 有線（固定 IP `140.113.110.77`） | `enp2s0` | 目前設定 |
| WiFi | `wlp0s20f3` | DOWN，不使用 |

## Provision 流程

只用到以下五個元件：

```bash
./deploy.sh 5GC
./deploy.sh UPF-EES
./deploy.sh UPF-EES2
./deploy.sh gNB
./deploy.sh gNB2
```

或一次全部啟動：

```bash
./deploy.sh 5GC UPF-EES UPF-EES2 gNB gNB2
```

`I-UPF`、`I-UPF2`、`PSA-UPF`、`PSA-UPF2`、`MEC`、`MEC2` 不使用。
