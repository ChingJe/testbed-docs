# Atomic Repository Documentation Restructure

日期：2026-08-13

## 結論

`5G_NWDAF_Infrastructure` 已從 11 個分散、重疊且部分過時的 parent-owned Markdown 入口，
一次收斂為 root `README.md` 與 `docs/` 下九份 topic guides。最短可執行的基礎實驗流程留在
root README；詳細標準流程在 `docs/operations.md`；從 UE registration 到 federated model
generation cutover 的完整概念流程在 `docs/architecture.md`。

本輪沒有修改任何 NF／ML／RAN／gtp5g／WebConsole submodule source、gitlink、testbed、config、
scenario、N6、VM 或 runtime state。

## Canonical hierarchy

現行 repository 文件固定為：

```text
README.md
docs/
├── README.md
├── architecture.md
├── installation.md
├── configuration.md
├── operations.md
├── commands.md
├── components.md
├── validation.md
└── troubleshooting.md
```

root README 只保留 purpose、placement 摘要、第一次實驗、停止方式、導航與 license。其餘文件
分別管理 runtime architecture、Host installation、single-TESTBED/config/dataset contract、標準與
進階操作、完整 Make command surface、source locks、目前 verified behavior 與 troubleshooting。

`OPERATIONS.md`、`COMPONENTS.md`、四個分類 README、config README、Host/Guest scripts README
及 Consumer README 已直接刪除。新 hierarchy 已吸收仍適用內容，因此沒有留下 redirect、archive
或 compatibility copy，也不保留已被目前 E2E evidence 推翻的舊 limitation。

## Basic experiment flow

文件將使用者主路徑固定為：

1. recursive submodule initialization；
2. `config-create` 建立 complete config set；
3. `dataset-generate` 產生 content-addressed Path A/B Parquet；
4. read-only `experiment-validate`；
5. `vm-up`；
6. `experiment-start`；
7. `experiment-status` 與 logs；
8. `experiment-stop`，需要時再 `vm-halt`。

文件明確區分 first/retained-state run 與 clean subsequent run。reset 不是每次 start 的必要步驟；
只有下一次實驗不可繼承 ML、ADRF、model 或 ADRF registration state 時，才先 review
`reset-show`，再以 scenario name 作 `RESET_CONFIRM` 執行 scoped reset。

## Component metadata contract

`.gitmodules` 的 16 個 submodule URL 原本已全部使用 HTTPS，但 `components.lock.yaml` 仍有九個
team repository 保存 SSH metadata。本輪將這九個 remote metadata 對齊 HTTPS，沒有改 branch、
commit 或 installed checkout。

既有 preflight 繼續驗證 installed HEAD 與 readable lock commit，並檢查 submodule worktree clean
state。local submodule `origin` 不作 hard contract，以保留合法 credential rewrite。這次不新增
component metadata 專用命令或獨立驗證腳本。

## Documentation verification

文件 review 確認：

- parent-owned Markdown 只能是 root README 與九份 canonical docs；
- 十個舊路徑不得重新出現；
- 所有相對 Markdown links 必須存在；
- Makefile 的每個 target 都在 `docs/commands.md` 出現。

結果為 10 份 canonical Markdown、所有相對連結有效，且現有 Make targets 全部有功能與
state-effect 說明。這些是此次 atomic migration 的 review evidence，不增加使用者 command
surface。

## Validation evidence

`make help-all`、shell syntax、Python compilation 與 `git diff --check` 全部通過。

完整 `make test` 通過 config negative contracts、mobile identity、component/provisioning locks、
single-testbed resolution、WebConsole disabled/enabled render、Netplan、deterministic/tamper-resistant
dataset、Compose baseline/CPU、stale local rejection、non-VirtualBox rejection 與 Vagrant definition
validation；沒有啟動 VM、container 或 service lifecycle。

實際 Host read-only preflight 為 0 failures、1 warning：

- available RAM 約 26,625 MiB，高於 10,240 MiB VM allocation 加 6,144 MiB Host reserve；
- workspace、VirtualBox storage 與 Docker data-root filesystem 均約 216 GiB free；
- VirtualBox driver/allowlist、Docker daemon、Host SBI bind address 與五個 ML ports 正常；
- submodule gitlinks、clean worktrees、16 個 component metadata、default config 與 generated dataset
  全部通過。

唯一 warning 為 free swap 約 2 MiB，低於 1 GiB warning threshold；RAM hard gate 通過，且本輪
沒有調整 Host swap。
