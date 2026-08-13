# Current Runtime Validation Baseline（2026-08-14）

## 目的與解讀邊界

這份報告整合 `5G_NWDAF_Infrastructure` 截至 2026-08-14 的主要 runtime
驗證結果，取代 runtime repository 內無日期的 `docs/validation.md`。

它只證明特定 parent gitlink、component revision、config、dataset 與 artifact
identity 的組合曾在實驗室 reference Host 上成功執行，不是一般容量、真實流量效能或
未來 revision 的保證。操作方式與目前支援介面仍以 Infrastructure repository 的
README 與 `docs/` 為準；本目錄只保存有日期的驗證證據。

## Current source baseline

2026-08-14 文件移轉時的 source identity：

- Infrastructure：`4f34413861d0f73d32c6efce444b1aaef6083272`
- NWDAF：`c53f05804c6533ec4c5fa7e230e90048fb219162`
- PyAnLF：`08798f15c3693027e00bc60dd53f74ebaa26c3a1`
- PyMTLF：`7e8ab7f23bf5d6398eb1cd5f053dd8bda9439a87`
- UPF：`234bae063ffb6a7c99b361bfcdb2bda9452af1f1`
- UERANSIM：`2a3ef81f189ca95d5c1996a28ed7af9734f5cfb4`
- gtp5g：`8d723c29fc0de3eeeff3e9a91132838579e8ee1b`
- WebConsole：`70d282f80e285d1fbae38f48c5c4ffefebde9944`

本 baseline 的完整 component inventory 仍由該 Infrastructure revision 的
`components.lock.yaml` 與 parent gitlinks 決定。

## Repository 與 preflight checks

截至本報告建立前，Infrastructure `make test` 已反覆通過。它在不啟動 production
VM/container stack 的情況下涵蓋：

- shell syntax、Python compilation、YAML 與 testbed schema；
- component gitlink、provisioning lock 與 resolved-version rules；
- native config rendering、跨檔案 drift rejection 與 PLMN-derived identities；
- persistent Netplan alias rendering 與 stale-address migration；
- deterministic PseudoDriver dataset generation、manifest/hash audit 與 tamper rejection；
- production／CPU-smoke Compose contracts；
- Consumer discovery/subscription contract、跨 thread/process atomic state、A/B callback accounting；
- current-invocation UE Registration/PDU readiness parsing；
- powered-off與backend failure的status語意；
- current-container FL milestone parsing與config identity隔離；
- Vagrant definition validation。

`make experiment-validate` 另曾在 reference Host 上通過 submodule cleanliness、
VirtualBox allowlist、dataset、RAM/storage、Docker access、Host SBI bind address 與
GPU CDI/CUDA probe。實驗期間持續出現的唯一 preflight warning 是 2 GiB swap 幾乎
全部使用；RAM 與 storage hard gates 均通過。

## Provisioning、network 與 5GC path

三台 Ubuntu 22.04 VM 已使用預設 Core/Path A/Path B `4/3/3 GiB` RAM 與各
`40 GiB` dynamic disk ceiling 完成 provisioning。Vagrant base addresses 與
repository-owned persistent Netplan aliases 已在 boot、config activation 與 migration
情境驗證；兩個 Path 會在 NF 啟動前檢查 gtp5g vermagic 與 running kernel。

完整 Guest lifecycle 曾啟動全部 23 個 units。六個 UE 均完成 Registration 與 PDU
Session：Path A 使用 `10.60.0.1`–`10.60.0.3`，Path B 使用
`10.61.0.1`–`10.61.0.3`。Consumer 經 NRF 找到兩個 distinct NWDAF provider，依
TAC `000001`／`000002` 建立兩筆 exact-location subscription，兩路 callbacks 均有
獨立計數證據。兩個 Path NWDAF 也都建立 Nupf Event Exposure resource，generated
PseudoDriver input 經正常 report path 送入。

## Latest full-core FL closure

NWDAF `c53f058` pin 後的 `full-core-cat-transition` GPU run 使用：

- Infrastructure 起始 revision：`c43c45c`
- Config：`blackbox-full-core-20260813`
- Effective config identity：`fb6f69881913…`
- Dataset set：`c3b428ea763834f34b2ff3a7e7674b5d082a2685e3825595f0b5cc33c356bb49`
- FL process：`61c95fe0-d019-45ea-9451-1f54d3872e85`
- Published model：`1786630122717`

本輪完成：

1. Path A degradation `evaluated=True triggered=True`；
2. A/B preparation 與 round 0、1 local training；
3. A/B 每輪各 6 samples，C 依 sample count 以各 `6/12` 執行 FedAvg；
4. final validation base WAPE `1.8392504554` 降至 candidate WAPE `0.3195272714`；
5. model 發布至 ADRF；
6. A/B scopes adoption 與 generation cutover complete；
7. 新 model 的 post-cutover accuracy `evaluated=true triggered=false`。

PyMTLF-A/B 使用 reference RTX 3080 的 `cuda:0`、NVIDIA runtime 與 CDI；PyAnLF-A/B
及 PyMTLF-C 使用 CPU。另有獨立 CPU disposable five-container lifecycle 通過 build、
health、status、logs、stop 與 scoped cleanup。空的五-service startup 約 1.3 GiB
container RSS，但此數字不含 active dataset、training tensors、model growth 或 VRAM，
不可當作完整實驗容量需求。

## Teardown revision boundary

較早的 2026-08-13 reports 記錄過 Model Monitor DELETE `503` 與 fixed 40-second
grace不足；那些結果屬於修正前的component/runtime組合，仍保留作歷史問題證據。

NWDAF 更新至 `c53f058` 後，上述 latest full-core run 的兩條 asynchronous Model
Monitor DELETE 都回 `204`，從第一個 Consumer DELETE 開始約 124 ms 內完成；沒有
cleanup `503` 或 reconciler failure。Infrastructure 後續移除依賴 application log wording
的 cleanup parser，改為 Consumer exact DELETE 後固定保留 40 秒，再停止 ML 與 Guest
backends。

這只證明 `c53f058` 組合的該次 teardown；不能推論網路故障或其他 revision 下永遠不會
超過 40 秒。若後續再出現 cleanup 疑慮，應保存三個 NWDAF、PyAnLF 與 PyMTLF-C 的
current-run logs另立報告。

## WebConsole

WebConsole 的首次 lazy-build、登入、subscriber read、billing loopback isolation 與 artifact
reuse 已於 2026-08-12 通過；2026-08-14 又在目前 Infrastructure lifecycle 上完成獨立
revalidation。最新結果見：

- [WebConsole Runtime Revalidation](webconsole-runtime-revalidation-2026-08-14.md)

本次 WebConsole 驗收沒有啟動 ML containers、Consumer、NWDAF subscriptions 或 FL。

## 結果保存要求

PseudoDriver 讓 experiment timing 與 traffic features 可重現，但不是 real application
traffic 或 user-plane throughput benchmark。未來報告新結果時至少應保存：

- Infrastructure parent revision 與 submodule gitlinks；
- config name/hash；
- dataset manifest/set ID；
- runtime image/artifact identity；
- scenario 與 CPU/GPU policy；
- Host resource observations與warning；
- 成功與失敗的current-run evidence。

缺少這些 identity 時，只能視為操作紀錄，不能併入此 baseline。

## Related detailed reports

- [Fresh User Full-core Black-box Acceptance](fresh-user-full-core-black-box-acceptance-2026-08-13.md)
- [Full-core Business FL E2E](full-core-business-fl-e2e-2026-08-11.md)
- [Observability Runtime Acceptance](observability-runtime-acceptance-2026-08-13.md)
- [Model Monitor Cleanup Runtime Validation](model-monitor-cleanup-runtime-validation-2026-08-13.md)
- [Optional WebConsole Lazy-Build Smoke](optional-webconsole-lazy-build-smoke-2026-08-12.md)
