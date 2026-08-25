# Persistent Netplan Alias Migration（2026-08-12）

## 結論

Core、Path A、Path B 的 process aliases 已從 boot-time `ip address add` 遷移為
Netplan／systemd-networkd 持久設定。Vagrant 繼續擁有 `50-vagrant.yaml` 的 VM adapters 與
base anchors；infrastructure 依 active config 產生獨立的
`60-5g-nwdaf-aliases.yaml`。三台 VM 從 poweroff 直接 `vagrant up --no-provision` 後，不需
Make wrapper、post-up hook 或 network reconciler 即可恢復全部 aliases。

本輪沒有修改 NF／ML source、component revision、Vagrant adapter topology 或 public IP plan。
驗證結束後 23 個 Guest services 全部停止，三台 VM 均為 poweroff；Host ML containers 與
subscriptions 全程沒有啟動。

## Runtime identity

- Infrastructure feature：`a792a1f feat(network): persist topology aliases with Netplan`
- Pre-install safety：`61bfc5b fix(network): reject default-route aliases before install`
- Rollback convergence：`7699574 fix(network): verify runtime convergence after rollback`
- Documentation plan：`b1f0d3e docs(infra): plan persistent Netplan alias ownership`
- Vagrant：2.4.3
- Guest：Ubuntu 22.04、Netplan 0.106.1、systemd-networkd
- Provider：VirtualBox
- Effective config：`config/default`，config SHA-256
  `6fad76decad6a7ebfd5a324b86bcff6a6539b0628e706e7dd4c9fc8dad17d55d`

## 原始 race 實證

舊實作的 Core cold boot 時序如下：

- `17:13:25 UTC`：`5g-nwdaf-network.service` 成功以 runtime command 加入 14 個 aliases；
- `17:13:31 UTC`：Vagrant 覆寫 `/etc/netplan/50-vagrant.yaml`；
- `17:13:32`–`17:13:33 UTC`：networkd reconfigure 四張 host-only interfaces；
- 最終只剩 `.56.10`、`.57.2`、`.58.2`、`.61.2` anchors，但 oneshot 仍顯示
  `active (exited)`。

因此故障不是 reset 或 Makefile 本身造成，而是舊 aliases 不屬於 Netplan desired state，會在
Vagrant 最後一次 `netplan apply` 被清除。

## Ownership 與 migration

新實作維持三層責任：

| Owner | 管理內容 |
| --- | --- |
| Vagrant／`50-vagrant.yaml` | VM adapters、host-only segments、base anchors |
| Active `network/<role>.yaml` | 該 role 應存在的 NF／database／consumer aliases |
| Infrastructure `60-5g-nwdaf-aliases.yaml` | 將 aliases 映射到 anchor 所在實際介面 |
| Netplan／networkd | anchor 與 aliases 的 effective persistent addresses |

Guest renderer 會驗證 role、schema、IPv4 subnet、duplicate address、anchor uniqueness、既有
cross-interface collision 與 default-route interface safety。候選 fragment 先在隔離 root 執行
`netplan generate`，再原子安裝；套用只 reload networkd 並 reconfigure 受影響的 host-only
interfaces。`5g-nwdaf-network.service` 現為 static on-demand reconciler，不再 boot-enable。

三台 migration 後均確認：

- fragment mode 為 `0600 root:root`；
- 舊 `/run/5g-nwdaf-infrastructure/network-aliases` 已移除；
- Core 具有 14 個 process aliases；
- Path A／B 各具有 7 個 process aliases；
- `network-setup --verify` 全部通過。

## Cold boot 與 reset-before-services

三台 VM 先 graceful halt，再直接執行 `vagrant up --no-provision`。每台本次 boot 均符合：

- `5g-nwdaf-network.service` 為 `static`、`inactive`；
- boot journal 沒有該 unit 的執行紀錄；
- Core 14、Path A 7、Path B 7 aliases 全部存在並通過 verify。

Core cold boot 後未先執行 `services-start` 或手動 restart network service，即可直接執行
read-only experiment reset plan。MongoDB alias `192.168.57.18` 可正常 bind／連線，並讀到既有
ADRF／NRF scoped state；plan 沒有刪除資料。

## Drift 與 rollback

Drift test 暫時從 Core `enp0s9` 移除 `192.168.57.18/24`：

1. `network-setup --verify` 正確拒絕 missing alias；
2. on-demand reconciler 恢復 alias；
3. 第二次 verify 通過。

Rollback test 使用 `/tmp` failure fixture，只在該測試程序攔截 candidate 的
`networkctl reconfigure`：candidate 已原子安裝後被強制失敗，script 恢復原 fragment SHA，使用
可信任的 system networkctl reconfigure 舊設定，並等待 runtime addresses 收斂。原 fragment
與 14 aliases 最終同時通過 verify；fixture 隨後移除。

第一次 failure injection 顯示只恢復 fragment、沒有驗證 runtime convergence 不足，因此另以
`7699574` 補上 prior persistent／legacy alias reconciliation、rollback convergence check 與
manual-repair failure reporting，再由相同 injection 通過回歸。

## Guest stack smoke 與 teardown

`services-start` 在三台都先輸出 `NETWORK VERIFY`，沒有輸出 `NETWORK RECONCILE`；gtp5g
vermagic／loaded gate 通過，Core 11、Path A 6、Path B 6，合計 23 個 Guest units 均 active。
NF systemd dependency 觸發 on-demand network unit 時，三台都回報 `state=unchanged`，沒有重新
設定網卡。

本輪只驗證 network migration 與既有 Guest stack 相容性，不啟動 Host ML containers、consumer
或 subscriptions，也不宣稱新的 FL business acceptance。完成後 23 個 units 全部確認 inactive，
三台再次通過 alias verify，再 graceful halt 回到 poweroff。最終 Host 約有 30 GiB available
RAM、222 GiB workspace free。

Vagrant 所產生的 `50-vagrant.yaml` mode 為 `0644`，Jammy Netplan 會提示 permissions warning；
該檔案仍由 Vagrant 擁有，warning 未影響 merge、cold boot 或 runtime gate。本輪沒有為消除提示
而接管或改寫 Vagrant 的檔案。
