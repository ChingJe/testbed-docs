# Guest Services 與 UE Registration Smoke（2026-08-09）

## 範圍

本輪在已完成 provisioning 的 Core、Path A、Path B 上驗證：

- complete config-set stage／activation；
- topology-derived process aliases；
- 5GC、ADRF、三個 NWDAF、兩個 UPF、兩個 gNB 與六個 UE 的 systemd lifecycle；
- 六個 subscriber 與單一 Internal Group 的可重現 provisioning；
- registration、PDU Session 與 TAI-specific UE pool。

Host ML containers、consumer subscriptions、N6 traffic 與 PseudoDriver replay 不在本輪
完成宣告內。

## 整合期間修正

第一次 activation 在 Core 拒絕 config hash。Host 與 Guest 原本把 `sha256sum` 輸出的
absolute filename 一起納入第二層 hash，因此相同內容位於不同 root 時必然不同。兩端改為先
進入 config-set root，再以排序後的相對路徑與檔案內容產生 identity。兩個不同暫存 root 的
相同 config tree 都得到 `ca21f404...`，後續含 UE schema 修正的有效 config identity 為
`a73ef32bb621b3a20efa836f12183a95bdf0e7bd34cfb8565f8884626f5a99c0`。

第二次啟動到 SMF 時，binary 仍以預設 `./config/uerouting.yaml` 查找 routing config。
Guest dispatcher 改為同時傳入 active `smfcfg.yaml` 與 active `uerouting.yaml` 的絕對路徑。
同時把啟動失敗清理從未跨 function 生效的 `ERR` trap 改為 success path 才解除的 `EXIT`
rollback；下一次 UE schema failure 實際證明所有已啟動 units 會被 reverse-order stop。

第三次啟動時，pinned UERANSIM `2a3ef81` 拒絕缺少 `uacAic` 的 UE config。六份 UE baseline
補上 canonical `uacAic`／`uacAcc`，checker 也新增同一 schema gate。Default 與 disposable
rendered config set 都通過。

process 全部 active 後，六個 UE 因 MongoDB 尚無 subscription data，在 AUSF authentication
收到 404。先以 `nwdaf-resources@d2634b84` 的既有 scoped fixture 證明 48 份 subscriber
documents 與一份 Internal Group 能完成 registration，再只把必要 fixture 與功能收進新
repository。正式工具使用 Core 已安裝的 `mongosh`，提供 validate／plan／apply／show／clear，
不依賴整個 `nwdaf-resources` 或 Host PyMTLF virtualenv；`clear` 不會 drop database／collection。
兩份 committed fixture 的 combined identity 為
`d30803f9c5904ae86bb222484170089cc4cf60ee3fe3f29e43c6487918113167`。

## 最終結果

完整執行一次 `services-stop` 後重新 `services-start`，內建 provisioning 在 UE 啟動前
idempotently apply，未再人工 restart UE。結果：

- 23 個 guest units 全部維持 `active`；
- UE1–UE3 都完成 Initial Registration 與 PSI 1 PDU Session，取得
  `10.60.0.1`–`10.60.0.3/16`；
- UE4–UE6 都完成 Initial Registration 與 PSI 1 PDU Session，取得
  `10.61.0.1`–`10.61.0.3/16`；
- Core AMF log 對六個 SUPI 都出現 Registration Complete；
- `subscriber-data-show` 在八個 scoped collections 各找到六筆資料，group collection
  找到 `00000001-466-92-01` 與完整六人 membership；
- 三台 VM 與 guest services 保持 running，Host ML containers 與 subscriptions 未啟動。

最終 Host 約有 24 GiB `MemAvailable`、166 GiB workspace free；2 GiB swap 仍幾乎全滿，
符合目前 warn-only policy，但不適合長時間 unattended run。

## 尚未解除的邊界

- SMF 在 PDU Session 期間會記錄找不到 BSF／CHF；目前 profile 明確延後這兩個 NF，且六個
  PDU Session 仍成功，但後續應確認 warning/fallback 符合預期設定。
- 尚未驗證 UE 經 N6 的實際封包路徑與 NAT。
- 尚未驗證兩個 UPF PseudoDriver dataset replay、Event Exposure、NWDAF collection。
- 尚未在這次 full guest stack 上啟動五個 production ML containers 或 consumer。
