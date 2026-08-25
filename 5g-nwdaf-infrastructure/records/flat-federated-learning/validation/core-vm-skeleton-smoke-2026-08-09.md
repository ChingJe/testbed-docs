# Core VM Skeleton Smoke — 2026-08-09

本報告記錄新版 `5G_NWDAF_Infrastructure` 第一台 VM 的 bounded skeleton smoke。範圍只包含
box import、VM resources、NAT、host-only network、SSH、source snapshot 與 dynamic disk；
使用 `--no-provision`，沒有安裝 package、編譯 NF 或啟動 5G service。

## Baseline

- Box：`ubuntu/jammy64` `20241002.0.0`，Host 已有 cache
- Guest：Ubuntu 22.04.5 LTS，kernel 5.15.0-171-generic
- Provider：VirtualBox 6.1.50
- VM：`5g-nwdaf-core`，4096 MiB RAM、4 vCPU、40 GiB dynamic VMDK
- Host allowlist：保留 `192.168.33.0/24`，testbed 範圍為 `192.168.56.0/21`

## 發現與修正

第一次 import 因 Host allowlist 只涵蓋 `192.168.56.0/24`，在配置 `192.168.57.2` 前失敗。
使用者擴充 allowlist 後 boot 與四張 host-only adapter 成功。

第一次 source sync 又發現 repository 內的 Host `.venv` 未被排除。同步在約 3.6 GiB 時中止；
雖然 guest 檔案可刪除，VMDK backing file 已膨脹到約 5.5 GiB。Vagrant 預設 `/vagrant`
shared folder 也讓 guest 直接看到 Host working tree。這台 VM 尚未 provision、沒有資料或
snapshot；取得使用者同意後銷毀並乾淨重建，回收 expanded disk。

最終 definition：

- rsync 排除 `ML/`、`.venv`、`__pycache__`、pytest cache 與 `node_modules`；
- 停用預設 `/vagrant` share；
- guest provisioning／service dispatch 移除 PyAnLF／PyMTLF 舊路徑；
- Host repository 與 Host Docker ML build context 不受 guest exclusions 影響。

## 最終驗證

| 項目 | 結果 |
| --- | --- |
| Vagrant state | `core running` |
| SSH | pass |
| Guest addresses | `10.0.2.15`, `192.168.56.10`, `192.168.57.2`, `192.168.58.2`, `192.168.61.2` |
| Host → guest | 四個 host-only IP ping pass |
| Guest → Host | `192.168.57.1` ping pass |
| Source snapshot | 106 MiB |
| Guest `ML/` | absent |
| `/vagrant` vboxsf | absent |
| Guest root usage | 約 1.7 GiB / 39 GiB |
| VMDK on-disk size | 約 1.6 GiB |

驗證後 Host 約有 26 GiB `MemAvailable`、179 GiB workspace free；swap 仍只剩約 1 MiB，維持
warning。Core 保持 running 以供下一個 Path skeleton 做跨 VM network smoke。

## 尚未驗證

- Guest provisioning 與 apt dependency
- MongoDB 8.0 Jammy installation
- Go toolchain 與 Core NF builds
- config activation、systemd units 與 service startup
- Path A/B、N3/N6、gtp5g 與 UERANSIM
- full-core traffic、subscription 與 federated training
