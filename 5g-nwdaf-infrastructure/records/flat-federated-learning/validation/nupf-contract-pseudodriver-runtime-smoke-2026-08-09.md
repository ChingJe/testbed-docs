# Nupf Contract 與 PseudoDriver Runtime Smoke

日期：2026-08-09

## 結論

三台 VM、五個 Host ML containers、雙 NWDAF subscription 與 generated PseudoDriver
在同一個 bounded run 中完成短時間資料閉環。新版 go-upf contract fix 已排除先前
SMF 建立 Nupf Event Exposure subscription 時的 `502`：A/B 各三個 PDU IP 都進入 replay，
PyAnLF-A/B 接受 callback，consumer 最後收到兩筆 analytics notification，分別來自
NWDAF-A 與 NWDAF-B。

這不是完整 federated-learning business E2E。驗證在 stable Phase 2 起始後停止，沒有等待
3,000 秒 breaking time，也沒有觸發 degradation、Model Monitor、training、FedAvg、model
publication、reprovision 或 generation cutover。

## Source 與靜態驗證

- Infrastructure revision：`e119dc9d5f9c75e1d70392ccd2c7e2922800c530`；
- go-upf branch：`fix/r18-nupf-event-exposure-contract`；
- go-upf revision：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`；
- direct parent：`9a4d95ca02ddcee29cd07989d7cf6153d01061c7`；
- generated dataset set：
  `3cc771b6d283ceee5927e3986dbe1920039e72ce69575389c10556a82a8be4a2`；
- effective config identity：
  `default:abb0f2eabbd5dda025b9b8e2c5abdfe1c82a211818121f5d3e12065eba54f959`。

`234bae0` 只修改 EES API contract 與測試，保留 parent 的 shared Parquet reader、batching、
network-clock synchronization 與 PseudoDriver。獨立 audit clone 中：

- `go test ./internal/ees`：通過；
- `go build ./cmd`：通過；
- full `go test ./...`：未通過；已知失敗包含 root `sim_my_test.go` 的 unused `math`
  import，以及需要 netlink／gtp5g privilege 的 forwarder/app tests，並非本次 contract
  patch 所引入。

Host preflight 為 0 failures、2 warnings：未由環境變數選定 provider，以及 free swap 約
1 MiB。實際 Vagrant commands 明確使用 VirtualBox。啟動前 Host 約有 25.1 GiB
`MemAvailable`，workspace／Docker filesystem 約 163–164 GiB free，16 個 component locks
一致，generated dataset audit 通過。

## Build 與 5GC bring-up

Path A/B 都由 `234bae0` 重建 UPF。兩份實際執行 binary 的 SHA-256 相同：

```text
931ea003d6055d0f1f20d26f19cdf7f454bf12b4139f8ae6bcacfd72e27ac0e7
```

Guest stack 啟動後 23 個 units 全部 active。六個 UE 完成 registration 與 PSI 1 PDU
Session：

| Path | PDU addresses |
| --- | --- |
| A | `10.60.0.1`, `10.60.0.2`, `10.60.0.3` |
| B | `10.61.0.1`, `10.61.0.2`, `10.61.0.3` |

本輪 bring-up 前補上 default UDM `internalGroupIdentifiersRanges`，renderer 從
`testbed.yaml` 的 `mobileNetwork.internalGroupId` 產生相同 range，checker 也驗證 exact
value。這是讓 UDM 正確解析單一 Internal Group 的必要 deployment config，不改 NWDAF
architecture ownership。

## Subscription 與 replay 證據

Consumer 透過 NRF 找到兩個不同 NWDAF，建立兩筆 active subscription：

| Path | NF instance | Subscription resource | Correlation |
| --- | --- | --- | --- |
| A | `11111111-1111-4111-8111-111111111111` | `d8b9b419-6daa-43f8-aef6-ddead01535c4` | `5g-nwdaf-a-a18d6a2740df` |
| B | `22222222-2222-4222-8222-222222222222` | `cbf1ea48-8adf-43fc-99e6-2623874d2423` | `5g-nwdaf-b-2317e748b44e` |

SMF 在本輪時間範圍內收到 12 次
`POST /nsmf-event-exposure/v1/subscriptions`，全部回 `201`，沒有 `502`。下游最終狀態是
每個 UPF 三筆 subscription，對應各自三個 PDU IP，reporting period 均為 30 秒，callback
分別指向 Host PyAnLF `192.168.57.1:9093` 與 `:9094`。

兩邊都完成 dataset scan／batch preparation，並開始 replay pacing：

- shared stream Phase 1：18,006 packets；
- shared stream Phase 2：8,994 packets；
- first URR signal 後立即 anchor timeline；
- 每個 Path 三筆 subscription 都完成 historical warm-start；
- 每筆 subscription 啟動 50 個 future windows 的 pacing；
- 本輪實際觀察到 window 100、101、102，每個 window 30 秒。

PyAnLF-A/B logs 各觀察到至少六次
`POST /callbacks/upf-event-exposure`，全部回 `204`，代表至少兩輪、每輪三個 subscriber 的
callback 已被接受。Consumer state 最後為 `notificationCount: 2`，A/B correlation 各收到
一筆，`lastNotificationAt` 為 `2026-08-09T12:22:04.414792Z`。

## 資源快照

完整 stack 運行時 Host 狀態：

- RAM：64,140 MiB total、35,452 MiB used、24,241 MiB available；
- swap：2,047 MiB 中 2,046 MiB 已使用；
- workspace：938 GiB filesystem、164 GiB available、82% used；
- 本專案五個 ML containers RSS 合計約 1.33 GiB。

| Container | Runtime RSS | Limit |
| --- | ---: | ---: |
| PyAnLF-A | 240.0 MiB | 768 MiB |
| PyAnLF-B | 240.1 MiB | 768 MiB |
| PyMTLF-A | 297.7 MiB | 1.5 GiB |
| PyMTLF-B | 297.5 MiB | 1.5 GiB |
| PyMTLF-C | 283.3 MiB | 1 GiB |

同一時間的 service cgroup current snapshot 中，UPF-A 約 8.1 MiB、UPF-B 約 9.5 MiB。
Host 使用 cgroup v1，沒有取得可採信的 `memory.peak`；page cache、整台 Path VM 與未來
training 也不在這兩個 current 數字內。因此這些資料只能證明短 smoke 沒有 OOM／kill，
不能用來縮小 Path VM RAM 或關閉 replay peak gate。Swap 已滿也是後續長跑前必須持續觀察
的 Host 壓力訊號。

## Lifecycle 修正與 teardown

Consumer state file 由 service identity 建立，Host status/delete helper 原先用 Vagrant SSH
登入使用者讀取，會遇到 permission denied。本輪將 CLI 固定以 `5g-nwdaf` identity 執行，
start/status/stop 共用同一條 helper，沒有放寬 state file 權限。

Guest stop list 也移除已遷移到 Host containers 的 PyAnLF/PyMTLF unit names；VM power、Guest
services、Host ML 與 subscriptions 仍維持四個分離 lifecycle。

結束時先 exact DELETE 兩筆 NWDAF subscription，再依序停止 consumer、五個 Host ML
containers、23 個 Guest services 與三台 VM。最終結果：

- Core、Path A、Path B 都是 `poweroff`；
- 五個本專案 containers 都是 exited，images、volumes 與 state 保留；
- 原有八個共用 containers 全部仍是 running；
- Host `MemAvailable` 回升到 29,737 MiB；swap 仍為 full。

## 尚未證明

- 3,000 秒 stable reference 與 Path A degraded tail 的完整時間行為；
- WAPE degradation 是否按 plan 觸發，Path B 是否維持 stable；
- NWDAF-C Model Monitor 是否協調 A/B training；
- PyMTLF-A/B concurrent GPU training 的 VRAM peak 與 timeout 行為；
- ADRF local data、sample-count-weighted FedAvg、publication 與 transaction identity；
- A/B reprovision、monitor generation cutover 與 post-training recovery；
- PseudoDriver／Path VM 的可靠 peak RAM。
