# Path B VM Provisioning Smoke（2026-08-09）

本報告記錄新版 `5G_NWDAF_Infrastructure` Path B VM 的第一次完整 provisioning。範圍包含
Guest toolchain、gtp5g kernel module、UPF、NWDAF 與 UERANSIM build，以及 disabled service
baseline；沒有啟動 UPF、gNB、UE、NWDAF、Host ML containers 或 subscription，也沒有執行
registration、PDU Session 或 PseudoDriver replay。

## 執行前條件

- Infrastructure revision：`02f7ca4`；
- Core 與 Path A 已完成 provisioning 並 poweroff；
- Host preflight：0 failures、2 warnings，warnings 為 provider 未寫入 local YAML 與 free swap
  約 1 MiB；
- provisioning 前 Host 約有 27 GiB `MemAvailable`、168 GiB workspace free；
- 八個既有 Docker containers 保持 running。

執行命令：

```text
VAGRANT_DEFAULT_PROVIDER=virtualbox vagrant up path-b --provision
```

命令以 exit 0 完成。Vagrant 重新套用 Path B 的六張 host-only interfaces、rsync 受控 source
snapshot，再依序執行 `common.sh path-b` 與 `path.sh B setup`。

## Guest 結果

- OS：Ubuntu 22.04.5 LTS；kernel：`5.15.0-171-generic`；
- Go：`go1.26.2 linux/amd64`；
- gtp5g 0.9.16 成功針對 running kernel 編譯、安裝並由 upstream `make install` load；
- module path：`/lib/modules/5.15.0-171-generic/kernel/drivers/net/gtp5g.ko`；
- vermagic：`5.15.0-171-generic SMP mod_unload modversions`；
- 驗收時 `gtp5g` 與 `udp_tunnel` 已 load、usage count 為 0；
- installed Go binaries：`upf` 27,238,303 bytes、`nwdaf` 25,719,055 bytes；
- UERANSIM 2091-step Ninja build 完成，產生 `nr-gnb`、`nr-ue`、`nr-cli`；
- source snapshot：106 MiB；build work tree：234 MiB；
- RAM：2968 MiB total，驗收時約 2598 MiB available；Guest 無 swap；
- root disk：39 GiB filesystem、約 3.8 GiB used、35 GiB available；
- `5g-nwdaf-stack.target` inactive，active config symlink 不存在；
- `5g-nwdaf-network.service` enabled，但 `ConditionResult=no`、state inactive。

gtp5g 同樣出現 kernel GCC package revision `22.04.2` 與目前 compiler `22.04.3` 的提示；兩者
都是 GCC 11.4.0，module 最終成功 load，沒有 symbol／vermagic error。

Guest 只有 Vagrant base addresses：

```text
192.168.56.12/24  management
192.168.57.4/24   SBI anchor
192.168.58.4/24   N2 anchor
192.168.60.2/24   N3-B anchor
192.168.61.4/24   N4 anchor
192.168.63.2/24   N6-B anchor
```

UPF Event Exposure `.57.50`、NWDAF-B `.57.51`、gNB `.58.30/.60.20` 與 UPF
`.60.10/.61.30/.63.10` 尚未出現，符合 config activation 才套用 process aliases 的設計。

## PseudoDriver dataset identity

兩組 dataset 都隨 pinned UPF source snapshot 進入 Guest，hash 與 committed manifest 相符；
Path B runtime 後續只會選用 group2：

| Dataset | SHA-256 |
| --- | --- |
| `group1/training_packets_run001.parquet` | `d9e2772de8529870e272f44a3bc02863e8831d9c90d51eb0b33961eb28a29030` |
| `group2/training_packets_run001.parquet` | `b8482a21f3370de491a67fa1f9908e1b8b3aec7671787bbbdb0d9287680b662e` |

本輪沒有讀入 dataset 或量測 replay RAM。

## Host 影響與驗證邊界

Provisioning 後、Path B running 時，Host 約有 24 GiB `MemAvailable`、166 GiB workspace free；
八個既有 Docker containers 均保持 running。加上 Core 與 Path A 的前序結果，三台 Guest 都已
完成各自的 pinned build responsibility。

尚未驗證：

- `network/path-b.yaml` 的 alias 套用與 rollback；
- UPF bind、PFCP association、EES/PseudoDriver behavior；
- gNB SCTP、UE registration、PDU Session 或 N3/N6 traffic；
- dataset replay peak RAM；
- 三 VM 完整 config activation 與 dependency-ordered service startup。

下一個 bounded step 是 graceful halt Path B。之後三台同時開機，但先只 stage/activate default
config 並驗證三份 role-specific network YAML 的 aliases；若 activation 正常，再進入完整 Guest
service startup smoke。
