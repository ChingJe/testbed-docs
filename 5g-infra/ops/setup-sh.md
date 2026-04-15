# .agent/setup.sh — Local Patch Reference

`git pull` 會覆蓋 Vagrantfile 和 `config/` yaml，每次 pull 後需執行此腳本重新套用本地設定。

```bash
.agent/setup.sh
```

腳本是冪等的——已套用過的設定不會重複套用，可安全重複執行。

---

## 步驟一：Vagrantfile — bridge 介面 + IP shift

**目標**：將所有 Vagrantfile 的 bridge 介面改為本機網卡，並將 subnet 第三段 +20（避免與實驗室其他 testbed 衝突）。

**掃描範圍**：`find $REPO -name "Vagrantfile"`（所有 11 個元件）

**修改內容**：

| 項目 | 原值 | 改成 |
|------|------|------|
| bridge 介面 | `enp6s0` | `enp2s0` |
| `192.168.103.x` | → | `192.168.123.x` |
| `192.168.105.x` | → | `192.168.125.x` |
| `192.168.106.x` | → | `192.168.126.x` |
| `192.168.107.x` | → | `192.168.127.x` |
| `192.168.108.x` | → | `192.168.128.x` |
| `192.168.109.x` | → | `192.168.129.x` |
| `192.168.110.x` | → | `192.168.130.x` |
| `192.168.200.x` | → | `192.168.220.x` |

**冪等邏輯**：IP shift 只在偵測到原始 `192.168.10x.` 的檔案上執行；已 shift 過的檔案不再處理。

---

## 步驟二：config yaml + component setup.sh + daisy nodes.yaml — IP shift

**目標**：將 NF config 和元件 setup.sh 裡的 IP 一併 shift，讓 NF 之間能正確定址。

**掃描範圍**：

| 路徑 | 說明 |
|------|------|
| `config/**/*.yaml` | AMF / SMF / UPF / NRF / UDR / PCF / CHF / NWDAF config |
| `<component>/setup.sh`（depth ≤ 2，排除 `.agent/`） | 各 VM 的 provision setup，含 `ip route add default via <gateway>` |
| `daisy/**/nodes.yaml` | Daisy 節點 IP 設定 |
| `NWDAF/nwdaf_uecomm_consumer/**/*.json` | consumer 連線設定 |

**修改內容**：同步驟一的 IP 對應表，邏輯相同。

---

## 步驟三：provision.sh — 補 submodule `.git` 清除

**目標**：provision 時 `cp -r` 複製 submodule 目錄後，需刪除殘留的 `.git`，否則 VM 內的工作目錄會被 git 誤認為 nested repo，導致後續 `make` 或路徑解析異常。

**掃描對象**：

| 檔案 | 修改內容 |
|------|----------|
| `UPF-EES/provision.sh` | 在 `cp -r /go-upf ~/free5gc/NFs/upf` 後插入 `rm -f ~/free5gc/NFs/upf/.git` |
| `UPF-EES2/provision.sh` | 同上 |
| `5GC/provision.sh` | 在 `cp -r /vagrant/smf-nwdaf-ext ~/free5gc/NFs/smf` 後插入 `rm -f ~/free5gc/NFs/smf/.git` |

**冪等邏輯**：先檢查 `rm -f .../.git` 是否已存在，存在則跳過。

---

## 步驟四：config yaml — MongoDB URL

**目標**：將所有 NF config 的 MongoDB URL 指向本機 port 27018（避免與其他使用者的 27017 衝突）。

**掃描範圍**：`config/**/*.yaml`

**修改內容**：

```
mongodb://<任意 host>:<任意 port>  →  mongodb://10.0.2.2:27018
```

`10.0.2.2` 是 VirtualBox NAT 固定的 host IP，VM 內永遠可達。

**受影響檔案**：

| 檔案 | key |
|------|-----|
| `config/5GC/nrfcfg.yaml` | `MongoDBUrl` |
| `config/5GC/webuicfg.yaml` | `mongodb.url` |
| `config/5GC/udrcfg.yaml` | `mongodb.url` |
| `config/5GC/pcfcfg.yaml` | `mongodb.url` |
| `config/5GC/chfcfg.yaml` | `mongodb.url` |
| `config/5GC/nwdafcfg.yaml` | `mongodb.url` |

---

## 從零重建

若 `.agent/setup.sh` 遺失，依下列步驟手動重現：

1. **bridge**：所有 Vagrantfile 的 `bridge: "enp6s0"` → `bridge: "enp2s0"`
2. **IP shift**：所有 Vagrantfile、`config/**/*.yaml`、各元件 `setup.sh`、`daisy/**/nodes.yaml`、`NWDAF/nwdaf_uecomm_consumer/**/*.json`，按步驟一的對應表做 sed 取代
3. **provision.sh .git 清除**：在 UPF-EES、UPF-EES2、5GC 的 `provision.sh` 對應 `cp -r` 行後各插一行 `rm -f .../.git`
4. **MongoDB URL**：`config/**/*.yaml` 全域取代 `mongodb://.*` → `mongodb://10.0.2.2:27018`
