# 5G Testbed Troubleshooting

## 5g-viz 顯示 `PermissionDenied for user vagrant on host 127.0.0.1`

**症狀**：`./start.sh` 啟動 `5g-viz` 後，stderr 持續出現：

```text
[collector] 5gc SSH error: PermissionDenied: Permission denied for user vagrant on host 127.0.0.1, retry in 5s
```

手動測試 `ssh -i ... -p 2222 vagrant@127.0.0.1` 也可能失敗。

**根本原因**：`5g-viz` 的 `profiles/default/.env` 把 `SSH_5GC_PORT` 寫死成 `2222`，但 Vagrant 的 SSH forwarded port 並不固定。當本機同時有多台 Vagrant VM 運行時，`2222` 可能被其他 VM 先占用，5GC 這次實際被分配到其他 port（例如 `2203`）。

也就是說，失敗原因通常不是 private key 錯，而是 **連到了錯的 VM port**。

**確認方式**：

```bash
cd ~/testbed/5G_Infrastructure/5GC
vagrant ssh-config
```

查看：

- `HostName`
- `Port`
- `User`
- `IdentityFile`

例如：

```text
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2203
  IdentityFile /home/chingje/testbed/5G_Infrastructure/5GC/.vagrant/machines/default/virtualbox/private_key
```

這表示 `5g-viz` 應該使用 `2203`，不是 `2222`。

**手動驗證**：

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  -i /home/chingje/testbed/5G_Infrastructure/5GC/.vagrant/machines/default/virtualbox/private_key \
  -p 2203 vagrant@127.0.0.1
```

若能登入，代表 SSH key 沒問題，只是原本 port 設錯。

**修正方式**：

編輯 `5g-viz/profiles/default/.env`：

```env
SSH_5GC_HOST=127.0.0.1
SSH_5GC_PORT=2203
SSH_5GC_USER=vagrant
SSH_5GC_KEY=/home/chingje/testbed/5G_Infrastructure/5GC/.vagrant/machines/default/virtualbox/private_key
```

之後重啟 `5g-viz`：

```bash
cd ~/testbed/5g-viz
./start.sh
```

**補充**：

- 若手動 `ssh` 顯示 `REMOTE HOST IDENTIFICATION HAS CHANGED!`，代表該 host port 對應的 VM host key 已變更，先移除舊紀錄：

```bash
ssh-keygen -f "/home/chingje/.ssh/known_hosts" -R "[127.0.0.1]:2203"
```

- `vagrant ssh` 通常不會受這個問題影響，因為它會自帶 `StrictHostKeyChecking no` 與正確的 `IdentityFile`。

## 5GC VM 重啟後 /NWDAF、/daisy 等 synced folder 沒有掛載

**症狀**：`vagrant reload`（或任何重啟 5GC VM 的操作）結束後出現：

```
Vagrant was unable to mount VirtualBox shared folders...
mount -t vboxsf -o uid=1000,gid=1000,_netdev NWDAF /NWDAF
/sbin/mount.vboxsf: shared folder '/home/vagrant/NWDAF' was not found
```

`vagrant reload` 回傳 exit code 1，且 `/NWDAF`、`/daisy` 是空目錄，`/config`、`/adrf` 則正常。

**根本原因**：VirtualBox Guest Additions 的 `mount.vboxsf` 有一個 bug：
當 VM 的 `$HOME`（`/home/vagrant/`）下存在與 shared folder **同名**的目錄時，它會把
shared folder name 誤解析成本地路徑，導致找不到。

- `~/NWDAF` 存在（provision 時 `cp -r /NWDAF/NWDAF ~/NWDAF` 建立）→ `NWDAF` 掛載失敗
- `~/daisy` 存在（setup.sh 用到 `~/daisy/examples/`）→ `daisy` 掛載失敗
- `~/adrf`、`~/config` 不存在 → 正常掛載

> **2026-04-22 更新**：`~/adrf` 目錄後來被建立，導致 `adrf` 也開始失敗。
> 已在 fstab 補上 `Adrf /adrf vboxsf uid=1000,gid=1000,_netdev,nofail 0 0`（見下方）。

**為什麼之前沒遇到**：VM 自 provision 後從未重啟過，synced folder 一直維持開機時掛好的狀態。
第一次執行 `vagrant reload`（例如新增 synced_folder 後）才會觸發此問題。

**已修復方式**（2026-03-26）：

已在 `/etc/fstab` 中加入使用不衝突大小寫的掛載條目：
```
nwdaf /NWDAF vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
Daisy /daisy vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
```
並移除 VBoxAdds 自動寫入的舊條目（`NWDAF /NWDAF`、`daisy /daisy`，沒有 `nofail`，開機失敗會卡 emergency mode）。

**若 fstab 修復失效（重新 provision 後 fstab 被覆寫等情況）**，手動恢復：

```bash
vagrant ssh   # SSH 進 5GC VM

# 確認哪些 folder 沒掛
mount | grep vboxsf

# 手動掛載（用小寫/不衝突的名稱繞過 bug）
sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev nwdaf /NWDAF
sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev Daisy /daisy
sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev vagrant /vagrant

# 重新寫入 fstab（移除舊條目，加入新條目）
sudo sed -i '/^NWDAF \/NWDAF vboxsf/d; /^daisy \/daisy vboxsf/d' /etc/fstab
grep -q 'nwdaf /NWDAF' /etc/fstab || echo 'nwdaf /NWDAF vboxsf uid=1000,gid=1000,_netdev,nofail 0 0' | sudo tee -a /etc/fstab
grep -q 'Daisy /daisy' /etc/fstab || echo 'Daisy /daisy vboxsf uid=1000,gid=1000,_netdev,nofail 0 0' | sudo tee -a /etc/fstab
```

**診斷工具**：

```bash
# 確認 VirtualBox 看到的 shared folder 清單
sudo /usr/sbin/VBoxControl sharedfolder list

# 查看 fstab 中的 vboxsf 條目
grep vboxsf /etc/fstab
```

## vagrant up 失敗後 /vagrant、/config 也沒掛載

**症狀**：`vagrant up` 因 adrf（或其他 folder）掛載失敗而提前退出，SSH 進去後只看到三個 vboxsf mount，缺少 `/vagrant` 和 `/config`。

**原因**：Vagrant 在第一個掛載失敗時就退出，後續的 `/vagrant`、`/config` 來不及處理。這兩個 folder 沒有衝突的 `~/` 目錄，fstab 裡雖然有條目，但時序問題導致開機時也沒掛上。

**解法**：SSH 進去後手動補掛：

```bash
sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev vagrant /vagrant
sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev config /config
```

確認全部 5 個都掛好：

```bash
mount | grep vboxsf
```

> **若 `/NWDAF`、`/daisy`、`/adrf` 也沒掛**：mount point 目錄可能不存在（fstab nofail 靜默失敗）。
> 先建目錄再手動掛：
> ```bash
> sudo mkdir -p /NWDAF /daisy /adrf
> sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev nwdaf /NWDAF
> sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev Daisy /daisy
> sudo mount -t vboxsf -o uid=1000,gid=1000,_netdev Adrf /adrf
> ```

目前 fstab 的完整狀態（5 個全部有 `nofail`）：

```
nwdaf   /NWDAF   vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
Daisy   /daisy   vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
Adrf    /adrf    vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
vagrant /vagrant vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
config  /config  vboxsf uid=1000,gid=1000,_netdev,nofail 0 0
```

---

## UE ping 不通（某組 gNB/UPF）

**症狀**：UE 有 register、uesimtun 存在、GTP 封包有到達 UPF N3 介面，但 upfgtp 完全無流量，ping 100% loss。

**根本原因**：WebConsole subscriber 的 slice charging method 設成 **Online**，但 CHF 沒有正確配置對應的計費規則。SMF 在 PDU session 建立時先把 uplink FAR 設為 BUFF（等 CHF 授予 quota），CHF 不回應 → FAR 永遠停在 BUFF → gtp5g 把封包 buffer 起來不轉發。

UPF log 會出現大量：
```
level="warning" msg="handleSessionReportRequestTimeout: SEID[0xN]"
```

**解法**：WebConsole → 對應 subscriber → Slice 設定 → Charging Method 改成 **Offline**。

---

## UE 無法 register（authentication 失敗）

**症狀**：gNB log 顯示 registration 失敗或 UE 一直重試。

**原因**：WebConsole subscriber 的 Operator Code Type 設錯。UERANSIM config 用 `opType: 'OP'`，但 WebConsole 建立時選了 OPC。

**解法**：刪除 subscriber 重新新增，Operator Code Type 選 **OP**，填入 `op` 欄位的值。

---

## 新增 subscriber 出現 `duplicate gpsi`

**症狀**：PUT/POST subscriber 回傳 `Status: 400, cause: duplicate gpsi`。

**解法**：用 mongosh 清掉全部 subscriber 再重新新增：
```bash
mongosh mongodb://10.0.2.2:27018
use free5gc
db.subscriptionData.provisionedData.amData.deleteMany({})
db.subscriptionData.provisionedData.smData.deleteMany({})
db.subscriptionData.provisionedData.smfSelectionSubscriptionData.deleteMany({})
db.subscriptionData.provisionedData.uecContextInSmfData.deleteMany({})
db.subscriptionData.authenticationData.authenticationSubscription.deleteMany({})
```
然後在 WebConsole 重新建立所有 subscriber。

---

## UPF 啟動失敗：open Gtp5g: cannot allocate memory

**症狀**：UPF 啟動時出現：
```
UPF Cli Run Error: open Gtp5g: open link: create: cannot allocate memory
```
dmesg 顯示：
```
upfgtp:[gtp5g] gtp5g_newlink: Failed to create a hash table
```

**根本原因**：`apt upgrade` 在背景升級了 kernel（例如 5.15.0-171 → 5.15.0-174），但 gtp5g 是 out-of-tree module，不會隨 apt 自動重編。VM 重開機後切換到新 kernel，舊的 `.ko` 無法載入，gtp5g 實際上沒有載入（`lsmod | grep gtp5g` 無輸出），UPF 建立 interface 時直接失敗。

> 注意：錯誤訊息 `cannot allocate memory` 有誤導性——實際原因是模組未載入，不是記憶體不足。

**確認方式**：
```bash
uname -r                    # 確認目前 kernel 版本
lsmod | grep gtp5g          # 若無輸出 → 模組未載入
ls /lib/modules/$(uname -r)/kernel/drivers/net/gtp5g* 2>/dev/null || echo 'no module'
```

**解法**：針對新 kernel 重新編譯 gtp5g（以 v0.9.14 為例）：

```bash
# 確認 headers 已裝
dpkg -l linux-headers-$(uname -r) | grep ^ii

# 加回 NAT route 才能連外網（編譯完後重開 VM 會自動還原）
sudo ip route add default via 10.0.2.2 dev enp0s3

# Clone source（版本對應原本安裝的 gtp5g 版本）
git clone --depth 1 --branch v0.9.14 https://github.com/free5gc/gtp5g.git ~/gtp5g

# 編譯、安裝、載入
cd ~/gtp5g
make
sudo make install
sudo modprobe gtp5g

# 確認載入成功
lsmod | grep gtp5g
```

`make install` 會自動將模組寫入 `/etc/modules-load.d/gtp5g.conf`，之後開機會自動載入。

---

## upf: command not found

**症狀**：UPF-EES 或 UPF-EES2 跑 `./run.sh` 出現 `upf: command not found`。

**原因**：UPF binary 沒有編譯。

**解法**：
```bash
cd ~/free5gc
git submodule update --init --recursive
make upf
```

---

## WebConsole 前端 404 / yarn 版本錯誤

**症狀**：`./run.sh` 出現 `packageManager yarn@4.1.0` 錯誤，前端 build 失敗，瀏覽器開 WebConsole 顯示 404。

**解法**：
```bash
sudo corepack enable
./run.sh
```
