# Path A VM Provisioning Smoke（2026-08-09）

本報告記錄新版 `5G_NWDAF_Infrastructure` Path A VM 的第一次完整 provisioning。範圍包含
Guest toolchain、gtp5g kernel module、UPF、NWDAF 與 UERANSIM build，以及 disabled service
baseline；沒有啟動 UPF、gNB、UE、NWDAF、Host ML containers 或 subscription，也沒有執行
registration、PDU Session 或 PseudoDriver replay。

## 執行前條件

- Infrastructure revision：`02f7ca4`；
- Core 已完成 provisioning 並 poweroff；Path B poweroff；
- Host preflight：0 failures、2 warnings，warnings 仍為 provider 未寫入 local YAML 與 free
  swap 約 1 MiB；
- provisioning 前 Host 約有 27 GiB `MemAvailable`、170 GiB workspace free；
- 八個既有 Docker containers 保持 running。

執行命令：

```text
VAGRANT_DEFAULT_PROVIDER=virtualbox vagrant up path-a --provision
```

命令以 exit 0 完成。Vagrant 重新套用 Path A 的六張 host-only interfaces、rsync 受控 source
snapshot，再依序執行 `common.sh path-a` 與 `path.sh A setup`。

## gtp5g 結果

gtp5g `v0.9.16` 已針對 running kernel `5.15.0-171-generic` 成功編譯及安裝：

```text
filename: /lib/modules/5.15.0-171-generic/kernel/drivers/net/gtp5g.ko
version:  0.9.16
vermagic: 5.15.0-171-generic SMP mod_unload modversions
```

編譯輸出提示 kernel 使用的 GCC package revision 是 `11.4.0-...22.04.2`，目前 Guest compiler
是 `11.4.0-...22.04.3`；major/minor compiler 相同，module 最終成功 load，未出現 symbol／
vermagic error。

需要注意：`path.sh` 雖未直接執行 `modprobe`，gtp5g upstream `make install` target 會執行
`modprobe udp_tunnel`、`modprobe gtp5g`，並寫入 `/etc/modules-load.d/gtp5g.conf`。驗收時：

```text
gtp5g      135168  0
udp_tunnel  20480  1 gtp5g
```

這是 Path Guest kernel dependency，不是 UPF process；但後續文件與 lifecycle 判斷不應再宣稱
provisioning 完全不載入 kernel module。

## Guest build 與狀態

- OS：Ubuntu 22.04.5 LTS；Go：`go1.26.2 linux/amd64`；
- installed Go binaries：`upf` 27,238,303 bytes、`nwdaf` 25,719,055 bytes；
- UERANSIM 2091-step Ninja build 完成，產生 `nr-gnb`、`nr-ue`、`nr-cli`；
- source snapshot：106 MiB；build work tree：234 MiB；
- RAM：2968 MiB total，驗收時約 2592 MiB available；Guest 無 swap；
- root disk：39 GiB filesystem、約 3.8 GiB used、35 GiB available；
- `5g-nwdaf-stack.target` inactive，active config symlink 不存在；
- `5g-nwdaf-network.service` enabled，但 `ConditionResult=no`、state inactive。

Guest 只有 Vagrant base addresses：

```text
192.168.56.11/24  management
192.168.57.3/24   SBI anchor
192.168.58.3/24   N2 anchor
192.168.59.2/24   N3-A anchor
192.168.61.3/24   N4 anchor
192.168.62.2/24   N6-A anchor
```

UPF Event Exposure `.57.40`、NWDAF-A `.57.41`、gNB `.58.20/.59.20` 與 UPF
`.59.10/.61.20/.62.10` 尚未出現，符合 config activation 才套用 process aliases 的設計。

## PseudoDriver dataset identity

兩組 dataset 都隨 pinned UPF source snapshot 進入 Guest；Path A runtime 後續只會選用 group1：

| Dataset | Bytes | SHA-256 |
| --- | ---: | --- |
| `group1/training_packets_run001.parquet` | 21,830,425 | `d9e2772de8529870e272f44a3bc02863e8831d9c90d51eb0b33961eb28a29030` |
| `group2/training_packets_run001.parquet` | 44,050,349 | `b8482a21f3370de491a67fa1f9908e1b8b3aec7671787bbbdb0d9287680b662e` |

檔案大小與 hash 和 committed testbed／manifest 一致；本輪沒有讀入 dataset 或量測 replay RAM。

## Host 影響與驗證邊界

Provisioning 後、Path A running 時，Host 約有 24 GiB `MemAvailable`、169 GiB workspace free；
八個既有 Docker containers 均保持 running。這輪證明 Jammy Path VM 能建置並 load pinned gtp5g，
且 UPF、NWDAF、UERANSIM source 可完成 build。

尚未驗證：

- `network/path-a.yaml` 的 alias 套用與 rollback；
- UPF bind、PFCP association、EES/PseudoDriver behavior；
- gNB SCTP、UE registration、PDU Session 或 N3/N6 traffic；
- dataset replay peak RAM；
- Path B build 與三 VM service startup。

下一個 bounded step 是 graceful halt Path A，再以相同流程 provision Path B。三台完成後才進行
完整 config activation 與 dependency-ordered service smoke。
