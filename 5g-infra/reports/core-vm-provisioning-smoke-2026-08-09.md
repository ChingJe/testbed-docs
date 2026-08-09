# Core VM Provisioning Smoke（2026-08-09）

本報告記錄新版 `5G_NWDAF_Infrastructure` Core VM 的第一次完整 provisioning。此次範圍是
Guest 套件安裝、source staging、Go binary build 與 disabled service baseline；沒有啟動
MongoDB、5GC、NWDAF、Host ML containers 或 subscription，也不是 full-core E2E 結果。

## 執行前條件

- Infrastructure revision：`02f7ca4`；
- provider：既有 VirtualBox Core VM；
- config：`config/default`，checker hash
  `4a07522a4390c67661517fcd5df1da5daa36b0f78319e64ecf07961f2b633ab7`；
- Host preflight：0 failures、2 warnings；warnings 是未在 local YAML 固定 provider，以及
  free swap 約 1 MiB、低於 1 GiB warning threshold；
- provisioning 前 Host 約有 27 GiB `MemAvailable`、175 GiB workspace free；
- 八個既有 Docker containers 保持 running。

執行命令：

```text
VAGRANT_DEFAULT_PROVIDER=virtualbox vagrant up core --provision
```

命令以 exit 0 完成。Vagrant 使用既有 Jammy VM，重新套用四張 host-only interface、rsync
受控 source snapshot，再依序執行 `common.sh core` 與 `core.sh setup`。

## Guest 結果

- OS：Ubuntu 22.04.5 LTS；kernel：`5.15.0-171-generic`；
- RAM：3911 MiB total，驗收時約 3426 MiB available；Guest 無 swap；
- root disk：39 GiB filesystem、約 5.2 GiB used、34 GiB available；
- source snapshot：106 MiB；build work tree：289 MiB；
- Go：`go1.26.2 linux/amd64`；MongoDB：8.0.28；
- 已建立 `nrf`、`nssf`、`udr`、`udm`、`ausf`、`pcf`、`amf`、`smf`、`adrf`、
  `nwdaf` 共十個 installed binaries；
- `mongod.service`、`5g-nwdaf-stack.target` 均為 inactive；
- `/etc/5g-nwdaf-infrastructure/active` 不存在，因此沒有 active config set；
- `5g-nwdaf-network.service` 已安裝並 enabled，但 `ConditionResult=no`、state inactive；這表示
  VM boot/provisioning 沒有在 config activation 前建立 process aliases。

Guest 只有 Vagrant base addresses：

```text
192.168.56.10/24  management
192.168.57.2/24   SBI anchor
192.168.58.2/24   N2 anchor
192.168.61.2/24   N4 anchor
```

NRF `.57.10`、AMF `.57.16/.58.10`、SMF `.57.17/.61.10`、ADRF `.57.19`、NWDAF-C
`.57.30` 與 consumer `.57.32` 尚未出現，符合「config activation 才套用 process alias」的
lifecycle。

## Host 影響與驗證邊界

Provisioning 後 Host 約有 23 GiB `MemAvailable`、171 GiB workspace free；八個既有 Docker
containers 仍保持原本 running 狀態。Guest Additions 6.0 與 Host VirtualBox 6.1 仍有版本提示，
但 repository 使用 rsync、停用 `/vagrant` share，這次 provisioning 沒有出現 shared-folder
failure。

本輪證明 Core Guest 可在 pinned Jammy baseline 完成 package install 與十個 Go component
build，並證明無 active config 時 network/service lifecycle 保持停止。尚未驗證：

- config archive upload、hash activation 與 `network/core.yaml` 實際 alias 套用／rollback；
- MongoDB 與各 NF process start、bind、registration 或 dependency order；
- Path gtp5g／UPF／UERANSIM build；
- Host ML、subscription、PseudoDriver 與 full business E2E。

下一個 bounded step 是只 provision Path A；成功後 graceful halt 或維持單一路徑資源範圍，再
以同樣方式 provision Path B。三台都完成後，才使用正式 `services-start` 一次驗證完整 config
activation、role-specific aliases 與 Guest process startup。
