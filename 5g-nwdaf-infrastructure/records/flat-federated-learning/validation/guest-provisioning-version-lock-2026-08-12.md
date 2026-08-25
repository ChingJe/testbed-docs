# Guest Provisioning Version Lock

日期：2026-08-12

## 結論

`5G_NWDAF_Infrastructure` 的 Guest provisioning 不再依賴浮動的 Go archive 或
MongoDB metapackage。repository 現在以 `provisioning.lock.yaml` 記錄 Ubuntu 22.04
amd64、Go artifact identity，以及 MongoDB preferred／compatible package families。

Go artifact integrity 與 runtime identity 採 fail-closed。MongoDB 在全新 Guest 優先安裝
已驗證的 exact package set；現有完整套件或 repository 缺少 preferred patch 時，只要仍在
宣告的 compatible family，就保留／解析實際版本、顯示 warning 並寫入 manifest。partial
install、server cohort 混版、跨 series 或 signing-key mismatch 仍直接失敗。

本輪沒有修改 NF、ML、UERANSIM、gtp5g 或 WebConsole submodule，也沒有更動 N6
topology。只為 read-only audit 暫時啟動 Core VM；未執行 provisioning、未升降級 package、
未新增 apt hold、未啟動 5G service，完成後已 graceful poweroff。Path A/B 全程保持關閉。

## Locked identity

| Component | Preferred identity | Compatible policy |
|---|---|---|
| Guest platform | Ubuntu 22.04 amd64 | 不符即停止 |
| Go | `1.26.2`, official linux-amd64 archive | URL、SHA-256、binary identity 任一不符即停止 |
| MongoDB server cohort | `8.0.28` | 完整 cohort 可接受 `8.0.x`，drift warning |
| mongosh | `2.9.2` | 可接受 `2.x`，drift warning |
| MongoDB database tools | `100.17.0` | 可接受 `100.x`，drift warning |
| MongoDB signing key | `4B0752C1BCA238C0B4EE14DC41DE058A4E7DCA05` | fingerprint 不符即停止 |

Go archive SHA-256 為
`990e6b4bbba816dc3ee129eaeaf4b42f17c2800b88a2166c265ac1a200262282`；本次 lock
file SHA-256 為
`23b2e8bb0560f66f5a93db09d0a79ec0e08dbda9e94038cd7576ae04581fc34f`。

## Existing Core audit

現有 Core VM 為 Ubuntu 22.04 amd64、kernel `5.15.0-171-generic`，已安裝：

```text
Go                              1.26.2 linux/amd64
mongodb-org                     8.0.28
mongodb-org-database            8.0.28
mongodb-org-server              8.0.28
mongodb-org-shell               8.0.28
mongodb-org-mongos              8.0.28
mongodb-org-tools               8.0.28
mongodb-org-database-tools-extra 8.0.28
mongodb-mongosh                 2.9.2
mongodb-database-tools          100.17.0
```

resolver 對此完整 installed set 回報 `source=installed`、`drift=false`，因此未來再次
provision 時會保留它，不因 apt candidate 已有 MongoDB server `8.0.29` 就自動升級。
現況沒有 package hold；hold 只會由新版 Core provisioning 在 lock 驗證成功後寫入。

現有 MongoDB key fingerprint 與 lock 相符。另以官方 URL重新下載 Go archive 與 MongoDB
public key，實際驗證 archive SHA-256、Go binary identity 與 signing-key fingerprint，均相符。

## Disposable clean-install smoke

`scripts/host/provisioning-install-smoke.sh` 在一次性 `ubuntu:22.04` container 內執行：

1. 驗證 lock schema 與 package coverage；
2. 從 official URL 下載 Go archive，核對 SHA-256、解壓並執行 binary identity check；
3. 從 official URL 下載 MongoDB key 並核對 fingerprint；
4. 設定 MongoDB `jammy/mongodb-org/8.0` repository；
5. 確認 resolver 選出 preferred package set 且無 drift；
6. 使用 `package=version` 實際安裝全部九個必要 packages；
7. 由 installed-state resolver 再次核對完整 cohort且無 drift。

安裝結果為 server `8.0.28`、mongosh `2.9.2`、database tools `100.17.0`，smoke
成功後容器由 `--rm` 移除。此項測試不會碰觸既有 Docker containers。

## Regression coverage

repository smoke 覆蓋 preferred exact resolution、compatible repository fallback、compatible
installed preservation，以及以下 hard failures：

- partial MongoDB package set；
- cross-series package；
- mixed server cohort；
- repository 無共同 compatible version；
- 無效 Go SHA-256；
- lock 缺少必要 MongoDB package mapping。

完整 `make test`、shell syntax、Python compile、lock validation、config/network/dataset、
Compose 與 `vagrant validate` 均通過。新 provisioning 完成後，每台 Guest 會將 requested／
resolved identity、lock hash、OS／kernel 與 drift reason 原子寫入
`/etc/5g-nwdaf-infrastructure/provisioning-manifest.yaml`，供後續稽核與實驗證據使用。
