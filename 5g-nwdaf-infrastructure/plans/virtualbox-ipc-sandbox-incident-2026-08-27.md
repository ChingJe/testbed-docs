# VirtualBox IPC Sandbox Incident Report And Remediation Plan

日期：2026-08-27

狀態：Open；runtime containment、Codex execution guard、repository guard、stable policy guard 與 cleanup 尚未完成

## 1. 摘要

本次事件的直接 root cause 不是 `vagrant up`，而是在 Codex sandbox 內執行看似唯讀的：

```bash
VBoxManage list runningvms
```

當時 sandbox 將 host `/tmp` 以可寫方式帶入，但使用隔離的 `/dev`、PID／network namespace，因而看得到
VirtualBox 的 host IPC pathname，卻看不到 `/dev/vboxdrv`，也無法正常使用 host VirtualBox control plane。
`VBoxManage` 在 IPC 連線失敗後移除了 host 的 VirtualBox IPC lock／socket pathname。原本的 `VBoxHeadless`
processes 沒有因此停止，但 VirtualBox／Vagrant control plane 無法再辨識它們，形成相對於管理面的 orphan runtime。

後續在 host context 執行的 `make vm-up`／`vagrant up` 看到三台 VM 都是 `poweroff`，因而再啟動同一組 declared
VMs，將既有異常放大為 duplicate runtime。第二個動作需要另外防護，但它不是造成 control plane 異常的第一原因。

本事件說明 `AGENTS.md` 或 development policy 的文字規範不足以單獨保護 runtime。主要 remediation 必須在
Codex shell command 啟動前阻止 `VBoxManage`／`vagrant` 進入 sandbox；repository lifecycle guard 與
duplicate-runtime preflight 是第二層；sandbox／OS 不共享 host VirtualBox IPC 才是最強的 enforcement boundary。

## 2. 範圍與文件位置

本文件記錄：

- incident timeline 與直接 evidence；
- root cause、contributing conditions 與影響；
- 目前尚未完成的 containment／cleanup；
- Codex、repository、sandbox／OS 與 stable policy remediation layers；
- implementation、verification 與 approval gates。

事件在 Phase 2 real three-VM lifecycle verification 期間發現，會阻塞任何新的 VM lifecycle evidence，但不是
static Flat／HFL topology contract 的一部分。因 remediation 尚未完成，本文件留在 `plans/`，不能移入
`records/` 或作為 Phase 2 acceptance evidence。

受影響邊界：

- runtime host：VirtualBox、Vagrant、三台 declared VMs 與 `/tmp/.vbox-<user>-ipc`；
- agent runtime：Codex Linux sandbox、shell／unified exec、approval flow；
- source repository：`5G_NWDAF_Infrastructure` 的 VM lifecycle、preflight 與 repository tests；
- documentation repository：`testbed-docs`。

本文件本身不授權停止／刪除 VM、重建 control plane、清除 state、修改 Codex user config，或修改
Infrastructure source。

## 3. 已確認時間線

所有時間使用 Asia/Shanghai（UTC+8）。

1. 2026-08-26 21:24–21:25 左右，第一組 declared VM processes 已由正常 host VirtualBox control plane 啟動。
2. 2026-08-27 02:23:15.404，Codex session 發出 sandboxed `VBoxManage list runningvms`。
3. 02:23:15.548，命令以 exit status 1 結束，直接輸出 `/dev/vboxdrv` 不存在與
   `NS_ERROR_SOCKET_FAIL`。事後 filesystem inode 時序將原 IPC lock pathname 的 unlink 定位於同一 command
   window，約 02:23:15.493。
4. 第一組 VM processes 持續存在，但原 VirtualBox control plane 失去可達 IPC pathname；後續
   `VBoxManage`／Vagrant state 不再能代表實際 process inventory。
5. 12:09 左右，一個經 host approval 執行的 Phase 2 command 最後呼叫 `make vm-up`。Vagrant 將 `core`、
   `path-a`、`path-b` 判定為 `poweroff` 並重新 boot declared VM set，形成 duplicate runtime。
6. 13:17:50，調查期間再次從 Codex sandbox 執行相同的 `VBoxManage list runningvms`，得到相同
   `/dev/vboxdrv`／`NS_ERROR_SOCKET_FAIL`，並重現第二個 control plane 的 IPC pathname 異常。
7. 最後一次 host-level 調查 evidence 顯示兩組 VM processes 仍存在；沒有執行 VM termination。服務層已停止，
   但 VM／VirtualBox cleanup 尚未取得明確批准，因此仍視為 open runtime incident。

第 7 點是易變 runtime observation。任何 cleanup 前必須從 host namespace 重新取得 fresh process inventory，
不能以本文件取代當下 evidence。

## 4. Evidence

### 4.1 Codex session evidence

原始 session event 位於：

```text
/home/chingje/.codex/sessions/2026/08/25/
rollout-2026-08-25T15-20-22-01a037ca-a6ca-7ee2-9f75-2135de28c3a7.jsonl
```

其中 ordinal 6033–6036 記錄 02:23:15 的 tool call、command execution 與錯誤輸出。直接錯誤為：

```text
WARNING: The character device /dev/vboxdrv does not exist.
VBoxManage: error: Code NS_ERROR_SOCKET_FAIL ... IPC daemon socket error
```

目前 session event 位於：

```text
/home/chingje/.codex/sessions/2026/08/27/
rollout-2026-08-27T02-30-01-01a03f56-1504-7491-b014-381765e6e9c1.jsonl
```

其中 ordinal 3063 記錄 13:17:50 的 deterministic reproduction；ordinal 1727 記錄 host-context Phase 2
command 進入 `make vm-up` 後，Vagrant 重新 boot 三台 declared VMs。

### 4.2 Sandbox construction evidence

Codex Linux sandbox process arguments 顯示：

- `--bind /tmp /tmp`：host `/tmp` 對 sandbox 可寫且共享；
- `--dev /dev`：sandbox 使用隔離的 `/dev`；
- `--unshare-pid` 與 `--unshare-net`：process 與 network view 並非 host view。

這個組合使 `VBoxManage` 能修改 host `/tmp` 內的 IPC pathname，卻沒有完整 host VirtualBox device／control
context。問題不是單純「sandbox 內無法啟動 VM」，而是失敗路徑仍能改變 host control-plane IPC state。

### 4.3 Upstream mechanism evidence

Oracle VirtualBox manual 說明 VirtualBox applications 透過位於 system temporary directory 的
`.vbox-<username>-ipc` 和 background service 溝通，並將移除該 IPC directory 列為 communication problem 的
處理方式。這支持「client failure path 可能清理 IPC pathname」的機制判讀；本次哪一個 invocation 觸發 unlink
則由 local session event 與 inode timing 建立。

參考：[Oracle VM VirtualBox User Manual](https://www.virtualbox.org/manual/ch02.html)

## 5. Root cause 與 contributing conditions

### 5.1 Direct root cause

直接 root cause 是：

> `VBoxManage list runningvms` 在不具 host VirtualBox context 的 Codex sandbox 中執行，同時仍具有 host
> VirtualBox IPC pathname 的寫入能力。

`list runningvms` 雖然在 operator 語意上是 read-only query，VirtualBox client 啟動與 IPC recovery path 並非
side-effect-free。因此不能再把任何 `VBoxManage list`、`showvminfo` 或類似查詢預設為 sandbox-safe。

### 5.2 為何 read-only query 仍造成刪除

`list runningvms` 的 read-only 只表示 operator 沒有要求修改 VM inventory；它不表示 `VBoxManage` process 從
啟動到結束都不會寫入 filesystem 或改變 control-plane state。任何 `VBoxManage` subcommand 在讀取 inventory
前，都必須先初始化 VirtualBox client，並透過 `.vbox-<username>-ipc` 連接 `VBoxSVC`；這個共用的 bootstrap、
connection 與 recovery path 可能建立、檢查或清理 IPC artifacts，和最後執行的是 `list` 或 mutation command
是兩件事。

本次 sandbox 形成不對稱能力：它能看見並寫入 host `/tmp` 內真正的 socket／lock pathname，卻缺少
`/dev/vboxdrv` 與完整 host namespace／VirtualBox context。連線因此以 `NS_ERROR_SOCKET_FAIL` 失敗，而同一
command window 內的 IPC pathname 被 cleanup／recovery path unlink。可用下圖理解：

```text
VBoxManage／Vagrant
        │
        ▼
host .vbox-<user>-ipc pathname  ← sandbox client 看得到、也能 unlink
        │
        ▼
VBoxSVC control plane
        │
        ▼
VBoxHeadless VM processes       ← 沒有因 pathname unlink 而停止
```

因此，被刪除的不是 VM process、VM disk、VM memory 或「running」狀態資料本身，而是新 client 用來找到既有
VirtualBox control plane 的 filesystem rendezvous point，亦即觀測兼控制的聯絡入口。Unix socket pathname 被
unlink 不會自動終止已存在的 processes 或已開啟的 resources，但後續 client 無法再以原 pathname 連入；此時
provider-reported `poweroff` 只能表示新 control plane 看不到舊 processes，不能證明 VM 已停止。

Local evidence 已確認 unlink 發生在該 invocation window，且第二次相同 sandbox query 重現 IPC 異常；目前尚未
以 source-level trace 定位是 `VBoxManage` main process 或它啟動的 helper 執行 unlink，也未定位 VirtualBox
內部觸發 cleanup 的 exact branch。因此本報告只將「共同 IPC initialization／recovery path 具有實際副作用」
列為 confirmed mechanism boundary，不對未直接觀察的內部函式作更細斷言。

### 5.3 Contributing conditions

- sandbox 共享可寫 host `/tmp`，但隔離 `/dev` 與 namespace；
- Codex 當時沒有針對 `VBoxManage`／`vagrant` 的 `PreToolUse` block；
- `~/.codex/rules/default.rules` 沒有匹配這兩個 commands；
- `AGENTS.md` 與 development policy 沒有 executable enforcement；
- repository `preflight.sh` 本身會直接呼叫 `VBoxManage`，只能在呼叫失敗後報錯；
- Makefile、host scripts 與 tests 存在多個 direct Vagrant／VirtualBox call paths；
- `vm-up` 前沒有先以 host OS process inventory 偵測「process 存在但 provider state 為 poweroff」；
- control-plane state 被誤當成 actual runtime process state。

### 5.4 Secondary amplifier

`make vm-up`／`vagrant up` 是 secondary amplifier：它沒有建立第一個 IPC 異常，但沒有發現 control plane 和
actual process inventory 已分裂，因而再次啟動 declared VM set。防止 duplicate startup 是必要的 containment，
但只修正 `vm-up` 不能防止第一個 `VBoxManage` query 再次破壞 IPC。

## 6. 影響與風險

- VirtualBox／Vagrant state 無法可靠代表 actual running VM processes；
- 同一 declared identity 可能出現多組 `VBoxHeadless` processes；
- 同 UUID、MAC、IP、port forwarding 或 disk ownership 的 concurrent runtime 可能造成網路衝突、state ambiguity
  或 storage corruption risk；目前不宣稱已確認發生 disk corruption；
- Vagrant lifecycle commands 可能作用在錯誤 control plane 或漏掉 orphan processes；
- Phase 2 的 three-VM capacity、activation、lifecycle 與 cleanup evidence 全部失效或保持未驗證；
- 在未重建 exact host process／VM／disk inventory 前執行 halt、destroy、reset 或再次 up 都不安全。

## 7. 防護目標

1. `VBoxManage` 或 `vagrant` 在 sandbox 中不得啟動 process，包括 read-only subcommands。
2. Codex 必須改用 explicit elevated／host-context request，並由使用者批准後才可執行。
3. 直接命令、absolute path、常見 shell wrapper 與 repository lifecycle path 都必須涵蓋。
4. 即使 Codex policy 漏判，repository 正常入口仍應在第一次 VirtualBox call 前 fail closed。
5. `vm-up` 前必須以 host OS process inventory 偵測 duplicate、orphan 或 provider／process mismatch。
6. 驗證不得為了測 guard 而再次在 sandbox 執行真實 `VBoxManage`。
7. Cleanup 必須先建立 fresh exact inventory，再由使用者批准 exact targets。
8. `AGENTS.md` 與 development policy 必須保存穩定操作規則，但不能被當成 executable enforcement 的替代品。

## 8. Proposed remediation

### 8.1 Layer 1：Codex command-start guard

使用 user-level Codex `PreToolUse` hook 攔截 shell／unified exec：

- 解析 `tool_input.command`，辨識直接命令、absolute binary path、compound shell 與已知 wrappers；
- command 若會在 sandbox 直接或間接執行 `vagrant`、`VBoxManage`，在 process 啟動前回傳
  `permissionDecision: "deny"`；
- rejection reason 明確要求 agent 以 sandbox 外 execution request 重試；
- hook 對 simple command、absolute path、`bash -lc`／compound command 與已知 repository wrappers 建立 fixture
  tests；無法安全解析但含 sensitive command token 時 fail closed。

OpenAI Docs 確認 `PreToolUse` 可攔截 shell 與 unified exec，並能在 tool process 執行前 deny；同一文件也指出
specialized tool paths 可能例外，因此 hooks 是針對本次 Codex shell path 的 executable guardrail，不宣稱等同
完整 OS security boundary。

目前設計尚有一個 implementation gate：OpenAI Docs 保證 `PreToolUse` 收到 tool-specific `tool_input`，但文件沒有
保證所有 CLI／tool path 都以相同欄位暴露 per-call sandbox／escalation intent。實作前必須先用 synthetic capture
fixture 證明目前版本能可靠區分 sandbox request 與 approved host-context request：

- 若存在穩定且不可由 command text 偽造的 execution-context signal，hook 才能實作「sandbox deny、approved
  host context allow」；
- 若不存在可靠 signal，不得猜測或只搜尋 command 內的 `require_escalated` 字串。Hook 應 fail closed，拒絕所有
  matching real provider calls，再由另一個可明確辨識、經批准的 host-only wrapper／tool path 執行；
- specialized tool path 是否經過 hook 也必須納入 coverage，不能以 shell path 通過推論所有入口都受保護。

參考：[OpenAI Docs — Hooks](https://learn.chatgpt.com/docs/hooks)

### 8.2 Layer 2：Codex sandbox-outside approval rule

在 `~/.codex/rules/default.rules` 對至少下列 executable forms 設 `decision = "prompt"`：

- `vagrant`、`/usr/bin/vagrant`；
- `VBoxManage`、`/usr/bin/VBoxManage`。

Rules 負責控制 sandbox 外 execution：`prompt` 要求每次 matching invocation 先取得 approval。它不能單獨保證
agent 不在 sandbox 內嘗試，所以必須和第 8.1 節 hook 一起使用。Shell wrappers 只有在 Codex 能安全拆解時才會
逐 command 套用 rule；複雜 shell expression 需由 hook 或更保守的 wrapper policy 補足。

OpenAI Docs 目前將 Rules 標為 experimental；實作必須記錄並驗證實際 Codex CLI version，config／CLI 更新後
重跑 `codex execpolicy check` fixtures，不把一次通過視為永久相容保證。

參考：[OpenAI Docs — Rules](https://learn.chatgpt.com/docs/agent-configuration/rules)

目標 flow：

```text
sandboxed VBoxManage／vagrant request
→ PreToolUse deny before process start
→ agent selects a verified host-context request／host-only entrypoint
→ prefix rule prompts user
→ approved host-context execution
```

### 8.3 Layer 3：Repository fail-closed guard

延伸既有 `5G_NWDAF_Infrastructure` host lifecycle，不新增平行 VM workflow：

- 在共用 host library 增加 VirtualBox host-context guard；
- 在任何 Vagrant／VBoxManage process 啟動前確認 `/dev/vboxdrv` 為可用 character device；
- Makefile、preflight、lifecycle、upload 與 tests 全部走同一 guard／wrapper；
- repository test 禁止新增未受保護的 direct call；
- 只需要 config syntax 的 tests mock／isolate provider，不在 Codex sandbox 呼叫真實 `vagrant validate`。

這一層只能保護 repository-owned flow；直接執行 `/usr/bin/VBoxManage` 仍由 Codex hook／OS boundary 負責。

### 8.4 Layer 4：Host duplicate／orphan preflight

在 host-context `vm-up` 前，先使用 OS `ps`／`pgrep` inventory，不以 `VBoxManage` 作第一個觀測來源：

- 解析實際 `VBoxHeadless --startvm <UUID>`；
- 每個 expected UUID 必須是 exact allowed count；
- 同 UUID 超過一個時拒絕；
- provider 顯示 `poweroff` 但 process 存在時拒絕；
- unknown／unparseable inventory 時 fail closed；
- 只有 host-context guard 與 process inventory 通過後，才允許呼叫 Vagrant／VBoxManage。

### 8.5 Layer 5：Sandbox／OS IPC isolation

最強防護是 sandbox 根本不能修改 host `.vbox-<user>-ipc`：

- sandbox 使用 private temporary directory；或
- host VirtualBox IPC path 不帶入 sandbox、或以不可寫方式呈現；或
- 由 managed permission profile／OS MAC policy 阻止 sandbox process unlink 該 pathname。

OpenAI Docs 將 sandbox 與 approvals 定義為兩個共同作用的控制面：sandbox 提供 technical boundary，approval
決定何時跨越。依目前本機 Codex CLI／user config 查核，尚未確認有 user-level per-path deny 能只封鎖
`/tmp/.vbox-<user>-ipc`；這一層保持 platform／administrator hardening investigation，不能用未驗證環境變數或
`chattr` workaround 取代。

參考：

- [OpenAI Docs — Sandbox](https://learn.chatgpt.com/docs/sandboxing)
- [OpenAI Docs — Agent approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security)

### 8.6 Layer 6：`AGENTS.md` 與 development policy 軟防護

文件規範不能阻止 process 啟動，但應保存跨 session、跨 agent 仍有效的操作與 review contract，避免 executable
guard 被繞過、移除或錯誤解讀。Incident-specific 時間線、runtime count、path inode 與 cleanup progress 繼續只放
在本報告，不複製到穩定文件。

Workspace root `AGENTS.md` 應加入穩定的 agent execution boundary：

- 所有 real `vagrant`／`VBoxManage` commands，包括 `list`、`status`、`validate` 等 read-only-looking
  subcommands，都不得在 Codex sandbox 執行；
- direct command、absolute path、Makefile、shell wrapper、test 或其他間接入口一律適用；
- agent 必須使用 verified sandbox-outside approval flow；guard verification 只能用 synthetic fixtures，不得以
  real VirtualBox invocation 測試；
- 此規則是行為要求，不宣稱取代 hook、repository guard 或 OS boundary。

`testbed-docs/5g-nwdaf-infrastructure/development_policy.md` 應加入穩定的 provider lifecycle gate：

- Infrastructure production code／tests 不得留下未受共用 host-context guard 保護的 provider direct calls；
- real provider call 前必須驗證 host context 與 `/dev/vboxdrv`，失敗時在第一個 VirtualBox process 前 fail closed；
- `vm-up` 前先取得 host OS process inventory，再和 provider state 比對；duplicate、orphan、parse failure 或
  provider／process mismatch 一律拒絕；
- provider-reported `poweroff` 不能單獨證明 VM 已停止；runtime acceptance 必須同時核對 OS process inventory、
  provider state 與 active config identity；
- real provider verification 只允許在 approved host context 執行，sandbox tests 必須 mock／isolate provider。

Root `AGENTS.md` 位於目前各 git repositories 之外，不能和 `testbed-docs` commit 混在一起；policy 與本報告則按
`testbed-docs` repository 的 user-review／commit gate 處理。

## 9. Remediation 與 cleanup 順序

1. 在 guard 未生效前，禁止從 Codex sandbox 呼叫任何 real `VBoxManage`／Vagrant command。
2. 經獨立 approval 修改 Codex user-layer hook 與 rules，先驗證 execution-context discriminator，再使用 synthetic
   JSON／`codex execpolicy check` 驗證，不碰 real VirtualBox；若無可靠 discriminator，改用 fail-closed hook＋
   verified host-only entrypoint。
3. 經文件 review 分別將第 8.6 節的穩定規則加入 root `AGENTS.md` 與 testbed development policy；不得複製
   incident runtime details 或把軟規範標成 hard enforcement。
4. 經 Phase 2 plan／implementation approval，在既有 Infrastructure lifecycle 加入 host-context guard、direct-call
   regression 與 duplicate-runtime preflight。
5. 從 host namespace 只讀取得 fresh `VBoxHeadless`、`VBoxSVC`、`VBoxXPCOMIPCD`、UUID、disk、network 與 Vagrant
   metadata inventory；此步不得先呼叫 `VBoxManage`。
6. 提出 exact cleanup targets、保留內容、順序、rollback／recovery 與預期 residual state，取得使用者批准。
7. 依批准範圍關閉 duplicate／orphan processes、重建單一 control plane，核對 VM disk／network identity。
8. 只從 approved host context 執行最小 VirtualBox／Vagrant status check，確認 control plane 和 OS process
   inventory 一致。
9. 重新建立 Phase 2 capacity 與 real three-VM lifecycle evidence；事故前或異常期間 evidence 不得沿用。

## 10. Verification matrix

| Layer | Verification | Acceptance |
| --- | --- | --- |
| Hook context | synthetic capture sandbox／escalated fixtures 與 supported tool paths | 有不可由 command text 偽造的 reliable discriminator；否則明確採 deny-all＋host-only entrypoint |
| Hook blocking | 對 sandboxed direct／absolute／simple wrapped commands 注入 synthetic tool input | process 未啟動即 deny；parse ambiguity fail closed；specialized paths 不被假定涵蓋 |
| Exec policy | `codex execpolicy check` 驗證 basename／absolute path fixtures | 全部得到 `prompt`，safe non-matches 不受影響 |
| Approval flow | 使用無 VirtualBox side effect 的 controlled fixture 模擬 elevated request | 未批准不執行；批准後只走 outside-sandbox path |
| Soft policy | review root `AGENTS.md` 與 development policy 的 stable clauses | 包含第 8.6 節 invariants，無 incident-specific runtime details，且不宣稱取代 executable guard |
| Repository context | host-context guard unit／shell tests | `/dev/vboxdrv` 缺少時，在第一個 VirtualBox process 前 fail closed |
| Direct-call coverage | structural search／test | production scripts 與 tests 無未受保護 direct calls |
| Runtime preflight | synthetic UUID／process inventories | duplicate、orphan、provider mismatch、parse failure 全部拒絕 |
| Cleanup | host fresh inventory、exact approved targets、post-cleanup process check | 只剩 expected control plane／VM set，無 unapproved state removal |
| Phase 2 | 重新執行三 VM lifecycle acceptance | provider state、OS process inventory、active config identity 一致 |

不得把「hook script 存在」、「rule syntax 通過」、「soft policy 已寫入」或「repository tests 通過」單獨視為
incident closed。只有 execution guards、stable policy、host inventory、cleanup 與 post-cleanup identity evidence
全部完成並通過 review，才能把本文件整理為 verified record。

## 11. Approval gates 與未決事項

下列工作分別需要明確批准，不能由本報告自動授權：

1. 修改 workspace 外的 `~/.codex/hooks.json`、hook implementation 與 `~/.codex/rules/default.rules`；
2. 修改 workspace root `AGENTS.md` 與 testbed development policy；兩者需按各自 filesystem／repository boundary
   review，不能假設同一 commit；
3. 將 repository guard／preflight 納入 Phase 2 source diff；
4. 在 host context 執行新的 VirtualBox／Vagrant commands；
5. 終止任何 VM／VirtualBox process 或重建 IPC／control plane；
6. 若需 managed sandbox／OS policy change，確認 administrator-owned mechanism。

目前建議：先 review 並加入穩定軟防護，同時完成 Codex hook context gate＋rules；文件本身不視為 protection
ready。接著完成 repository guard／runtime preflight；上述 executable layers 都通過 synthetic verification 後，
才盤點並提出 exact runtime cleanup proposal。OS-level IPC isolation 持續調查，但不阻塞先建立前述 guards。

## 12. Phase 2 關係與退出條件

本 incident 是 Phase 2 real-environment verification blocker。Guard 與 cleanup 完成前：

- 可以執行不會間接啟動 Vagrant／VirtualBox 的 source inspection、render 與 pure tests；
- 不得執行 real VM lifecycle 或把 provider-reported state 當成 actual runtime evidence；
- 不得宣稱 Slice 1 capacity evidence、Slice 7 validation gate 或 Phase 2 three-VM acceptance 已完成。

Incident closure 需要：

1. Codex sandbox command-start guard 已生效並通過 context／command／tool-path fixtures；
2. sandbox-outside approval rules 已生效；
3. root `AGENTS.md` 與 testbed development policy 已保存穩定軟防護，且未被當成 hard enforcement；
4. repository direct paths 與 duplicate-runtime preflight 已 fail closed；
5. exact approved cleanup 完成；
6. 單一 VirtualBox control plane、provider state 與 actual processes exact match；
7. post-remediation review 完成，結果另整理至 `records/`。
