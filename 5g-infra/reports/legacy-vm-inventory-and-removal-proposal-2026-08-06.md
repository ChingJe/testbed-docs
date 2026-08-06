# Legacy VM Inventory and Removal Report

日期：2026-08-06

狀態：已完成；使用者明確決定不備份舊 VM，五台 VM 與 orphan state 已永久刪除

## 1. 結論

清點時主機有五台由舊 `5G_Infrastructure` 管理的 VirtualBox VM。Vagrant metadata、
VirtualBox registry 與各 project 的 `.vagrant` UUID 三者一致，五台都被 registry 標為
`poweroff`，而且沒有已登記的 VirtualBox snapshot。

五台已登記 VM 實際占用合計約 **130.68 GiB**。另有一個未登記、只剩 saved-state
的舊 UPF-EES 目錄，占用約 **0.43 GiB**。2026-08-06 依使用者明確授權直接刪除，
不建立 backup。清理完成後 NVMe filesystem 從約 **47.10 GiB available** 增加到
**178.24 GiB available**，高於新版三 VM 計畫暫定的 120 GiB preflight 目標。

離線檢查確認五台 guest 都有 build tree、database、log、pcap、model artifact 或
手動部署內容。使用者確認這些舊 runtime assets 未來不再使用，接受不可恢復的資料
遺失，因此未採用原先的完整 raw backup 建議。舊 `5G_Infrastructure` source working
tree 與本文件保存的設定 inventory 未刪除。

## 2. 盤點方法與限制

本次只進行：

- 讀取 Vagrant global machine index 與各 project `.vagrant` metadata；
- 使用 `vagrant global-status`、`VBoxManage list/showvminfo/showmediuminfo` 交叉確認；
- 讀取 VirtualBox registry、machine `.vbox` 與 storage directory；
- 使用 `7z` 離線讀取 VMDK 中的 ext4 directory／file metadata；
- 量測 host RAM、swap、filesystem 與 `/hdd` 可用空間。

初始 inventory 階段沒有：

- 啟動或登入 VM；
- 執行 `vagrant global-status --prune`；
- 建立 snapshot、export appliance 或複製 VMDK；
- 抽出 guest 檔案、修改 guest filesystem 或修復 journal；
- destroy、unregister、delete 或移動任何 VM／disk；
- 修改舊 `5G_Infrastructure` source repository。

`VBoxManage` 查詢會啟動它自己的 service 並更新 service log，可能刷新 medium container
metadata；本次沒有要求它變更 VM configuration 或 guest filesystem。

VirtualBox 版本是 `6.1.50_Ubuntur161033`，目前缺少 `/dev/vboxdrv`，因此不能啟動 VM。
這不影響 registry、disk 與 offline filesystem inventory。主機沒有 `virsh`，Vagrant
也只登記 `virtualbox` provider；本次沒有發現另一組 libvirt VM inventory。

## 3. Host 資源

| 項目 | 清理前 | 清理後 | 判斷 |
| --- | ---: | ---: | --- |
| RAM | 62 GiB total；約 10 GiB available | 約 19 GiB available | 殘留 VBoxHeadless 退出後回收約 9 GiB available memory |
| Swap | 2 GiB／2 GiB used | 約 2 MiB free | 仍幾乎用滿；新 VM bring-up 前需另查 host memory pressure |
| VM 所在 filesystem | `/dev/nvme1n1p1`，約 47.10 GiB available | 約 178.24 GiB available | 已超過 120 GiB 暫定門檻 |
| 備份候選 filesystem | `/hdd`，約 1.86 TiB free | 未使用 | 使用者決定不備份舊 VM |
| Vagrant boxes | 約 1.2 GiB | 保留 | 不在本次 VM removal 範圍 |
| 舊 source repository | 約 13 GiB | 保留 | 不在本次 VM removal 範圍 |

`/hdd` 是獨立實體 HDD，而 VM 位於 NVMe root filesystem；它原本可作為 backup target，
但本次沒有在 `/hdd` 建立任何 backup directory 或複製 VM data。

## 4. 移除前已登記 VM

所有 main disk 都是 40 GiB dynamic VMDK；表中的占用是 host filesystem 實際 allocated
size，並包含該 machine directory 內的小型 config drive、`.vbox` 與 log。

| Role | Vagrant ID | VirtualBox UUID | VirtualBox machine name | Project directory | Box | RAM / vCPU | State | Snapshot | 實際占用 |
| --- | --- | --- | --- | --- | --- | ---: | --- | --- | ---: |
| `5GC` | `c087afe` | `37610364-2f66-4118-af00-978bf5b98e01` | `5GC_default_1773646725425_1023` | `/home/chingje/testbed/5G_Infrastructure/5GC` | `ubuntu/focal64` `20240821.0.1` | 4 GiB / 2 | poweroff | 無 | 38.20 GiB |
| `UPF-EES` | `041278a` | `c9b60eb0-2941-4fe0-897f-52c5adb83fda` | `UPF-EES_default_1778596845007_91100` | `/home/chingje/testbed/5G_Infrastructure/UPF-EES` | `ubuntu/jammy64` `20241002.0.0` | 1 GiB / 2 | poweroff | 無 | 18.49 GiB |
| `UPF-EES2` | `d332d4b` | `baf436bb-d4ff-44b5-8def-646d30dc5752` | `UPF-EES2_default_1773133780447_88201` | `/home/chingje/testbed/5G_Infrastructure/UPF-EES2` | `ubuntu/jammy64` `20241002.0.0` | 1 GiB / 2 | poweroff | 無 | 36.36 GiB |
| `gNB` | `15e77d6` | `556af05f-d6a4-4160-912a-9ec10c0a7a73` | `gNB_default_1773133047666_6617` | `/home/chingje/testbed/5G_Infrastructure/gNB` | `ubuntu/focal64` `20240821.0.1` | 1 GiB / 2 | poweroff | 無 | 17.09 GiB |
| `gNB2` | `18a993a` | `22167cd1-5094-48e0-a4bb-70ab7b79c451` | `gNB2_default_1773133339160_81122` | `/home/chingje/testbed/5G_Infrastructure/gNB2` | `ubuntu/focal64` `20240821.0.1` | 1 GiB / 2 | poweroff | 無 | 20.54 GiB |

### 4.1 Exact machine directories

- `/home/chingje/VirtualBox VMs/5GC_default_1773646725425_1023`
- `/home/chingje/VirtualBox VMs/UPF-EES_default_1778596845007_91100`
- `/home/chingje/VirtualBox VMs/UPF-EES2_default_1773133780447_88201`
- `/home/chingje/VirtualBox VMs/gNB_default_1773133047666_6617`
- `/home/chingje/VirtualBox VMs/gNB2_default_1773133339160_81122`

每台都有一個 40 GiB-capacity base VMDK 和約 10 MiB-capacity config-drive VMDK。
所有 medium 都顯示 `normal (base)`，沒有 parent／differencing disk。

## 5. 已放棄的 Guest-only state

由於 source 可能先經 synced folder，再被 provisioning copy 到 guest，guest copy 不能
自動視為與 host repository 相同。離線 inventory 至少發現以下內容；使用者已確認不再
使用，並接受它們隨 VM 永久刪除：

### 5.1 `5GC`

- `~/NWDAF`：約 24 MiB；
- `~/NWDAF-ML-Service`：約 6.9 GiB；
- `~/daisy`：約 9.3 GiB；
- `~/adrf`：約 23 MiB；
- `~/free5gc`：約 738 MiB；
- `~/nwdaf_uecomm_consumer`：5 個 script／JSON config；
- `~/adrf.log` 與 `~/nwdaf.log`：合計約 171 MiB；
- Daisy model／artifact／log 相關檔案：約 75 MiB；
- NWDAF-ML model／artifact／log 相關檔案：約 7.8 MiB；
- `/var/lib/mongodb`：約 380 MiB；
- persistent journal：約 3.87 GiB；
- `~/.cache`：約 11.8 GiB，主要應可重建，但 raw backup 會一併保存。

這台包含 database、模型、consumer 與 component logs，是 inventory 中資料遺失風險
最高的 target；使用者評估後仍決定不備份。

### 5.2 `UPF-EES`

- `~/free5gc`、Go build tree 與 cache；
- `~/upfb`：約 63 MiB，包含 UPF-EES source/config、PseudoDriver patch 與兩組
  `training_packets_run001.parquet`；目前 host 有相同名稱的來源與資料，但 guest copy
  仍需靠 backup 或 checksum comparison 才能證明完全相同；
- persistent journal 與 system logs。

### 5.3 `UPF-EES2`

- `~/free5gc`、`~/gtp5g`、Go build tree 與 cache；
- persistent journal 與 system logs。

### 5.4 `gNB`／`gNB2`

- 各自有完整 `~/UERANSIM` source/build tree；
- 各有四組 trace directory、48 個 log／metadata entry 與四個 pcap；
- 各自 persistent journal 約 3.8–4.0 GiB。

pcap 很小，但其實驗 identity 未與 host report 自動比對；inventory 原先建議保存，
最後依使用者決策隨 VM 刪除。

## 6. Orphan saved-state directory

另發現：

```text
/home/chingje/VirtualBox VMs/UPF-EES_default_1773133631581_82160/
└── Snapshots/2026-05-12T13-04-27-908688000Z.sav
```

實際占用約 0.43 GiB。這個 machine name 不在目前 VirtualBox registry、Vagrant global
index 或任何 project `.vagrant` UUID 中；directory 也沒有 `.vbox` 或 VMDK。VirtualBox
舊 log 顯示它曾被啟動，之後留下 saved-state。單獨的 `.sav` 不足以恢復 VM，但仍可能
包含當時 guest memory，因此 inventory 原先建議：

- 不將它當作可由 `vagrant destroy` 管理的 VM；
- backup 時以 orphan artifact 保存並限制讀取權限；
- checksum 驗證後，才以這個 exact directory 作為獨立 removal target。

使用者後續放棄 backup，這個 exact directory 已永久刪除。

## 7. Network side observation

VirtualBox global registry 已存在 `vboxnet0` DHCP：

```text
network: 192.168.56.0/24
host address: 192.168.56.100
DHCP range: 192.168.56.101-192.168.56.254
```

新版 `testbed.yaml` 草案也暫以 `192.168.56.0/24` 作 management network，靜態 VM 位址
為 `.10`–`.12`，目前不落在 DHCP pool。這可以作為 reuse candidate，但不能在 provider
選定前直接視為已核准；preflight 必須確認 host-only adapter、DHCP policy 與其他使用者
VM 沒有衝突。

## 8. 未採用的備份方案

原先建議使用完整 raw directory backup，而不是只挑 guest 檔案。原因是 VirtualBox
driver 尚不可用，且 guest source copy、MongoDB、model artifact 與 logs 的 ownership
還未完全比對。使用者後續明確表示這些 VM 不會再使用，決定直接刪除，因此本節只保留
當時曾評估過但未執行的方案。

原建議目的地（未建立）：

```text
/hdd/chingje/5g-testbed-legacy-vms-2026-08-06/
├── virtualbox-vms/       # 五台 machine directory + orphan directory
├── vagrant-metadata/     # 五個 project 的 .vagrant metadata
├── provider-metadata/    # VirtualBox registry/exported showvminfo
└── manifests/            # file list、size、SHA-256、recovery notes
```

要求：

1. root backup directory mode `0700`，manifest 以外的 raw disk／saved-state 至少 `0600`；
2. copy 時保留 sparse allocation，避免將 40 GiB logical disk 不必要地展開；
3. backup 後對所有 `.vbox`、`.vmdk`、`.sav` 與 Vagrant identity metadata 計算 SHA-256；
4. 重新計算目的地 size，並逐檔驗證 checksum；
5. recovery 文件說明如何將 machine directory copy 回 VirtualBox default machine
   folder 並 register `.vbox`；Vagrant metadata 只作 identity 輔助，不盲目覆寫 global index；
6. 在 checksum 全部通過前，不執行任何 removal。

預估需在 `/hdd` 使用約 131.2 GiB 加 manifest／filesystem overhead；目前約 1.86 TiB
free，容量足夠。由於 `/hdd` 是 HDD，實際 copy 與 SHA-256 時間要在執行時量測，不能
以 NVMe 速度估算。

## 9. 清理決策

### 原先建議備份後移除、後由使用者放棄備份

- `5GC`
- `UPF-EES`
- `UPF-EES2`
- `gNB`
- `gNB2`
- orphan `UPF-EES_default_1773133631581_82160` saved-state directory

### 不在本次範圍

- `/home/chingje/testbed/5G_Infrastructure` source／working tree；
- `/home/chingje/.vagrant.d/boxes`；
- VirtualBox global application config；
- Docker、k3s、其他使用者資料與 `/hdd` 既有 archive；
- host MongoDB data／service。

## 10. 執行結果

使用者明確確認「直接刪除五台 VM 與 orphan saved-state，不做備份」後：

1. 從五個原 Vagrant project 各自執行 `vagrant destroy -f`，五次皆成功回報
   `Destroying VM and associated drives`；
2. 移除未登記的
   `/home/chingje/VirtualBox VMs/UPF-EES_default_1773133631581_82160` exact directory；
3. 發現五個舊 `VBoxHeadless` process 雖然 registry 顯示 VM 為 `poweroff`，實際仍在
   背景執行並持有已刪 VMDK，所以最初 `df` 沒有回收空間；
4. 五個 process 都能以 exact PID `SIGTERM` 正常退出，未使用 `SIGKILL`；
5. 其中 `5GC`、`UPF-EES`、`UPF-EES2` 在退出時又寫回未註冊的 `.vbox` 與 `.sav`
   residual directory；確認 `MachineRegistry` 仍為空且沒有 VMDK 後，再以三個 exact
   path 移除；
6. 最終沒有 broad glob deletion，也沒有處理 source repo、Vagrant boxes、host MongoDB、
   Docker、k3s 或其他使用者資料。

最終驗證：

- Vagrant machine index：`{"version":1,"machines":{}}`；
- VirtualBox registry：`<MachineRegistry/>`；
- `/home/chingje/VirtualBox VMs`：空；
- `VBoxHeadless`：無；
- 舊 `5G_Infrastructure` source working tree：存在，原 tracked／untracked 差異仍保留；
- NVMe：`178.24 GiB available`，使用率約 80%；
- host available RAM：約 19 GiB；swap 仍幾乎用滿。

本次沒有 recovery backup。被刪除的 guest MongoDB、models、logs、pcaps、build tree 與
手動部署內容不可由本機 VM storage 還原。後續可進入新版 repository bootstrap，但在
實際建立 VM 前仍需決定 provider，並處理 VirtualBox `/dev/vboxdrv` 或評估 libvirt。
