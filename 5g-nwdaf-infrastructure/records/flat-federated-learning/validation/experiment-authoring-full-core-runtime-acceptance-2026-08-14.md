# 實驗設定建立與 Full-core Runtime 驗收 — 2026-08-14

## 驗收結果

Phase 8.7 的明確 scenario 路徑、非阻擋式診斷、dataset 摘要與新版設定文件
完成後，以三台既有 Ubuntu 22.04 VM 和主機 GPU 執行一次乾淨的
`full-core-cat-transition` 驗收。本輪完成六個 UE Registration／PDU Session、雙 Path
analytics callback、A/B GPU 本地訓練、C FedAvg、ADRF 發布、雙 scope adoption、
cutover 與 cutover 後 accuracy 評估；本輪摘要為 `complete`，沒有 FL failure。

標準停止流程也已完成：兩筆 Consumer 精確資源為 `deleted`，五個 container 已退出、
23 個 Guest unit 為 inactive、PyMTLF-C Model Monitor `active=0`，最後三台 VM 正常關機。
沒有修改任何 NF／ML／RAN submodule。

## 身分與準備

- Infrastructure runtime revision：`52fffd6124e2cc76b07d05361e28bcaa9b4416fa`
- Config：`config/local/runtime-validation-full-core-20260814-01`
- Runtime config hash：`f0a33667b07d28e2cb0359fd58fda807fbc9268afdb055186d079bc4ca3e5bbf`
- Scenario：`experiments/examples/full-core-cat-transition/scenario.yaml`
- Dataset set：`23697bf00ae0560c9f07f8ae451ebb91797943092317aea8cafdb37435c2fd59`
- NWDAF：`c53f05804c6533ec4c5fa7e230e90048fb219162`
- PyAnLF：`08798f15c3693027e00bc60dd53f74ebaa26c3a1`
- PyMTLF：`7e8ab7f23bf5d6398eb1cd5f053dd8bda9439a87`
- UPF：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- UERANSIM：`2a3ef81f189ca95d5c1996a28ed7af9734f5cfb4`
- gtp5g：`8d723c29fc0de3eeeff3e9a91132838579e8ee1b`
- GPU：RTX 3080 10 GiB，驅動程式 `535.183.01`

使用新的對外操作介面明確建立 config：

```sh
make config-create \
  NAME=runtime-validation-full-core-20260814-01 \
  FROM=experiments/examples/full-core-cat-transition/scenario.yaml \
  DEVICE=gpu WEBCONSOLE=false
```

`dataset-generate` 重用內容相符的 content-addressed set；新版 `dataset-show` 正確顯示
1 秒 raw window、30 秒 observation、90 秒 accuracy report、900 秒 warm-start、
1170／1290 秒最早／有界 trigger，以及 3090 秒有界 closure。

唯讀 `experiment-validate` 為 `failures=0, warnings=1`。唯一警告是 2 GiB swap
已完全使用；啟動前主機約有 27 GiB `MemAvailable`、223 GiB 可用儲存空間，五個 ML
port、SBI 主機位址、VirtualBox allowlist、16 個 component lock、GPU CDI 與 NVIDIA
runtime 均通過。`experiment-start` 沒有再次執行診斷，也沒有把診斷當成啟動門檻。

前一輪保留狀態先透過有確認機制的 reset 清除：ADRF data record 261 筆、model record 1 筆、
一個 model artifact、一筆 ADRF NRF URI-list state，以及五個 ML volume 的內容；container、
image、network、volume object、VM、config 與 dataset 都保留。

## 啟動與就緒狀態

`make vm-up` 只開啟既有 VM，沒有重新 provision。三台 VM 的持久化 network alias 均驗證成功，
Path A/B 的 gtp5g vermagic 都符合 `5.15.0-171-generic`，module 也已載入。

`make experiment-start` 沒有觸發 rollback：

- 23 個 Guest unit 均為 active；
- UE1–UE6 本次 systemd invocation 的 Registration 與 PDU Session 全部為 `successful`；
- 五個 ML container 均為 healthy；
- PyMTLF-A/B 為 `cuda:0`、NVIDIA runtime、CDI `nvidia.com/gpu=all`、CUDA `true`；
- PyAnLF-A/B 與 PyMTLF-C 使用 CPU；
- Consumer 經 NRF 建立 A/B 兩筆 provider、TAC、API root 與精確 Location 都不同的資源；
- A/B 本輪 callback timestamp 持續每 30 秒前進，兩路沒有中斷或明顯偏斜。

## FL 閉環證據

主要本輪時間線使用 UTC（Asia/Shanghai 為 UTC+8）：

| 事件 | UTC | 證據 |
| --- | --- | --- |
| ML coordinator 啟動 | 07:09:21 | container 的 config name/hash 身分相符 |
| Consumer 資源啟用 | 07:09:45 | A/B provider、TAC 與 Location 均不同 |
| 兩個 Model Monitor 啟用 | 07:09:47 | 兩個 scope |
| Degradation trigger | 07:29:47 | `evaluated=True triggered=true` |
| FL process | 07:29:47 | `dffabf31-200a-46a6-b3b1-7a8074ef7ca5`，兩個 scope |
| Preparation | 07:29:47 | `111...` 與 `222...` 兩個 participant |
| A/B round | 07:29:50 | 兩個 client 都完成 round 0、1 與 final validation |
| Validation | 07:29:51 | base WAPE `1.9099532573`、candidate `0.3053598029`，gate 判定可接受 |
| Publication | 07:29:51 | model `1786692591012`，需要兩個 scope |
| Adoption 與 cutover | 07:29:51 | 兩個 scope，`complete=True` |
| 第一筆通過的 cutover 後證據 | 07:31:21 | `evaluated=true triggered=false` |
| 最後保留的 cutover 後證據 | 07:35:51 | `evaluated=true triggered=false` |

PyMTLF-A/B 在訓練期間維持 healthy；觀察到的 container memory 峰值各約 720 MiB。
本輪 FL failure 始終為 `not-seen`。

## 停止與清理證據

`make experiment-stop` 刪除兩筆精確 Consumer 資源並停止 callback，依文件保留固定 40 秒的
backend 收尾時間，接著停止 WebConsole domain、五個 ML container 與所有 Guest unit。
VM 關機前的最終檢查顯示：

- Consumer service 為 inactive；A/B 資源均為 `deleted`；
- 23 個 Guest unit 全部為 inactive；
- 五個 ML container 都已退出，且 config identity 相符；
- PyMTLF-C monitor 為 `created=4, active=0`；
- 保留的 FL result 為 `complete`，failure 為 `not-seen`。

執行 `make vm-halt` 後，Core、Path A、Path B 全部為 `poweroff`。其他使用者的 Docker project
沒有被停止或修改。

## 本輪發現的問題

1. **Consumer callback 計數會跨實驗保留。** 第一筆本輪 callback 抵達前，新 subscription state
   仍顯示上一輪 A/B 的 `44/44` 與舊 timestamp。新 callback 抵達後雖會更新 timestamp，但計數
   會一路累加，停止時已成為 `97/97`。`createdAt` 之後的新 timestamp 仍足以證明本輪有收到資料，
   但計數代表保留的累積值，而非本輪值。**後續 Infrastructure 修正：**新的 subscription pair
   會在送出 POST 前取得全新的 A/B 與總計數器。Correlation ID 也會先保留，因此即使 callback
   在第二筆資源尚未建立完成時抵達，仍能歸屬到正確 Path。
2. **兩種 config digest 算法同時顯示，卻沒有清楚區分。** `config-check.py` 回報的 tree SHA-256
   是 `646fd3c0…`，activation、Compose label 與 runtime status 則使用 `f0a33667…`。兩者都會對
   相同目錄產生確定結果，但 framing 算法不同；runtime 內部身分仍維持一致。
   **後續 Infrastructure 修正：**validation、activation、Compose label 與 status 現在都共用
   以檔名與內容計算的 canonical tree SHA-256。
3. Low-swap preflight 警告仍使用已移除的 `hard RAM gate` 說法。這是過期的 Infrastructure
   訊息，不是 runtime 行為；本輪後已改成提醒操作者在長時間執行期間監看 `MemAvailable`。

前兩項問題不會推翻本次閉環驗收，因為本輪 timestamp、相符的 container identity 與完整 FL
證據能獨立證明結果。兩者都已在 Infrastructure 工具層修正，沒有改變 NF／ML component 行為。

## 觀測修正的針對性驗收

後續短版 runtime 驗收刻意保留上一輪 Consumer state，並重用
`config/local/runtime-validation-full-core-20260814-01`。啟動前沒有 reset ADRF、MongoDB、ML
volume 或已保存的 Consumer 計數，因此可以直接驗證新的 subscription pair 是否仍會繼承
先前的 `97/97`。

第一次啟動發現一項尚未整合完整的變更。Host validation 與 staging 使用 canonical tree
SHA-256 `646fd3c08383daffe2866c63ebeb0c2e4071afba07446f61c0ea62d7ca11497f`，但 Guest activation
腳本仍計算出舊 digest `f0a33667b07d28e2cb0359fd58fda807fbc9268afdb055186d079bc4ca3e5bbf`。
Guest integrity check 正確拒絕 staged directory，aggregate startup 隨即 rollback。修正方式是
把 tree hash 計算集中到一個隨 Guest runtime tools 安裝的共用 Python helper；Host runtime
identity、config validation 與 Guest activation 現在都呼叫同一份實作。

第二次啟動通過所有針對性驗收點：

- Core、Path A 與 Path B 都啟用完整 canonical hash `646fd3c0…11497f`；
- 五個 ML container label 都帶有相同完整 hash；
- 新建的 A/B subscription pair 顯示總計 `0`、Path 計數 `0/0`，沒有延續 `97/97`；
- 第一輪 periodic delivery 後，總 callback 數為 `2`，Path 計數分別為 `1/1`；
- 兩個 Path 使用新的 correlation ID、不同 provider 與不同的精確 resource Location；
- 23 個 Guest unit 均為 active，六個 UE 全部完成 Registration 與 PDU Session establishment，
  五個 ML container 在檢查期間也都維持 healthy。

這次 bounded run 刻意在第一輪雙 Path callback 後停止，沒有等待下一次 FL degradation 或
model cutover。`experiment-stop` 刪除兩筆精確資源、遵守 40 秒 cleanup window，最後留下兩個
已建立且 active 為零的 monitor，並停止 ML 與 Guest domain。三台 VM 隨後都正常關機。
停止後的 Infrastructure `make test` 完整通過，其中包含共用 hash identity regression 與
Consumer generation-counter 測試。
