# Operator Command And Test Layout Cleanup

日期：2026-08-13

## 結論

`5G_NWDAF_Infrastructure` 已將 Make command surface 從 55 個 targets 收斂為 38 個，並將
test-only runner/helper 從 `scripts/host/` 移至獨立 `tests/`。使用者不再看到細項 regression、
內部 implementation name 或指向同一腳本的重複 aliases；正式 start/validate 流程使用的必要
檢查仍保留在 `scripts/host/` 並自動執行。

本輪沒有修改任何 submodule、gitlink、committed config、scenario、PseudoDriver dataset、VM
或 production service state。`fl-closure-smoke` 仍是可執行的 bounded experiment example，沒有
因 test layout 清理而更名或移動。最後另建立過獨立的 CPU test project，完成後已移除它的
containers、volumes、network 與 generated config，只保留共用 images。

## Final command boundary

保留的 38 個 Make targets 分為：

- help：`help`、`help-advanced`、`help-dev`、`help-all`；
- aggregate experiment：`experiment-validate`、`experiment-start`、`experiment-status`、
  `experiment-stop`；
- config/dataset：`config-create`、`config-validate`、`dataset-generate`、`dataset-validate`、
  `dataset-show`、`dataset-load`；
- VM 與 process domains：三組 `vm-*`、`services-*`、`ml-*`、`webconsole-*`、
  `subscriptions-*`；
- retained data：`subscriber-data-show`、`subscriber-data-apply`、`subscriber-data-clear`、
  `reset-show`、`reset`；
- observation：`observe`、`logs`；
- tests：`test`、`test-containers`。

`observe` 已加入 advanced help，明確表示它是持續狀態觀察，不是 test。`make test` 是唯一
Host-only repository regression 入口；`make test-containers` 因會建立 disposable containers 而
保持獨立。

移除的 Make targets 包括四個細項 regression target、兩個 container-test aliases，以及
`config-check`、`config-render`、`dataset-check`、`dataset-stage*`、`preflight`、subscriber-data
validate/plan、experiment-reset plan/apply/verify 等內部名稱。保留的 user-facing target 現在直接
呼叫同一個既有 helper，因此沒有減少實際 validation、dataset staging 或 reset verification。

## Test layout

Repository tests 現在集中為：

```text
tests/
├── repository.sh
├── config-contract.py
├── dataset-determinism.sh
├── mobile-identity.py
├── network-config.py
├── provisioning-lock.py
├── testbed-definition.py
├── ml-container-lifecycle.sh
└── support/
    └── ml-cpu-config.py
```

這些是原有 tests 的一對一分類與命名整理，沒有增加新的 regression 種類。config、mobile
identity、Netplan、dataset、provisioning lock 與 single-testbed tests 保留，因為它們分別保護曾
發生過的 config mismatch、PLMN derivation、persistent alias、dataset integrity、Guest dependency
與 hidden local override 問題。ML container lifecycle 仍由獨立 command 執行。

`scripts/host/provisioning-install-smoke.sh` 是唯一直接刪除而未搬移的 test。它沒有任何 caller，
會重新下載並在 disposable Ubuntu 22.04 container 安裝完整 Go/MongoDB package set，且其
clean-install evidence 已保存在 Guest provisioning version-lock report，不適合作為日常 command。

整理後 `scripts/host/` 只保存 experiment/process lifecycle、config/dataset runtime、Host/gtp5g
preflight、subscriber/reset、observation/logs 及其 shared helpers。`scripts/guest/` 與 runtime
Consumer 不受影響。

## Validation

以下檢查通過：

- `make help-all` 只顯示最終 operator、advanced 與兩個 test commands；
- Makefile target definition count 為 38；
- 所有 `scripts/` 與 `tests/` shell/Python syntax；
- moved test paths 與 removed Make names 的 repository reference search；
- 完整 `make test`，包含 config negative contracts、二／三位 MNC、provisioning lock、
  single-testbed、WebConsole render、七個 Netplan cases、deterministic/tampered dataset、Compose
  baseline/CPU、stale local/provider rejection 與 Vagrant validation；
- read-only `make experiment-validate CONFIG_DIR=config/default`。
- disposable `make test-containers` container lifecycle regression。

Aggregate validation 為 0 failures、1 個既有 low-swap warning。當時約 27,313 MiB available RAM、
216 GiB free storage；VirtualBox、Docker、Host SBI ports、16 個 component locks、default config、
generated dataset、GPU CDI/runtime 與 Vagrantfile 全部通過。這證明移除 `preflight` 與
`ml-compose-check` Make targets 後，正式 `experiment-validate` 仍會執行兩者的必要行為。

`make test-containers` 實際建立五個獨立 CPU containers，全部達到 healthy，使用 UID 10001，且
PyMTLF-A/B 的 configured training device 均為 CPU。測試驗證 logs、status、約 1.3 GiB aggregate
empty-service RSS、stop，以及五個 stopped containers／五個 volumes 的 retained-state contract，
隨後確認 project containers、volumes、network 與 `config/generated/ml-container-test` 全部清除；
production project、VM、Guest services 與 subscriptions 未受影響。
