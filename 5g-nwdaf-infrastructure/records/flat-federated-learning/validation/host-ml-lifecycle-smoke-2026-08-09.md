# Host ML Lifecycle Smoke — 2026-08-09

## 結論

`5G_NWDAF_Infrastructure` 的 Host ML lifecycle 已在獨立 CPU-only project 完成 bounded
start → status → logs → stop 驗證。五個 services 全部到達 application readiness；stop
保留 stopped containers、named volumes 與 images，最終 cleanup 只移除 disposable smoke
project。沒有建立 VM、修改 Host network、安裝 NVIDIA Container Toolkit 或操作其他人的
Docker 資源。

## Lifecycle ownership

- Production project：`5g-nwdaf-infrastructure`
- Smoke project：`5g-nwdaf-infrastructure-lifecycle-smoke`
- Production commands：`make ml-start`、`make ml-status`、`make ml-stop`
- Bounded validation：`make ml-lifecycle-smoke`

Status、stop 與 ML log discovery 都使用 exact Compose project/service labels。`ml-stop` 只向
該 project 的 running container IDs 發出 bounded stop，不執行 `down`、volume/image delete
或 global prune。Production start 會先做 config/Compose checks、bind-address gate、Host RAM
reserve／Docker free-space gate、image build 與實際 CUDA probe；CUDA 不可見時不得啟動設定為
`cuda:0` 的 A/B services。Swap 依 topology 的 warn/require policy 處理。

## CPU smoke result

CPU override 將 PyMTLF-A/B effective training device 改成 `cpu` 並移除 Compose GPU request。
所有 container 都顯示 CUDA unavailable，符合此次無 GPU device request 的預期。
Lifecycle resource gate 當下觀測 `MemAvailable` 28,212 MiB、Docker data-root free 181 GiB，
分別通過 6,144 MiB reserve 與 120 GiB threshold；free swap 0 MiB 依既定 policy 只警告。

| Service | State/health | Empty-service RSS | Image | Source revision |
| --- | --- | ---: | --- | --- |
| PyAnLF-A | running / healthy | 約 229 MiB | `114933a70a70` | `9e64417fe053` |
| PyAnLF-B | running / healthy | 約 231 MiB | `114933a70a70` | `9e64417fe053` |
| PyMTLF-A | running / healthy | 約 283 MiB | `d5b00c37f757` | `e9c5b08725dc` |
| PyMTLF-B | running / healthy | 約 283 MiB | `d5b00c37f757` | `e9c5b08725dc` |
| PyMTLF-C | running / healthy | 約 283 MiB | `d5b00c37f757` | `e9c5b08725dc` |

五者使用同一 config-set `ml-lifecycle-smoke` 與 config hash prefix `d6a9d402053d`。總空載 RSS
約 1.28 GiB，和先前 image/health smoke 一致；這仍不是 training、dataset、FedAvg 或 GPU
peak capacity evidence。

`ml-status` 已證明可同時呈現 state、health、device、CUDA visibility、memory、image ID、
source revision 與 config identity。`logs.sh --source ml --service pyanlf-a --no-follow` 可只讀取
該 service 並以 `[ml:pyanlf-a]` prefix 呈現。Stop 後 smoke 驗證五個 containers 均不再
running，且五個 stopped containers 與五個 named volumes 仍存在；之後才由 smoke cleanup
移除它們與 generated config，images 保留。

## 發現與限制

第一次 lifecycle run 的五個 services 已 healthy，但 Host status reporter 使用系統 Python
不支援的 `str.removeprefix()` 而失敗；error trap 正確輸出 ML logs 並清除 disposable project。
Reporter 已改成相容寫法，後續完整 lifecycle 通過。

PyAnLF 另外警告 default callback buffer 的理論上限為 8192 × 4 MiB。它不會在啟動時配置
32 GiB，實測 empty RSS 約 230 MiB，且 container hard limit 為 768 MiB；但 callback backlog
仍可能導致 container-local OOM 或 drop-oldest。正式 traffic smoke 必須觀察 payload size、
queue depth、drop counters 與 peak RSS，再決定是否調低 capacity／request ceiling 或提高
service RAM，不能只以 empty-service 數值決定。

尚未驗證：

- production project 的 NVIDIA runtime、GPU visibility 與 `cuda:0` activation；
- PyMTLF-A/B single/dual training 的 Host RAM 與 RTX 3080 VRAM peak；
- `192.168.57.1` bind、VM-to-Host route、firewall 與五個 published endpoints；
- 三台 VM、guest services、subscription consumer 與 full-core business E2E。
