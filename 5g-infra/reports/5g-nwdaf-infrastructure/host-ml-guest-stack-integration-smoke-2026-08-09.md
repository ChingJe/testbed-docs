# Host ML 與 Guest Stack 整合 Smoke（2026-08-09）

## 結論

三台 VM 的完整 guest stack 與五個 production Host ML containers 已在同一個 bounded run
中成功啟動。PyMTLF-A/B 使用 NVIDIA runtime CDI mode 並看到 RTX 3080；五個 backend 都
healthy，且 VM 到 Host、Host backend 到 VM NWDAF 的週期性同步都成功。

這完成 production ML activation 與 hybrid transport gate，不是 full-core business E2E。
本輪沒有啟動 consumer subscription、N6 traffic、PseudoDriver replay、analytics callback、
model training 或 federated learning。

## 啟動條件與 identity

- Infrastructure config：`default`
- Config hash：`a73ef32bb621b3a20efa836f12183a95bdf0e7bd34cfb8565f8884626f5a99c0`
- PyAnLF revision：`9e64417fe053`
- PyMTLF revision：`e9c5b08725dc`
- PyAnLF image：`945911e04f05`
- PyMTLF image：`6c9faa5c5720`
- Host preflight：0 failure；swap free 約 1 MiB 的既定 warning
- 啟動前 Host `MemAvailable`：約 27 GiB
- Workspace free space：約 166 GiB

啟動順序是 `vm-up`、`services-start`、`ml-start`。Guest lifecycle 先 idempotently apply
六個 subscriber 與單一 Internal Group，再啟動 5GC、三個 NWDAF、兩個 UPF、兩個 gNB 與
六個 UE。ML lifecycle 沒有建立 subscription，也沒有隱含啟動 VM process。

## Production ML 結果

五個 services 全部到達 application readiness：

| Service | Placement | State | CUDA | 空載 RSS |
| --- | --- | --- | --- | ---: |
| PyAnLF-A | CPU | healthy | 不適用 | 約 244 MiB |
| PyAnLF-B | CPU | healthy | 不適用 | 約 239 MiB |
| PyMTLF-A | `cuda:0` | healthy | available | 約 298 MiB |
| PyMTLF-B | `cuda:0` | healthy | available | 約 298 MiB |
| PyMTLF-C | CPU | healthy | 不使用 | 約 298 MiB |

合計約 1.38 GiB；三台 VM、guest stack 與 containers 同時運行時 Host 約有 22 GiB
`MemAvailable`。這只是 idle/sync 狀態，不能作為 training peak 或 PseudoDriver peak。

當時 `nvidia-smi` 顯示 10240 MiB VRAM 中約 432 MiB 已使用，主要可見 process 是另一位
使用者的 Python process（約 406 MiB）。本 testbed 的 PyMTLF containers 已通過 CUDA probe，
但未開始 training，因此不能用這次 VRAM 數字推論雙 client training 一定放得下。

## 跨 VM／Host 驗證

由各 VM 直接呼叫所屬 Host endpoint，全部回 HTTP 200：

| Caller | Backend | Endpoint | Result |
| --- | --- | --- | --- |
| Core `.57.2` | PyMTLF-C | `192.168.57.1:9292/health/ready` | ready / `fl_server` |
| Path A `.57.3` | PyAnLF-A | `192.168.57.1:9093/health/ready` | ready |
| Path A `.57.3` | PyMTLF-A | `192.168.57.1:9092/health/ready` | ready / `fl_client` |
| Path B `.57.4` | PyAnLF-B | `192.168.57.1:9094/health/ready` | ready |
| Path B `.57.4` | PyMTLF-B | `192.168.57.1:9091/health/ready` | ready / `fl_client` |

三個 NWDAF 都向 NRF 註冊成功。Go NWDAF 週期性對對應 ML backend 執行
`POST /internal/v1/sync`，五個 backend logs 均為 200。PyAnLF-A/B 也將 SMF resource
associations 與 training-data descriptors 寫回各自 NWDAF，NWDAF logs 對兩個
`PUT /internal/v1/sync/anlf/...` endpoint 均回 204。這證明雙向 transport 與 sync contract
可用，但 descriptor 數量仍為零，不能解讀為 ADRF training data flow 已完成。

## 發現與限制

PyAnLF-A/B 均回報 callback buffer 的高理論 memory bound：8192 entries × 4 MiB request
ceiling，約 32 GiB。這不會在 startup 預先配置，且本輪每個 instance 只使用約 240 MiB；
但大量 callback backlog 仍可能先碰到 768 MiB container limit、drop-oldest 或 OOM。正式
traffic gate 必須量測 callback payload、queue depth、drop counter 與 peak RSS。

Subscriber fixture 的內容 identity 應是
`d30803f9c5904ae86bb222484170089cc4cf60ee3fe3f29e43c6487918113167`。本輪 wrapper log
暴露出 Host script 將 clone 的 absolute path 納入二次 hash，導致相同內容換 root 後 identity
不同；修正為 repository-relative filenames，不改 fixture 或 MongoDB 資料。

## Teardown

本輪依序執行 `ml-stop`、`services-stop`、`vm-halt`：

- 五個 production containers 都是 stopped；images、volumes 與 config identity 保留；
- guest services 依 reverse order 停止；
- Core、Path A、Path B 都是 `poweroff`；
- 執行中的 Docker containers 回到測試前的八個共用 containers；
- Host `MemAvailable` 回到約 27 GiB。

下一個 bounded gate 應先啟動 consumer 的兩個 NWDAF subscriptions 與 callback observation，
再驗證 Event Exposure／PseudoDriver data path；只有資料與 coordination path 可用後，才執行
有 timeout 的 A/B concurrent training 與 VRAM/RAM peak 量測。
