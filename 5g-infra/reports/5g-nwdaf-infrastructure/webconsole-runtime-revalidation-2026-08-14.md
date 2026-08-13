# WebConsole Runtime Revalidation（2026-08-14）

## 結論

Optional WebConsole 在目前 Infrastructure lifecycle 上再次通過獨立 runtime 驗收。
Host 端以 HTTP/API request 取代瀏覽器操作，驗證 frontend assets、admin login、authenticated
subscriber read、billing loopback bind、artifact reuse 與獨立 stop。全程沒有啟動 Host ML
containers、Consumer、NWDAF subscriptions 或 FL。

## Identity 與前置條件

- Infrastructure：`4f34413861d0f73d32c6efce444b1aaef6083272`
- WebConsole：`70d282f80e285d1fbae38f48c5c4ffefebde9944`
- Config：`config/local/observability-acceptance-20260813`
- Static config validation hash：`4c03aeb446b40eecc44c88f81b7c9da7149787444b318e412eb0298ab579650c`
- Effective config hash：`d466833f1c21064289d85be4832a57feac13c9a425cfccc2f9c0a9f82271c02f`
- Dataset set：`2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`
- Artifact identity：`d28f97cce9c838c5aef8f69a07a7e607049d964f9e8193127d32f5147710b177`
- Guest kernel／gtp5g check：Path A/B `5.15.0-171-generic`，vermagic matched、loaded

`make experiment-validate` 為 0 failures、1 warning。約 28.1 GiB RAM available、
220 GiB workspace free；warning 是 free swap約 2 MiB。16個component locks、config、
dataset、VirtualBox、Docker、GPU runtime/CDI均通過。

## Startup boundary

三台既有 VM 正常 boot，沒有重新 provision。`make services-start`：

- staged同一份config與Path A/B dataset；
- verified Core 14個、Path A/B各7個persistent network aliases；
- 啟動全部23個Guest units；
- idempotently apply六個subscriber與一個Internal Group；
- 沒有啟動ML containers或subscriptions。

`make webconsole-start`直接重用既有content-addressed artifact，沒有重新執行Node/Yarn/Go
build。Status為：

```text
enabled=true state=active http=ready endpoint=http://192.168.56.10:5000
```

## HTTP 與 API evidence

從 Host 對 management host-only address驗證：

- `GET /`：HTTP 200，包含frontend root；
- `/assets/index-B9Y_elYo.css`：HTTP 200；
- `/assets/index-DUdcO8NW.js`：HTTP 200；
- 未帶token的`GET /api/subscriber`：HTTP 401；
- `POST /api/login`使用`admin/free5gc`：成功取得access token；
- 帶`Token` header的`GET /api/subscriber`：回傳恰好六筆；
- 六筆`ueId`為`imsi-466920000000001`至`imsi-466920000000006`；
- 六筆`plmnID`均為`46692`。

這證明frontend檔案可取得、後端authentication與MongoDB subscriber read path可用；本次沒有
使用真實browser執行JavaScript，因此不宣稱完成browser rendering／interaction test。

## Billing network boundary

Core `ss` 顯示billing control listener僅為：

```text
127.0.0.1:2121
```

沒有`0.0.0.0:2121`或`192.168.56.10:2121`listener；Host對
`192.168.56.10:2121`的連線測試失敗，符合只允許Core loopback的設定。本次沒有測試FTP
transfer、passive data session、TLS或certificate。

## Reuse、隔離與終態

獨立執行`webconsole-stop`後再次`webconsole-start`，第二次仍回報相同artifact identity與
revision為`reused`，HTTP立即恢復ready；重新登入與六筆subscriber read再次通過。

隔離檢查確認production Compose project沒有running container，Core Consumer為inactive，
subscriptions未建立。最後依序停止WebConsole與23個Guest units，三台VM graceful poweroff。

終態：

- Core、Path A、Path B：`poweroff`；
- running ML containers：0；
- Infrastructure parent與所有submodule worktrees：clean；
- Host RAM available約28 GiB；
- workspace free約221 GiB；
- WebConsole artifact、toolchain、config與MongoDB subscriber state依lifecycle contract保留。
