# Optional WebConsole Lazy-Build Smoke（2026-08-12）

## 結論

Infrastructure 已完成 optional free5GC WebConsole 的 config、lazy build、systemd lifecycle、
aggregate experiment integration 與 bounded smoke。WebConsole 預設 disabled；只有完整 config
明確設定 `optionalServices.webconsole.enabled: true` 才會在 Core 安裝 toolchain、建置並啟動。
本輪沒有修改 WebConsole、NF 或 ML source，也沒有啟動 Host ML containers。

enabled smoke 通過 frontend HTTP、預設管理員登入、authenticated subscriber read、FTP listener
隔離、獨立 stop 與 artifact reuse。WebConsole stop 後其餘 23 個 Guest units 全部仍 active；
最終所有 services 均停止，三台 VM graceful poweroff。

## Identity 與環境

- Infrastructure 起始 revision：`6a58c466bcee8f6bac2675674d1e1ccdfcd289d3`；本報告涵蓋其上
  尚未提交的 WebConsole integration diff。
- WebConsole：`70d282f80e285d1fbae38f48c5c4ffefebde9944`，worktree clean。
- WebConsole artifact：
  `d28f97cce9c838c5aef8f69a07a7e607049d964f9e8193127d32f5147710b177`。
- 最終 runtime helper bundle：
  `8ba2f5895e0198c71816fc28e94ccfc055f98c3715af9a050a3bbb9fe94c4bed`。
- Config：ignored `webconsole-contract`，CPU `fl-closure-smoke`，effective config hash
  `9c7a7774f613099fe701beae608f2a7752dc352e3d95a8a7027b2f2034a49fe9`。
- Dataset：`2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`。
- Guest／provider：Ubuntu 22.04／VirtualBox；Host 起跑約 26 GiB available RAM、219 GiB
  filesystem free。

Repository tests 的 shell/Python syntax、negative config contract、disabled no-mutation、enabled
config、network renderer、deterministic dataset、Compose contract 與 Vagrant validation 全部通過。

## Upstream billing 相容性決策

WebConsole 原本於 2024-03 的 upstream PR #79 加入 `billingServer.enable`，讓沒有 CHF 的環境
可設為 `false`。隔日 PR #80 將 boolean validator 改為 `required,type(bool)`；所用 validator
把 `false` 視為 zero value，因此鎖定 revision 會在啟動前拒絕原本合法的 disabled 值。到本輪
查驗時 upstream 仍保留此 regression，也沒有找到後續公開修正。

使用者決定暫時不 fork、patch 或修改 submodule，而採 upstream 可接受的 `true`。Infrastructure
checker 逐欄固定以下 compatibility contract：control listener `127.0.0.1:2121`、passive range
`2123`-`2130`、base path `/tmp/webconsole` 與未使用的 CGF client port `2122`。實測只有
`127.0.0.1:2121` 監聽；沒有 FTP session 時 passive ports 不開 listener。HTTP 只綁 VirtualBox
host-only management address `192.168.56.10:5000`。Billing transfer、TLS 與 certificate 未測。

## 首次 build 與修正

第一次 enabled start 在 Core 安裝 Node.js `v20.20.2`，下載約 32.2 MB package，安裝後 package
footprint 約 196 MB。初版 helper 在 `runuser` 後仍由 `/home/vagrant` 呼叫 Corepack；runtime
帳號無權讀該目錄，Yarn install 前以 `EACCES` 結束。cleanup trap 移除 staged source、build
directory 與 release stage，沒有發布 incomplete artifact。

Infrastructure helper 隨後改為在執行 Yarn 前明確切到 staged `frontend/`。重試完成 Yarn
4.1.0 immutable install、Vite production build 與 Go server build。Yarn 回報既有 peer dependency、
TypeScript compatibility patch 與 bundle-size warnings，但 build exit 0；沒有修改 lock 或 source。
發布後 WebConsole runtime root 約 48 MB，其中 release 約 45 MB、Corepack cache 約 2.7 MB；
Core root filesystem 為 39 GiB、約 7.6 GiB used、32 GiB available。其他 package／Go caches 保留
在 Guest，普通 stop 不清除。

## Runtime smoke

完整 startup 先同步 runtime helpers、啟用同一 config、stage 相同 dataset，驗證 Core 14 個與
兩個 Path 各 7 個 network aliases，以及兩邊 gtp5g `5.15.0-187-generic` vermagic。23 個 Guest
units 全部 active，subscriber apply idempotently 寫入六個 UE 對應的 48 筆 documents 與一個
Internal Group。

WebConsole 隨後成功註冊 NRF、連接 MongoDB 並啟動：

- `GET /` 回覆 HTTP 200；
- `POST /api/login` 使用 `admin/free5gc` 取得 access token；
- 帶 `Token` header 的 `GET /api/subscriber` 回傳六筆 subscriber；
- Core `ss` 只看到 `192.168.56.10:5000` 與 `127.0.0.1:2121`；
- journal 顯示 WebConsole-AF NRF registration success、Billing listener 位址正確；
- stop 時 Billing、HTTP 與 NRF registration 都正常收斂，systemd unit 變成 inactive。

upstream `SetAdmin` 在每次 startup 都刪除並重建 WebConsole-owned `admin` tenant/user，因此本輪
登入帳密回到 `admin/free5gc`。Infrastructure start 現在會先輸出此 state-mutation warning。
這個動作沒有替換或刪除 config-owned 六筆 subscriber，但代表 WebConsole start 並非完全唯讀。

單獨執行 `webconsole-stop` 後，MongoDB、10 個其他 Core units、Path A 六個與 Path B 六個 units
仍全部 active。再次 `webconsole-start` 回報 artifact `reused`，沒有再次執行 Node/Yarn/Go build；
確認 HTTP ready 後再次獨立停止。

最後按既有 reverse order 停止 23 個 Guest units 並正常關閉三台 VM。artifact、Node toolchain、
config、dataset、MongoDB/subscriber state 依 lifecycle contract 保留，供下次 enabled start 重用。
