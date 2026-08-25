# Hybrid Host Readiness Inventory — 2026-08-09

本文件記錄 `5G_NWDAF_Infrastructure` 從全 VM 改為三台 VM 加五個 Host ML containers 後，
正式建立任何新 VM／container 前的唯讀 Host 清點。清點沒有建立、停止或刪除既有 VM、
container、image、volume、route 或 interface。

## 結論

- Host 有 62 GiB RAM，清點時約 27 GiB `MemAvailable`；新 baseline 的 VM allocation 為
  10 GiB，另保留 6 GiB 給 Host／ML，短時間 bring-up 的 RAM gate 可滿足。
- 2 GiB swap 幾乎完全使用。這不直接阻止 bounded smoke，但必須警告，不適合無人長時間
  運行；deployment script 不自動建立、清除或重設 swap。
- Workspace 與 Docker data-root 都位於 `/dev/nvme1n1p1`；938 GiB filesystem 約使用
  703 GiB、可用 188 GiB。三台 VM 各宣告 40 GiB dynamic logical disk，不會立即配置
  120 GiB，但 box、build、image layer 與 guest writes 仍共享這 188 GiB。
- Docker 27.4.1、Compose 2.32.1、overlay2 與 `/var/lib/docker` 可由實際 Host user context
  存取。Daemon 是多人共用資源；本專案不得清理未由自己建立的 container/image/volume。
- VirtualBox 6.1.50 與 Vagrant 2.4.3 已安裝；實際 Host context 的
  `VBoxManage list vms` 可初始化。受限執行 sandbox 看不到 `/dev/vboxdrv`、Docker socket
  或 netlink，不可用該 sandbox 視圖取代 Host preflight。
- 公開 isolated candidate `192.168.56.0/21` 在清點時未出現在 Host route；ML ports
  `9091`、`9092`、`9093`、`9094`、`9292` 無既有 listener。Host `9090` 已被 Prometheus
  使用，因此 consumer callback 必須維持在 Core VM `192.168.57.32:9090`，不能發布到
  Host wildcard。Host `27017` 也已有共用 MongoDB listener；Core VM MongoDB 使用不同的
  `192.168.57.18:27017`，不可改成 Host wildcard forwarding。

## 固定的初始資源與 endpoint

| 執行單位 | RAM | vCPU | Logical disk / endpoint |
| --- | ---: | ---: | --- |
| Core VM | 4096 MiB | 4 | 40 GiB dynamic primary |
| Path A VM | 3072 MiB | 3 | 40 GiB dynamic primary |
| Path B VM | 3072 MiB | 3 | 40 GiB dynamic primary |
| PyAnLF-A | dynamic Host RSS | CPU | `192.168.57.1:9093` |
| PyAnLF-B | dynamic Host RSS | CPU | `192.168.57.1:9094` |
| PyMTLF-A | dynamic Host RSS | `cuda:0` | `192.168.57.1:9092` |
| PyMTLF-B | dynamic Host RSS | `cuda:0` | `192.168.57.1:9091` |
| PyMTLF-C | dynamic Host RSS | CPU | `192.168.57.1:9292` |

Container RSS 不會在 VM 啟動時一次預留。`cuda:0` 只表示預期 device；在 NVIDIA
Container Toolkit 安裝獲得另行授權、且 A/B 單獨與同時 training smoke 通過前，不能宣稱
GPU runtime 已可用，也不能排除 10 GiB RTX 3080 需要 sequential fallback。

## PseudoDriver dataset gate

| Path | Bytes | Rows | SHA-256 |
| --- | ---: | ---: | --- |
| A / group1 | 21,830,425 | 2,720,063 | `d9e2772de8529870e272f44a3bc02863e8831d9c90d51eb0b33961eb28a29030` |
| B / group2 | 44,050,349 | 5,759,921 | `b8482a21f3370de491a67fa1f9908e1b8b3aec7671787bbbdb0d9287680b662e` |

兩份檔案的 direct streaming probe 約為 23 MiB process RSS，但這不包含完整 UPF、page
cache、gtp5g、UERANSIM、NWDAF 或多 subscription。每個 Path 在 replay warm-start 前仍需
至少 512 MiB guest `MemAvailable`；未達時停止，不靠 swap 硬撐。

## 下一個安全 gate

1. 先提交 topology/config/preflight 與本文件，不建立 runtime。
2. 實作兩種 ML image、五個 Compose service 及 `ml-start/status/stop`，先做 CPU-only
   container/config/health smoke，不安裝 NVIDIA toolkit。
3. 由 provider 建立 isolated host-only network 後，驗證 `192.168.57.1` 存在、三台 VM
   雙向 route、五個 port 與 firewall；失敗時才以 `testbed.local.yaml` 調整 physical bind。
4. 另行取得 Host prerequisite 授權後才安裝／設定 NVIDIA Container Toolkit 並做 GPU smoke。
