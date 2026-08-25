# Path VM Skeleton Smoke — 2026-08-09

本報告記錄 Path A、Path B 與三台 VM 同時運行的 bounded skeleton/network smoke。所有 VM
使用 `--no-provision`；沒有 apt install、kernel module build、UERANSIM build 或 5G service。

## VM baseline

| VM | Guest | RAM / vCPU | Interfaces | Source | VMDK on disk |
| --- | --- | --- | --- | ---: | ---: |
| Core | Ubuntu 22.04.5 / kernel 5.15.0-171 | 4096 MiB / 4 | NAT + management/SBI/N2/N4 | 106 MiB | 1619 MiB |
| Path A | Ubuntu 22.04.5 / kernel 5.15.0-171 | 3072 MiB / 3 | NAT + management/SBI/N2/N3-A/N4/N6-A | 106 MiB | 1619 MiB |
| Path B | Ubuntu 22.04.5 / kernel 5.15.0-171 | 3072 MiB / 3 | NAT + management/SBI/N2/N3-B/N4/N6-B | 106 MiB | 1619 MiB |

三台都沒有 guest `ML/`，也沒有 `/vagrant` vboxsf。SSH forwarded ports 由 Vagrant 分配為
Core 2222、Path A 2200、Path B 2201。

## Address verification

- Core：`192.168.56.10`、`192.168.57.2`、`192.168.58.2`、`192.168.61.2`
- Path A：`192.168.56.11`、`192.168.57.3`、`192.168.58.3`、`192.168.59.2`、
  `192.168.61.3`、`192.168.62.2`
- Path B：`192.168.56.12`、`192.168.57.4`、`192.168.58.4`、`192.168.60.2`、
  `192.168.61.4`、`192.168.63.2`

Host 建立八個獨立 host-only interfaces，`.1` 位址涵蓋 56–63 八個 `/24`。以下均通過：

- Host → 所有 VM interface ping
- 每個 Path → 所屬 Host interface ping
- Core ↔ Path A 的 management、SBI、N2、N4
- Core ↔ Path B 的 management、SBI、N2、N4
- Path A ↔ Path B 的 management、SBI、N2、N4

N3-A/N6-A 與 N3-B/N6-B 是刻意分離的 segment，本輪沒有新增跨 Path route。

## Host ML TCP path

Host 暫時在 `192.168.57.1:9091` 啟動 HTTP listener。Core、Path A、Path B 各執行一次 GET，
全部取得 200；Host log 顯示來源依序是 `192.168.57.2`、`.3`、`.4`。Listener 測試後停止，
port 已確認無殘留。Production ML services 未啟動，因此這是 reachability evidence，不是
application health evidence。

## Resource and cleanup

三台同時 running 時 Host 約有 24 GiB `MemAvailable`、175 GiB workspace free，swap 約剩
1 MiB。八個既有 shared Docker containers 保持 running。完成後三台 VM 都 graceful halt；
VM 與 dynamic disks 保留，沒有 snapshot。

下一個 gate 是分階段 provisioning：先 Core package/toolchain/MongoDB/NF build，再對 Path
做 gtp5g、UPF、NWDAF 與 UERANSIM build。Provisioning 前仍應重新執行 Host preflight。
