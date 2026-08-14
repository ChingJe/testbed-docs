# Experiment Authoring Full-core Runtime Acceptance — 2026-08-14

## Result

Phase 8.7 的 explicit scenario path、non-gating diagnostics、dataset summary 與新版設定文件
完成後，以三台既有 Ubuntu 22.04 VM 和 Host GPU 執行一次乾淨的
`full-core-cat-transition` 驗收。本輪完成六個 UE Registration／PDU Session、雙 Path
analytics callbacks、A/B GPU local training、C FedAvg、ADRF publication、雙 scope adoption、
cutover 與 post-cutover evaluated accuracy；current-run summary 為 `complete`，沒有 FL failure。

標準 teardown 也完成：兩筆 Consumer exact resources 為 `deleted`，五個 containers exited、
23 個 Guest units inactive、PyMTLF-C Model Monitors `active=0`，最後三台 VM graceful poweroff。
沒有修改任何 NF／ML／RAN submodule。

## Identity and preparation

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
- GPU：RTX 3080 10 GiB，driver `535.183.01`

使用新的 public interface 明確建立 config：

```sh
make config-create \
  NAME=runtime-validation-full-core-20260814-01 \
  FROM=experiments/examples/full-core-cat-transition/scenario.yaml \
  DEVICE=gpu WEBCONSOLE=false
```

`dataset-generate` 重用 matching content-addressed set；新版 `dataset-show` 正確顯示
1 秒 raw windows、30 秒 observations、90 秒 accuracy reports、900 秒 warm-start、
1170／1290 秒 earliest／bounded trigger 與 3090 秒 bounded closure。

唯讀 `experiment-validate` 為 `failures=0, warnings=1`。唯一 warning 是 2 GiB swap
完全使用；啟動前 Host 約有 27 GiB `MemAvailable`、223 GiB free storage，所有五個 ML
ports、SBI Host address、VirtualBox allowlist、16 個 component locks、GPU CDI 與 NVIDIA
runtime 均通過。Diagnostics 沒有被 `experiment-start` 再次執行或作為 start gate。

前一輪 retained state 先用 guarded reset 清除：ADRF data records 261 筆、model record 1 筆、
一個 model artifact、一筆 ADRF NRF URI-list state，以及五個 ML volumes 內容；containers、
images、network、volume objects、VM、config 與 dataset 都保留。

## Startup and readiness

`make vm-up` 只開啟既有 VM，沒有 reprovision。三台 persistent network aliases verified，
Path A/B gtp5g vermagic 都符合 `5.15.0-171-generic` 且 module loaded。

`make experiment-start` 沒有 rollback：

- 23 個 Guest units active；
- UE1–UE6 的 current-invocation Registration 與 PDU Session 全部 `successful`；
- 五個 ML containers healthy；
- PyMTLF-A/B 為 `cuda:0`、NVIDIA runtime、CDI `nvidia.com/gpu=all`、CUDA `true`；
- PyAnLF-A/B 與 PyMTLF-C 使用 CPU；
- Consumer 經 NRF 建立 A/B 兩筆不同 provider、TAC、API root 與 exact Location 的 resources；
- A/B fresh callback timestamps 持續每 30 秒前進，兩路沒有中斷或明顯偏斜。

## FL closure evidence

主要 current-run 時間線使用 UTC（Asia/Shanghai 為 UTC+8）：

| Event | UTC | Evidence |
| --- | --- | --- |
| ML coordinator start | 07:09:21 | matching config name/hash container identity |
| Consumer resources active | 07:09:45 | distinct A/B providers, TACs and Locations |
| Two Model Monitors active | 07:09:47 | two scopes |
| Degradation trigger | 07:29:47 | `evaluated=True triggered=true` |
| FL process | 07:29:47 | `dffabf31-200a-46a6-b3b1-7a8074ef7ca5`, two scopes |
| Preparation | 07:29:47 | both `111...` and `222...` participants |
| A/B rounds | 07:29:50 | rounds 0 and 1 plus final validation from both clients |
| Validation | 07:29:51 | base WAPE `1.9099532573`, candidate `0.3053598029`, gate would accept |
| Publication | 07:29:51 | model `1786692591012`, two required scopes |
| Adoption and cutover | 07:29:51 | two scopes, `complete=True` |
| First accepted post-cutover evidence | 07:31:21 | `evaluated=true triggered=false` |
| Latest retained post-cutover evidence | 07:35:51 | `evaluated=true triggered=false` |

PyMTLF-A/B remained healthy during training and each used about 720 MiB container memory at the
observed peak. Current-run FL failure stayed `not-seen`.

## Teardown evidence

`make experiment-stop` deleted both exact Consumer resources and stopped the callback, retained the
documented fixed 40-second backend grace, then stopped WebConsole domain, five ML containers and all
Guest units. Final checks before halt showed：

- Consumer service inactive；A/B resources both `deleted`；
- all 23 Guest units inactive；
- five ML containers exited with matching config identity；
- PyMTLF-C monitors `created=4, active=0`；
- retained FL result `complete` and failure `not-seen`。

`make vm-halt` then left Core、Path A、Path B all `poweroff`. Other users' Docker projects were not
stopped or modified.

## Findings exposed by this run

1. **Consumer callback counters are retained across runs.** Before the first fresh callback, the new
   subscription state still showed the previous run's A/B counts `44/44` and timestamps. New callbacks
   updated the timestamps and continued counts to `97/97` by stop. Fresh delivery remained provable by
   timestamps after `createdAt`, but the count is cumulative retained state rather than a current-run
   count. A future Infrastructure-only change should either reset per-Path counters when a new pair of
   resources is created, or display cumulative and current-subscription counters separately.
2. **Two config digest algorithms are displayed without a clear distinction.** `config-check.py`
   reported tree SHA-256 `646fd3c0…`, while activation, Compose labels and runtime status used
   `f0a33667…`. Both are deterministic over the same directory but use different framing algorithms.
   Runtime identity was internally consistent; operator output should name them distinctly or converge
   on one canonical digest.
3. The low-swap preflight warning still used the removed phrase `hard RAM gate`. This was a stale
   Infrastructure message, not runtime behavior; it was corrected after the run to say that operators
   should monitor `MemAvailable` during long runs.

The first two findings do not invalidate this closure because current timestamps, matching container
identity and full FL evidence establish the tested run independently. They should not be hidden in
documentation or attributed to NF／ML component behavior.
