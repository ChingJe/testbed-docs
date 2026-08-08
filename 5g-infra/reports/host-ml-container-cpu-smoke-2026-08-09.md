# Host ML Container CPU Smoke — 2026-08-09

## 結論

`5G_NWDAF_Infrastructure` 的兩種 ML images 與五個 Compose services 已完成 bounded
CPU-only bring-up。PyAnLF-A/B、PyMTLF-A/B/C 全部到達 application readiness；本次未安裝
或使用 NVIDIA Container Toolkit，也未建立 VM、route 或 Host interface。

## 驗證範圍

- Python base：3.12.11 slim Bookworm digest pin
- PyTorch：`2.5.1+cu121`
- PyAnLF source：`9e64417fe053e03c2a616abea6f284df8acd1b38`
- PyMTLF source：`e9c5b08725dc06835485b29ff6c264340f9805f9`
- Compose services：`pyanlf-a`、`pyanlf-b`、`pymtlf-a`、`pymtlf-b`、`pymtlf-c`
- Smoke bind：Host loopback only
- Smoke device：PyMTLF-A/B config=`cpu`，Compose GPU request removed

Production definition 仍保留 PyMTLF-A/B 的 `cuda:0` config 與 NVIDIA GPU request；CPU
override 只供無 toolkit 時驗證 image、config、process、health 和基本資源，不改 production
semantics。

## 結果

| Service | Readiness | Empty-service RSS | Limit | CPU smoke device |
| --- | --- | ---: | ---: | --- |
| PyAnLF-A | healthy | 231.2 MiB | 768 MiB | CPU / CUDA unavailable |
| PyAnLF-B | healthy | 229.6 MiB | 768 MiB | CPU / CUDA unavailable |
| PyMTLF-A | healthy | 283.3 MiB | 1536 MiB | `cpu` / CUDA unavailable |
| PyMTLF-B | healthy | 283.2 MiB | 1536 MiB | `cpu` / CUDA unavailable |
| PyMTLF-C | healthy | 283.2 MiB | 1024 MiB | CPU / CUDA unavailable |

五個 containers 的觀測總 RSS 約 1.28 GiB。這只代表 empty-service startup，不包含 local
dataset materialization、training tensor、artifact download/extract、FedAvg、GPU VRAM 或
full-stack traffic；正式 RAM/VRAM budget 必須以 single-client 與 dual-client training peak
重新量測。

## Image 與磁碟

| Image | ID | Virtual size | Source revision |
| --- | --- | ---: | --- |
| `5g-nwdaf-infrastructure/pyanlf:local` | `2d874bf1baf8` | 5.42 GB | `9e64417...` |
| `5g-nwdaf-infrastructure/pymtlf:local` | `015964ada36c` | 5.42 GB | `e9c5b08...` |

兩個 image 各有十層，前六層相同，包含共同 Python、uv 與 CUDA-enabled PyTorch runtime。
Docker 當下的 verbose disk report 對兩者都顯示 5.421 GB shared、0 B unique；這個數值會隨
其他 image/build cache references 改變，不能把兩個 virtual size 直接相加成實際新增占用。
測試後 Docker summary 為 images 27.51 GB、build cache 3.158 GB；這是多人共用 daemon 的
整體狀態，不全屬本專案，也未執行 global prune。

## 發現與修正

第一次 bring-up 找到 image source import path 缺失，image target 已明確設定
`PYTHONPATH=/opt/app/src`。第二次 bring-up 找到 PyMTLF default `data/publications` 會嘗試寫入
read-only root filesystem；設定已將 publication journal 和既有 artifact、model state、FL
workspace 一起固定到 service-specific named volume root，checker 也會拒絕相對或越界路徑。

Smoke build 改為每種 target 各 build 一次，其餘三個 service 重用相同 image tag，避免
Compose 對五個 service 排程重複 build。

## Cleanup 與未驗證項目

Smoke 結束後，project `5g-nwdaf-infrastructure-smoke` 的 containers、network、volumes 與
generated config 均已移除；兩個 images 刻意保留。沒有停止、刪除或 prune 其他使用者的
Docker 資源。

尚未驗證：

- NVIDIA Container Toolkit 與 container GPU visibility；
- PyMTLF-A/B single/dual-client training、peak Host RAM 與 10 GiB RTX 3080 VRAM；
- `192.168.57.1` Host interface、VM-to-Host route、firewall 與 published endpoint；
- long-lived `ml-start`／`ml-status`／`ml-stop`、observe/log integration；
- 三台 VM 與 full-core business E2E。
