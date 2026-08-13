# 5G NWDAF Infrastructure 驗證報告

此目錄集中保存新版 `5G_NWDAF_Infrastructure` 的主機準備、容器 runtime、VM
骨架與 provisioning 驗證紀錄。舊 `5G_Infrastructure`、Daisy、事件與 replay
相關報告仍保留在上一層目錄，避免混淆兩套環境的時間線。

目前整體驗證狀態先看：

- [Current Runtime Validation Baseline（2026-08-14）](current-runtime-validation-baseline-2026-08-14.md)

以下文件是各階段的詳細或歷史證據；結果應連同報告日期、Infrastructure 與
component revision 解讀，不能視為目前所有 revision 的永久保證。

## Host 與 ML runtime

- [Hybrid Host Readiness Inventory](hybrid-host-readiness-inventory-2026-08-09.md)
- [Host ML Container CPU Smoke](host-ml-container-cpu-smoke-2026-08-09.md)
- [Host ML Lifecycle Smoke](host-ml-lifecycle-smoke-2026-08-09.md)
- [Host GPU Runtime Activation](host-gpu-runtime-activation-2026-08-09.md)

## VM 骨架

- [Core VM Skeleton Smoke](core-vm-skeleton-smoke-2026-08-09.md)
- [Path VM Skeleton Smoke](path-vm-skeleton-smoke-2026-08-09.md)

## VM provisioning

- [Core VM Provisioning Smoke](core-vm-provisioning-smoke-2026-08-09.md)
- [Path A VM Provisioning Smoke](path-a-vm-provisioning-smoke-2026-08-09.md)
- [Path B VM Provisioning Smoke](path-b-vm-provisioning-smoke-2026-08-09.md)
- [Guest Provisioning Version Lock](guest-provisioning-version-lock-2026-08-12.md)

## Guest services 與 5GC flow

- [Guest Services 與 UE Registration Smoke](guest-services-and-ue-registration-smoke-2026-08-09.md)
- [Host ML 與 Guest Stack 整合 Smoke](host-ml-guest-stack-integration-smoke-2026-08-09.md)
- [Persistent Netplan Alias Migration](persistent-netplan-alias-migration-2026-08-12.md)
- [PLMN Single-Source Config Regression](plmn-single-source-config-regression-2026-08-12.md)
- [Single Testbed Definition Regression](single-testbed-definition-regression-2026-08-13.md)

## Repository contract 與文件

- [Atomic Repository Documentation Restructure](atomic-repository-documentation-restructure-2026-08-13.md)
- [Operator Command And Test Layout Cleanup](operator-command-and-test-layout-cleanup-2026-08-13.md)

## Optional management

- [Optional WebConsole Lazy-Build Smoke](optional-webconsole-lazy-build-smoke-2026-08-12.md)
- [WebConsole Runtime Revalidation](webconsole-runtime-revalidation-2026-08-14.md)

## PseudoDriver data path

- [Generated PseudoDriver Dataset Tooling](generated-pseudodriver-dataset-tooling-2026-08-09.md)
- [PseudoDriver Dataset Guest Staging Smoke](pseudodriver-dataset-guest-staging-smoke-2026-08-09.md)
- [Nupf Contract 與 PseudoDriver Runtime Smoke](nupf-contract-pseudodriver-runtime-smoke-2026-08-09.md)

## Full-core E2E

- [Stateless Full E2E Smoke](stateless-full-e2e-smoke-2026-08-10.md)
- [Full-core Business FL E2E](full-core-business-fl-e2e-2026-08-11.md)
- [FL Closure Smoke 與 Persistent Netplan 回歸](fl-closure-smoke-netplan-regression-2026-08-12.md)
- [Runtime Helper Sync 與 FL Lifecycle 回歸](runtime-helper-sync-and-fl-lifecycle-regression-2026-08-12.md)
- [Fresh User Full-core Black-box Acceptance](fresh-user-full-core-black-box-acceptance-2026-08-13.md)
- [Model Monitor Cleanup Runtime Validation](model-monitor-cleanup-runtime-validation-2026-08-13.md)
- [Observability Runtime Acceptance](observability-runtime-acceptance-2026-08-13.md)
