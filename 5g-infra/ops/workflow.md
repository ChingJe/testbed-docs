# 5G Testbed 操作流程

## MongoDB（本機）

自己的 MongoDB 跑在 port 27018（避免與其他使用者的 27017 衝突），已設定開機自啟動。

```bash
# 查看狀態
sudo systemctl status mongod-27018

# 手動啟動 / 停止
sudo systemctl start mongod-27018
sudo systemctl stop mongod-27018

# 連進去管理
mongosh mongodb://127.0.0.1:27018
```

VM 內透過 `10.0.2.2:27018` 連線（VirtualBox NAT）。

---

## 前置：git pull 後

```bash
.agent/setup.sh
```

自動 patch：bridge → `enp2s0`、IP shift +20、MongoDB URL → `10.0.2.2:27018`

### 更新 submodule

Submodule 在 parent repo 裡記錄的是「應停在哪個 commit」，而不是 branch。
checkout 後是 detached HEAD，**不能直接在 submodule 裡 `git pull`**（沒有 branch，不知道 merge 進哪裡）。

正確做法（從 repo 根目錄）：

```bash
# 更新單一 submodule 到 remote 最新
git submodule update --remote NWDAF/NWDAF
git submodule update --remote NWDAF/NWDAF-ML-Service
git submodule update --remote go-upf-ess/go-upf
# ... 依需要選擇
```

更新完 submodule 後，接著依照下方「更新 Submodule（同步到 VM）」章節把新版本同步進 VM 並重新編譯。

---

## 啟動 VM

**第一次（provision）：**
```bash
./deploy.sh 5GC UPF-EES UPF-EES2 gNB gNB2
```

**之後每次：**
```bash
for d in 5GC UPF-EES UPF-EES2 gNB gNB2; do cd $d && vagrant up && cd ..; done
```

確認全部 running：
```bash
vagrant global-status
```

---

## 啟動服務（依序）

### 1. 5GC
```bash
cd 5GC && vagrant ssh
cd ~/free5gc && ./run.sh
```

### 2. UPF-EES / UPF-EES2（SMF 起來後）
```bash
cd UPF-EES && vagrant ssh
cd ~/free5gc && ./run.sh
```

> **若出現 `upf: command not found`**：binary 未編譯，執行：
> ```bash
> cd ~/free5gc
> git submodule update --init --recursive
> make upf
> ```
```bash
cd UPF-EES2 && vagrant ssh
cd ~/free5gc && ./run.sh
```

### 3. gNB / gNB2（AMF 就緒後）
```bash
cd gNB && vagrant ssh
cd ~/UERANSIM && ./run.sh
```
```bash
cd gNB2 && vagrant ssh
cd ~/UERANSIM && ./run.sh
```

gNB 起來 2 秒後自動啟動 ue1~3，gNB2 啟動 ue4~6。

---

## 確認服務狀態

UPF / gNB 在背景執行，確認方式：

```bash
# 確認 process 是否存活
ps aux | grep upf
ps aux | grep nr-gnb

# 看 log
tail -f ~/free5gc/log/upf.log        # UPF
tail -20 ~/free5gc/log/upf.log       # 最後幾行
```

---

## 關閉 / 重啟

```bash
# 關閉單一 VM
cd 5GC && vagrant halt

# 關閉全部
for d in 5GC UPF-EES UPF-EES2 gNB gNB2; do cd $d && vagrant halt && cd ..; done

# 刪除全部 VM（重新 provision 用）
for d in 5GC UPF-EES UPF-EES2 gNB gNB2; do cd $d && vagrant destroy -f && cd ..; done
```

**重建單一元件**（destroy + provision + setup 一次完成）：
```bash
cd <元件> && vagrant destroy -f && cd .. && ./deploy.sh <元件>
# 例如：
cd UPF-EES2 && vagrant destroy -f && cd .. && ./deploy.sh UPF-EES2
```

---

## tmux 速查

```bash
tmux new -s 5g        # 建立 session
tmux attach -t 5g     # 重新接回
tmux ls               # 列出所有 session
```

| 快捷鍵 | 功能 |
|--------|------|
| `Ctrl+B %` | 左右分割 |
| `Ctrl+B "` | 上下分割 |
| `Ctrl+B 方向鍵` | 切換 pane |
| `Ctrl+B z` | 放大/縮小當前 pane |
| `Ctrl+B c` | 新視窗 |
| `Ctrl+B 數字` | 切換視窗 |
| `Ctrl+B d` | detach（離開但繼續跑） |

---

## VM 內編譯（需要網路時）

`setup.sh` 會把 VM 的 default route 改成 bridge gateway，覆蓋掉 VirtualBox NAT，導致 VM 無法連外網。

**需要在 VM 內重新編譯**（`make`、`go build` 等）時，先暫時加回 NAT route：

```bash
sudo ip route del default
sudo ip route add default via 10.0.2.2 dev enp0s3
```

編譯完成後不需要手動移除，VM 重啟後 setup.sh 會自動恢復正確 route。

> 只改 config 檔重啟服務不需要此步驟。

---

## UE 流量測試

UE 成功 register 後，UERANSIM 會建立 TUN 介面（uesimtun0、uesimtun1...），透過它送流量：

```bash
# 確認介面存在
ip addr show | grep uesimtun

# ping（持續發送）
ping -I uesimtun0 8.8.8.8

# 加快頻率
ping -I uesimtun0 8.8.8.8 -i 0.2
```

確認 GTP 流量有進到 UPF：
```bash
# 在 UPF-EES VM
sudo tcpdump -i enp0s8 udp port 2152 -n   # N3 interface（gNB1 → UPF-EES）

# 在 UPF-EES2 VM
sudo tcpdump -i enp0s8 udp port 2152 -n   # N3 interface（gNB2 → UPF-EES2）
```

> 注意：gNB1 的 UE 走 UPF-EES（N3: 192.168.123.10），gNB2 的 UE 走 UPF-EES2（N3: 192.168.128.11）

---

## WebConsole

瀏覽器開 `http://192.168.125.5:5000`（或對應 5GC VM 的 IP）。

**首次啟動（或 git pull 後）需先 build 前端：**

```bash
cd ~/free5gc/webconsole
sudo corepack enable   # 一次性，啟用 Node.js corepack（需 sudo）
./run.sh               # 自動安裝 yarn 4.1.0 並 build 前端
```

> 若出現 `yarn: command not found` 或 `packageManager yarn@4.1.0` 版本錯誤，代表 corepack 還沒 enable。

之後每次重啟只需：

```bash
cd ~/free5gc/webconsole && ./run.sh
```

---

## 更新 Submodule（同步到 VM）

### 更新流程原理

host 端 `git submodule update` 後，VirtualBox synced_folder 會自動反映最新內容（`/go-upf`、`/NWDAF`、`/daisy` 等掛載點即時更新）。但 VM 內的**工作目錄是 provision 時 `cp -r` 建立的副本**，synced_folder 的變更不會自動套用進去，需手動同步。

**通用步驟（適用任何元件）：**

1. **停止 process**：先 pgrep 確認是否在跑，有才 pkill（見下方通用原則）
2. **刪除舊工作目錄**：`rm -rf ~/<目標目錄>`
3. **從 synced_folder 複製**：`cp -r /<掛載點>/<子目錄> ~/<目標目錄>`
   - 若 synced_folder 直接就是目標內容（如 `/daisy/daisy`），複製那一層
   - 若有 `.git` 殘留（submodule 帶進來的），`rm -f <目標>/.git`
4. **重新編譯（僅 Go binary）**：
   - 非互動式 shell 需手動帶入 Go PATH：`PATH=/usr/local/go/bin:$PATH make ...`
   - Python 不需編譯，直接進下一步
5. **重新套用 config**：`/vagrant/setup.sh`（會覆寫 IP、路由等設定，每次同步後都要跑）
6. **重啟服務**

**判斷元件屬於哪種 synced_folder：**

| 掛載點 | 包含子目錄 | 適用 VM |
|--------|-----------|---------|
| `/go-upf` | 即為 UPF source | UPF-EES, UPF-EES2 |
| `/NWDAF` | `NWDAF/`、`NWDAF-ML-Service/`、`nwdaf_uecomm_consumer/`、`run_script/` | 5GC |
| `/daisy` | `daisy/` | 5GC |

**沒有寫在下方的元件**，只要確認它的 synced_folder 掛載點、目標工作目錄、是否需要編譯，依上述步驟推導即可。

---

### UPF（UPF-EES / UPF-EES2 各做一次）

以下指令需**在 VM 內互動式 shell 執行**（`vagrant ssh` 進去後再跑）：

```bash
# 若 UPF 有在跑，先停止（沒跑的話直接跳過）
pgrep -f "bin/upf" && sudo pkill -f "bin/upf"; true

# 同步原始碼並重新編譯
cd ~/free5gc
rm -rf NFs/upf
cp -r /go-upf NFs/upf
rm -f NFs/upf/.git
make upf

# 重新套用 config
/vagrant/setup.sh
```

> 若 `make upf` 需要下載新依賴，先加回 NAT route：
> ```bash
> sudo ip route add default via 10.0.2.2 dev enp0s3
> ```
> 編譯完後 VM 重啟 setup.sh 會自動還原。

**Claude 用（從對應元件目錄執行）：**

```bash
vagrant ssh -c "pgrep -f 'bin/upf'"
vagrant ssh -c "sudo pkill -f 'bin/upf'"   # 有輸出才執行
vagrant ssh -c "cd ~/free5gc && rm -rf NFs/upf && cp -r /go-upf NFs/upf && rm -f NFs/upf/.git"
vagrant ssh -c "PATH=/usr/local/go/bin:\$PATH make -C ~/free5gc upf"
vagrant ssh -c "/vagrant/setup.sh"
```

**驗證（更新後確認兩台 VM 原始碼與 host 一致）：**

先查出此次更新實際變更了哪些檔案：
```bash
# 從 5G_Infrastructure/go-upf-ess/go-upf 目錄，<old>..<new> 換成實際 commit hash
git diff <old>..<new> --name-only
```

再對有變更的檔案做 checksum 比對（以 `aggregator.go` 為例）：
```bash
# host
md5sum go-upf-ess/go-upf/internal/ees/aggregator.go

# UPF-EES（從 5G_Infrastructure/UPF-EES 目錄）
vagrant ssh -c "md5sum ~/free5gc/NFs/upf/internal/ees/aggregator.go"

# UPF-EES2（從 5G_Infrastructure/UPF-EES2 目錄）
vagrant ssh -c "md5sum ~/free5gc/NFs/upf/internal/ees/aggregator.go"
```

三邊 hash 一致即更新成功。

---

### NWDAF（5GC VM 內）

> **透過 `vagrant ssh -c` 停止 process（通用）**：
> `pkill ... || true` 在非互動式 shell 中，process 不存在時 exit 1，vagrant 回傳 255，
> 無法區分「沒在跑」和「SSH 失敗」。**務必兩步驟**：
> ```bash
> vagrant ssh -c "pgrep -f '<pattern>'"          # 有輸出才繼續
> vagrant ssh -c "pkill -f '<pattern>'"          # 無輸出則跳過
> ```
> 此規則適用於所有元件，不限於本節。

**NWDAF 主程式**（Go binary，需重新編譯）：
```bash
pkill -f "nwdaf" || true
rm -rf ~/NWDAF
cp -r /NWDAF/NWDAF ~/NWDAF
cp /NWDAF/run_script/run_nwdaf.sh ~/NWDAF/run_nwdaf.sh   # run script 不在 submodule 裡，需手動補回
cd ~/NWDAF
make build
./run_nwdaf.sh
```

**Claude 用（從 5GC 目錄）：**
```bash
vagrant ssh -c "pgrep -f 'nwdaf'"
vagrant ssh -c "pkill -f 'nwdaf'"   # 有輸出才執行
vagrant ssh -c "rm -rf ~/NWDAF && cp -r /NWDAF/NWDAF ~/NWDAF && cp /NWDAF/run_script/run_nwdaf.sh ~/NWDAF/run_nwdaf.sh"
vagrant ssh -c "PATH=/usr/local/go/bin:\$PATH make -C ~/NWDAF build"
# 重啟 NWDAF
```

---

**NWDAF-ML-Service**（Python，不需編譯）：
```bash
pkill -f "NWDAF-ML-Service" || true
rm -rf ~/NWDAF-ML-Service
cp -r /NWDAF/NWDAF-ML-Service ~/NWDAF-ML-Service
cp /NWDAF/run_script/run_ml_service.sh ~/NWDAF-ML-Service/run_ml_service.sh   # run script 不在 submodule 裡，需手動補回
cp /config/nwdafMLcfg.yaml ~/NWDAF-ML-Service/config/config.yaml              # 本地 config（bind IP）覆蓋 submodule 預設值
# 重啟 ML Service
```

**Claude 用（從 5GC 目錄）：**
```bash
vagrant ssh -c "pgrep -f 'NWDAF-ML-Service'"
vagrant ssh -c "pkill -f 'NWDAF-ML-Service'"   # 有輸出才執行
vagrant ssh -c "rm -rf ~/NWDAF-ML-Service && cp -r /NWDAF/NWDAF-ML-Service ~/NWDAF-ML-Service && cp /NWDAF/run_script/run_ml_service.sh ~/NWDAF-ML-Service/run_ml_service.sh && cp /config/nwdafMLcfg.yaml ~/NWDAF-ML-Service/config/config.yaml"
# 重啟 ML Service
```

---

**nwdaf_uecomm_consumer**（Go binary，需重新編譯）：
```bash
pkill -f "nwdaf_uecomm_consumer" || true
rm -rf ~/nwdaf_uecomm_consumer
cp -r /NWDAF/nwdaf_uecomm_consumer ~/nwdaf_uecomm_consumer
cd ~/nwdaf_uecomm_consumer
go build ./...   # 或依該目錄的 Makefile
# 重啟
```

**Claude 用（從 5GC 目錄）：**
```bash
vagrant ssh -c "pgrep -f 'nwdaf_uecomm_consumer'"
vagrant ssh -c "pkill -f 'nwdaf_uecomm_consumer'"   # 有輸出才執行
vagrant ssh -c "rm -rf ~/nwdaf_uecomm_consumer && cp -r /NWDAF/nwdaf_uecomm_consumer ~/nwdaf_uecomm_consumer"
vagrant ssh -c "PATH=/usr/local/go/bin:\$PATH cd ~/nwdaf_uecomm_consumer && go build ./..."
# 重啟
```

---

**Daisy**（Python，不需編譯，直接用 synced_folder）：
```bash
pkill -f "daisy" || true
rm -rf ~/daisy
cp -r /daisy/daisy ~/daisy
# 重啟 daisy（例如 07 example）
```

**Claude 用（從 5GC 目錄）：**
```bash
vagrant ssh -c "pgrep -f 'python.*daisy'"
vagrant ssh -c "pkill -f 'python.*daisy'"   # 有輸出才執行
vagrant ssh -c "rm -rf ~/daisy && cp -r /daisy/daisy ~/daisy"
# 重啟（依當次使用的 example 決定）
```

---

## 常見問題

詳細內容見 [troubleshooting.md](troubleshooting.md)。

- 5GC VM 重啟後 /NWDAF、/daisy 等 synced folder 沒有掛載
- UE ping 不通（某組 gNB/UPF）
- UE 無法 register（authentication 失敗）
- 新增 subscriber 出現 `duplicate gpsi`
- UPF 啟動失敗：open Gtp5g: cannot allocate memory
- upf: command not found
- WebConsole 前端 404 / yarn 版本錯誤

---

## 常用 Vagrant 指令

```bash
vagrant up          # 啟動 VM
vagrant halt        # 關閉 VM
vagrant destroy -f  # 刪除 VM
vagrant ssh         # SSH 進入 VM
vagrant status      # 查看當前目錄 VM 狀態
vagrant global-status  # 查看所有 VM 狀態
```
