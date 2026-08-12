# Single Testbed Definition Regression

日期：2026-08-13

## 結論

`5G_NWDAF_Infrastructure` 已移除不完整的 `testbed.local.yaml` override layer。
`TESTBED` 現在選擇唯一、完整且自足的 environment/topology definition；未指定時為
`testbed.yaml`。`CONFIG_DIR` 顯式選擇完整 config set，未指定時只 fallback 至 selected
testbed 的 `config.directory`。

本輪只修改 Host orchestration、repository tests 與現行入口文件。沒有修改 N6、任何
NF／ML／RAN submodule、config/default 內容、Guest runtime 或既有 VM definition，也沒有啟動、
重建或 provision VM。

## Removed local fields

舊 local 範本的五項設定依責任分類後全部移除，而不是移到另一個隱藏 overlay：

| 舊欄位 | Effective source |
|---|---|
| `provider.name` | reference runtime 固定 VirtualBox |
| `provider.expectedVmStorage` | `VBoxManage` 回報的實際 machine folder |
| `host.dockerDataRoot` | `docker info` 回報的實際 data root |
| `host.mlBindAddress` | selected testbed 的 `mlRuntime.bindAddress` |
| `config.directory` | 顯式 `CONFIG_DIR`，否則 selected testbed 的 `config.directory` |

因此 `testbed.local.example.yaml` 與其 `.gitignore` rule 已刪除，`configlib.py` 不再包含
local loader。Vagrantfile 與 preflight 在發現舊 `testbed.local.yaml` 時明確停止並提供遷移
訊息，避免舊內容被 silent ignore。未設定 `VAGRANT_DEFAULT_PROVIDER` 時直接使用
VirtualBox；若環境變數要求其他 provider，則明確拒絕。

## Resolution regression

新增 `scripts/host/testbed-definition-smoke.py`，驗證：

- committed `testbed.yaml` 提供 `config/default` 與 `192.168.57.1` ML bind address；
- alternate selected testbed 的 config directory 與 ML bind address 實際生效；
- explicit `CONFIG_DIR` 高於 selected testbed default；
- selected testbed 缺少 `config.directory` 或 `mlRuntime.bindAddress` 時明確失敗；
- shared Python resolution source 不再引用 `testbed.local` 或 local loader。

repository test 另在 temporary Vagrant root 建立 stale `testbed.local.yaml`，確認
`vagrant validate` 失敗且包含 migration evidence；刪除該 temporary 檔後，以
`VAGRANT_DEFAULT_PROVIDER=libvirt` 確認非 VirtualBox provider 同樣被拒絕。正常 selected
`testbed.yaml` 最後通過 `vagrant validate`。

## Host preflight

不提供 local YAML 或 provider environment 的實際 read-only preflight 結果為 0 failures、
1 warning：

- VirtualBox host driver 與 host-only allowlist 正常；
- 自動找到 VM storage `/home/chingje/VirtualBox VMs`；
- 自動找到 Docker data root `/var/lib/docker`；
- workspace、VM storage 與 Docker filesystem 各有約 216 GiB free；
- 約 28,452 MiB `MemAvailable`，高於 10,240 MiB VM allocation 加 6,144 MiB reserve；
- selected testbed 的 `192.168.57.1` bind address 存在，五個 ML ports 均可用；
- config、component locks、submodules 與 generated PseudoDriver dataset 均通過。

唯一 warning 是 free swap 約 2 MiB，低於 1 GiB warning threshold；RAM hard gate 通過，
此 warning 與 single-testbed change 無關。

## Full regression

完整 `make test` 通過 shell／Python syntax、config negative contracts、PLMN derivation、
provisioning lock、WebConsole disabled/enabled render、network alias、deterministic dataset、
Compose baseline/CPU、stale-local/provider negatives 與正常 Vagrant definition validation。
測試沒有改變 VM、container 或 service lifecycle。
