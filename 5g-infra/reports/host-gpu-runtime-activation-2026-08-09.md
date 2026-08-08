# Host GPU Runtime Activation — 2026-08-09

本報告記錄共用 Docker Host 的 CDI 啟用結果。範圍只包含 NVIDIA Host prerequisite、daemon
reload 安全性與 disposable `nvidia-smi` probe；沒有啟動 testbed ML project、建立 VM、執行
training 或停止其他使用者的 container。

## 結果

- GPU：NVIDIA GeForce RTX 3080，10240 MiB VRAM
- Driver：535.183.01
- Docker：27.4.1，`runc` 仍是 default runtime
- NVIDIA Container Toolkit Base：1.19.1
- CDI devices：index 0、GPU UUID、`nvidia.com/gpu=all`
- Named runtime：`nvidia-container-runtime`
- Probe：NVIDIA runtime CDI mode 的 Ubuntu container 成功執行 `nvidia-smi`
- PyMTLF image：PyTorch `2.5.1+cu121`、CUDA runtime 12.1、CUDA available

Docker reload 前後的 daemon PID 都是 `1759357`，`NRestarts=0`，啟動時間未變。原有八個
running containers 的 ID 和狀態也未改變。Disposable probe 使用 `--rm`，完成後沒有殘留
container。

## 為何沒有使用 Docker native CDI

Docker daemon 接受並驗證 `features.cdi=true`，但 27.4.1 在 reload 後沒有初始化 native CDI
manager。兩次 `--device nvidia.com/gpu=all` probe 都在 container start 前失敗，錯誤是無法
選取 `cdi` driver；測試 container 已清除。若為此 restart daemon，因 Host
`live-restore=false`，會中斷八個共用 containers。

因此改為註冊 named `nvidia` runtime，並透過 reload 套用；沒有改變 default runtime。驗證
成功的 device selection 是：

```text
runtime: nvidia
NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=all
NVIDIA_DRIVER_CAPABILITIES=compute,utility
```

這仍使用 `/var/run/cdi/nvidia.yaml` 的 CDI-qualified device name，只是由 NVIDIA runtime
解析，而不是依賴 Docker 27 的 native CDI manager。

Integration repo 的 PyMTLF image probe 亦成功辨識 RTX 3080，且起始 allocated bytes 為 0。
更新後的 CPU lifecycle regression 讓五個 services 全部 healthy；PyMTLF-A/B 使用 `runc`、
`NVIDIA_VISIBLE_DEVICES=void` 且 CUDA unavailable。Smoke 的 stop、retention 與最終
container／volume cleanup 均通過。

## 尚未完成

- PyMTLF-A、PyMTLF-B 各自 training 的 peak VRAM
- A/B 同時 training 是否可容納於 10 GiB，以及 OOM 時採 sequential 或 fail-fast
- 外部程序目前占用 GPU 時的 baseline、共存與效能影響
- Host-to-VM endpoint 與完整 full-core scenario

在上述量測完成前，不應把 `nvidia-smi` 成功解讀為雙 client training 已可長時間執行。
