# VirtualBox IPC Sandbox Incident Remediation Record

日期：2026-08-27

狀態：Completed / Verified Record；縮小版 Codex direct-provider guard、stable policy guard、repository host-context
guard 與 `vm-up` runtime process-inventory preflight 已實作；事故 runtime 已依批准範圍完成 exact cleanup 與 clean
rebuild；單一 control plane、provider state、OS process inventory 與 provisioning artifacts 已重新一致；mandatory
initial review 與 user review 已通過，Infrastructure remediation 已由 commit `531e335` 保存。Phase 2 selected-config
lifecycle acceptance 不屬於本 incident closure，仍保持未驗證

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

本事件說明 `AGENTS.md` 或 development policy 的文字規範不足以單獨保護 runtime。經 usability review 後採用的
折衷是：Codex hook 只攔截直接或 shell-wrapped 的 `VBoxManage`／`vagrant` invocation，不攔 Make targets 或
repository scripts；既有 Make／host lifecycle 則必須在其第一個 provider child process 前通過共同 host-context
guard。`vm-up` duplicate-runtime preflight 已完成 synthetic implementation checkpoint，且現存事故的 real-host
provider-before-query detection 已通過。Cleanup 期間另發現 lifecycle lock descriptor 會被 long-lived provider
helper 繼承；wrapper 已在 exec provider child 時關閉該 descriptor，並通過 synthetic 與 real-host evidence。
Sandbox／OS 不共享 host VirtualBox IPC 仍是最強的 enforcement boundary。

## 2. 範圍與文件位置

本文件記錄：

- incident timeline 與直接 evidence；
- root cause、contributing conditions 與影響；
- 已完成的 containment／cleanup、remediation、review 與 source identity；
- Codex、repository、sandbox／OS 與 stable policy remediation layers；
- implementation、verification 與 approval gates。

事件在 Phase 2 real three-VM lifecycle verification 期間發現，曾阻塞任何新的 VM lifecycle evidence，但不是
static Flat／HFL topology contract 的一部分。Incident remediation、cleanup、review 與獨立 source commit 已完成，
因此本文件保存於 `records/`；它只證明 incident scope，不作為 Phase 2 selected-config lifecycle acceptance evidence。

受影響邊界：

- runtime host：VirtualBox、Vagrant、三台 declared VMs 與 `/tmp/.vbox-<user>-ipc`；
- agent runtime：Codex Linux sandbox、shell／unified exec、approval flow；
- source repository：`5G_NWDAF_Infrastructure` 的 VM lifecycle、preflight 與 repository tests；
- documentation repository：`testbed-docs`。

本文件本身不授權停止／刪除 VM、重建 control plane、清除 state、修改 Codex user config，或修改
Infrastructure source。

### 2.1 Artifact identity、environment 與 verification boundary

- Infrastructure repository：branch `feat/r18-hierarchical-federated-learning`，incident commit `531e335`，parent
  `072748b`；commit 只包含 approved provider guard／preflight hunks，未包含其餘 Phase 2 working-tree changes。
- Documentation repository：本 record 以 `85ce008` 為先前 stable-policy baseline，再納入 `/dev/vboxdrv` device
  namespace 語意修正、Phase 2 evidence synchronization 與本 verified record。
- Runtime environment：2026-08-27 Asia/Shanghai，Linux host、VirtualBox／Vagrant、三台 declared VMs；exact UUID、
  process 與 guest artifact evidence 見第 4.4 節。
- Commit 前以由 Git index 匯出的 isolated `HEAD + incident hunks` snapshot 執行 shell syntax、
  `tests/provider-runtime-preflight.sh`、`tests/execution-policy.py` 與完整 `make test`，全部通過；tests 使用 synthetic
  provider fixtures，沒有啟動 real provider process。Temporary snapshot 因不含 `.git` 且 user Go cache 不可寫，
  明確設定 temporary `GOCACHE` 與 `GOFLAGS=-buildvcs=false`，不改變 repository source contract。
- Phase 2 static selected-config activation、service lifecycle、reset／seed restoration 與 managed sandbox／OS IPC
  isolation 未納入本 record；前者回到 active Phase 2 plan，後者維持 optional hardening。

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
7. Cleanup 前由 host namespace 重取 exact PID／parent／UUID inventory，確認仍是兩組 control planes 與每個
   declared UUID exactly two 個 `VBoxHeadless` processes；使用者隨後明確批准完整 cleanup／recovery。
8. 六個 exact `VBoxHeadless` targets 停止後，兩組 exact `VBoxSVC`／`VBoxXPCOMIPCD` 亦停止；stale IPC directory
   被移至隔離路徑，沒有以 wildcard 或 provider-reported state 推斷 destructive scope。
9. 舊 VM disk 首次 cold start 只讓 core 進入 provider running，但 SSH 持續 reset，不能建立 provisioning 完整性；
   依使用者確認 VM 內沒有需保留的重要資料，三台 declared VMs 全部 destroy 後 clean rebuild。
10. 新 `core`、`path-a`、`path-b` 全部完成 provisioning。Post-check 顯示單一 `VBoxSVC`／`VBoxXPCOMIPCD`、
    每個新 UUID exactly one 個 `VBoxHeadless`、provider state 全部 running，且沒有 lifecycle lock holder。

第 10 點是本次 cleanup／rebuild 後的 point-in-time evidence；後續 runtime acceptance 仍須在每次 destructive 或
lifecycle 操作前重取 host inventory，不能以本文件取代當下 evidence。

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

### 4.4 Cleanup／rebuild evidence

Cleanup 與 recovery 全部在 approved host context 執行，且 destructive steps 前以 fresh OS process inventory
exact revalidate targets：

- exact 停止六個 duplicated `VBoxHeadless` processes 與兩組 service pairs 後，host 不再有 VirtualBox process；
- stale `/tmp/.vbox-chingje-ipc` 移至 `/tmp/.vbox-chingje-ipc.quarantine-20260827-incident`，保留原 inode／socket／
  lock artifacts 作 provenance；
- 保守建立的舊 VM directory copy 位於
  `/home/chingje/VirtualBox incident backups/2026-08-27-duplicate-runtime`，`diff -qr` 一致。使用者後續確認舊 VM
  沒有重要資料、rebuild 不需要依賴此 copy；因刪除備份是另一個 destructive action，目前仍保留；
- 舊 disk cold-start 的 core 雖被 provider 顯示 running，SSH 在五分鐘觀察窗內持續 reset，因此沒有把該 state
  當成健康證據；依批准範圍 destroy 三台 exact declared VMs，確認 `.vagrant/machines` empty 後重新建立；
- 新 UUID 為 core `b1720075-a76a-4e50-809e-42654add78ec`、path-a
  `28aec4a7-171d-4109-ace3-5206b7e7cf4f`、path-b `2fc0ff86-5822-4a49-8d1e-8bede65db7c2`；
- 最終 provider state 為三台 running；host process inventory 為單一 `VBoxXPCOMIPCD`、單一 `VBoxSVC`，以及每個
  新 UUID exactly one 個 `VBoxHeadless`；沒有 `vagrant-up.lock` holder；
- guest artifact checks 全部通過：三台 role／hostname、Go 1.26.2、provisioning manifest 與 runtime tools 正常；
  core 具有全部十個 NF binaries 與 MongoDB 8.0.28；兩台 path 具有 UPF／NWDAF／UERANSIM binaries 與
  gtp5g 0.9.16。

這組 evidence 關閉事故 cleanup、single-control-plane recovery 與 base provisioning recovery；它沒有執行 static
Flat／HFL config activation、service lifecycle、reset 或 seed restoration，因此不能單獨作為 Phase 2 lifecycle
acceptance。

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
3. 直接命令、absolute path 與會直接啟動 provider 的 shell wrapper 由 hook 涵蓋；repository lifecycle path 由
   內部共同 guard 涵蓋，不要求 hook 維護 Make target／script allowlist 或 denylist。
4. Make 與 repository scripts 在 sandbox 內仍可執行非 provider 部分；若流程會進入 provider，必須在第一個
   VirtualBox／Vagrant process 前 fail closed。
5. `vm-up` 前必須以 host OS process inventory 偵測 duplicate、orphan 或 provider／process mismatch。
6. 驗證不得為了測 guard 而再次在 sandbox 執行真實 `VBoxManage`。
7. Cleanup 必須先建立 fresh exact inventory，再由使用者批准 exact targets。
8. `AGENTS.md` 與 development policy 必須保存穩定操作規則，但不能被當成 executable enforcement 的替代品。

## 8. Remediation layers

### 8.1 Layer 1：Codex command-start guard

使用 user-level Codex `PreToolUse` hook 攔截 shell／unified exec：

- 解析 `tool_input.command`，辨識直接命令、absolute binary path、compound shell 與直接執行 provider 的 shell
  payload；
- command 若會由目前 shell tool 直接執行 `vagrant`、`VBoxManage`，在 process 啟動前回傳
  `permissionDecision: "deny"`；
- rejection reason 明確要求 agent 使用 repository Make target，或改以 sandbox 外 host-only entrypoint 執行；
- hook 對 simple command、absolute path、`bash -lc`／compound direct call 建立 fixture tests；無法安全解析但含
  sensitive command token 時 fail closed；
- Make targets 與 repository scripts 本身由 hook 放行，避免 hook 維護容易 drift 的間接入口清單；其 provider
  child process 必須由第 8.3 節 repository guard 攔截。

OpenAI Docs 確認 `PreToolUse` 可攔截 shell 與 unified exec，並能在 tool process 執行前 deny；同一文件也指出
specialized tool paths 可能例外，因此 hooks 是針對本次 Codex shell path 的 executable guardrail，不宣稱等同
完整 OS security boundary。

OpenAI Docs 保證 `PreToolUse` 收到 tool-specific `tool_input`，但文件沒有保證所有 CLI／tool path 都以相同欄位
暴露 per-call sandbox／escalation intent。本機 synthetic capture 未找到可靠 discriminator，因此不以 command
text、環境變數或推測欄位判斷 context：

- 若存在穩定且不可由 command text 偽造的 execution-context signal，hook 才能實作「sandbox deny、approved
  host context allow」；
- 因目前不存在可靠 signal，Hook 對 matching direct provider calls fail closed，再由可明確辨識、經批准的
  host-only wrapper／tool path 執行；
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

目標 flow 分成兩條：

```text
direct sandboxed VBoxManage／vagrant request
→ PreToolUse deny before process start
→ agent selects repository Make target or verified host-only entrypoint

sandboxed Make／repository script
→ PreToolUse allow
→ repository host-context guard denies before first provider child process

approved host-context Make／host-only entrypoint
→ sandbox-outside approval flow；direct host-only entrypoint additionally matches prompt rule
→ repository／wrapper host-context guard passes
→ provider process starts
```

### 8.2.1 Codex user-level implementation checkpoint（2026-08-27）

本機 `codex-cli 0.147.0` 的官方 hook input schema 沒有提供可可靠區分「sandbox invocation」與「已批准
host-context invocation」的 per-call execution-context field。因此沒有用 command text、環境變數或推測欄位充當
discriminator，而是採用 direct-provider deny＋exact host-only entrypoint，並把 repository-owned indirect path
交給第 8.3 節共同 guard：

- `~/.codex/hooks.json` 註冊 user-level `PreToolUse` guard；
- `~/.codex/hooks/provider_execution_guard.py` 對 shell／unified exec 的直接、absolute、compound 與 shell-wrapped
  provider calls fail closed，但明確放行 Make targets 與 repository scripts；
- `~/.codex/bin/host-provider` 是唯一由 hook 放行的 provider entrypoint，只接受 `vagrant` 或 `VBoxManage` argv，
  並在 exec provider 前驗證 `/dev/vboxdrv` 是 character device；
- `~/.codex/rules/default.rules` 保留既有 rules，另對兩個 provider 的 basename／absolute path 與 exact
  host-only entrypoint 設為 `prompt`；原檔另存同目錄的 pre-provider-guard backup；
- `codex execpolicy check` 只作 rule decision simulation，不執行 provider，因此 hook 僅對這個 exact fixture path
  提供測試例外，沒有放寬一般 `codex`、shell 或 provider commands。

已完成的 synthetic verification：

- 9 個 hook unit tests 通過，涵蓋 direct／absolute provider、compound／shell-wrapped direct calls、Make／repository
  entrypoint allow、safe text inspection、invalid input fail-closed、exact host wrapper，以及 `rg --pre` 這類可啟動
  外部 command 的邊界；
- hook 與 test Python compile、`hooks.json` parse、host wrapper shell syntax 均通過；
- `codex execpolicy check` 對 `vagrant`、`/usr/bin/vagrant`、`VBoxManage`、`/usr/bin/VBoxManage` 與 exact
  host-only entrypoint 均得到 `prompt`，safe `rg` fixture 無 match；
- 在目前 sandbox 呼叫 host-only entrypoint 時，wrapper 因 `/dev/vboxdrv` 不可用而以 status 126 在 provider exec
  前拒絕；此檢查沒有啟動真實 Vagrant／VirtualBox process。

先前 full-wrapper deny 版本已在 fresh session 通過 activation probe：`make -n vm-status` 會在 Make process 啟動前
被拒絕。經使用者確認縮小範圍後，`hooks.json` 的 hook definition／command path 未變，只更新 handler 與 tests；
目前 session 已觀測到新 handler 行為：同一個 `make -n vm-status` 能展開 recipe，而 `make vm-status` 進入
repository guard 後以 status 126 拒絕。兩個 probe 都沒有啟動真實 provider process。這組 evidence 證明目前
shell path 正在使用縮小版 handler，但不把 hook 視為完整 enforcement boundary；任何日後 hook definition 變更仍
必須依 OpenAI Docs 以 `/hooks` review／trust current hash。

這個 user-level guard 是本次 shell／unified exec path 的前置攔截，不是完整 security boundary。它無法證明任意改名
alias 或 specialized tool path 都受保護；它也刻意不解析 Make 的 child-process graph。第 8.3 節 repository
regression guard 與第 8.5 節 OS／sandbox isolation 仍然必要。

### 8.3 Layer 3：Repository fail-closed guard

延伸既有 `5G_NWDAF_Infrastructure` host lifecycle，不新增平行 VM workflow：

- 在共用 host library 增加 VirtualBox host-context guard；
- 在任何 Vagrant／VBoxManage process 啟動前確認 `/dev/vboxdrv` 在 host device namespace 中可見且為 character
  device，避免缺少該 namespace 的 sandbox client 接觸共享 host IPC；
- Makefile、preflight、lifecycle、upload 與 tests 全部走同一 guard／wrapper；
- repository test 禁止新增未受保護的 direct call；
- 只需要 config syntax 的 tests mock／isolate provider，不在 Codex sandbox 呼叫真實 `vagrant validate`。

這一層只能保護 repository-owned flow；直接執行 `/usr/bin/VBoxManage` 仍由 Codex hook／OS boundary 負責。

### 8.3.1 Repository implementation checkpoint（2026-08-27）

Infrastructure commit `531e335` 已保存 host-context guard 的最小整合，沒有新增平行 lifecycle entrypoint：

- `scripts/host/lib.sh` 集中提供 `provider_vagrant`／`provider_vboxmanage`；兩者在執行 client 前先確認
  `/dev/vboxdrv` 在 host device namespace 中可見且是 character device，失敗時回傳 status 126；
- Makefile 的 `vm-up`／`vm-status`／`vm-halt`，以及 host lifecycle、upload、preflight 與 config validation paths
  全部改走共同 wrappers；
- `tests/execution-policy.py` structural regression 會拒絕 Makefile、host scripts 或 repository shell tests 新增未受
  保護的 direct provider command；只有共同 wrapper 內的兩個 final exec lines 是明確例外；
- `tests/repository.sh` 使用 fake provider executable 與 function override 驗證 guard-before-exec，不呼叫真實
  provider；原本三個 real `vagrant validate` checks 已改成 selected-testbed／provider contract 的 structural
  assertions，並以 Vagrant 內附 Ruby 單獨執行 `Vagrantfile` syntax／negative-selection fixtures，避免 sandbox
  repository suite 啟動 provider client。

Focused verification 結果：

- `python3 tests/execution-policy.py` 通過；
- isolated incident commit snapshot 的 shell syntax、provider guard mock、direct-call policy 與完整 `make test` 通過；
  目前較大的 Phase 2 working tree 仍另有 manifest exact-inventory ordering mismatch，不屬於本 commit；
- `make -n vm-status` 正常展開；sandbox 內的 `make vm-status` 在共同 guard 以 status 126 停止，沒有 provider
  process；
- 本 checkpoint 只關閉 host-context guard；後續完成的 OS process-inventory／duplicate-orphan preflight 另記於
  第 8.4.1 節。

後續 host evidence 證明 `/dev/vboxdrv` 的套件預設 contract 是 `root:root 0600`，原本要求 current-user access 的
predicate 會錯誤拒絕正常 host。經使用者確認後已將 predicate 修正為只確認 `/dev/vboxdrv` 在目前 device namespace
中可見且是 character device，並加入 source comment 說明這項檢查是為了在 provider client 接觸共享 host IPC 前
要求 host VirtualBox device namespace。修正後的 no-provider verification 結果：

- shell syntax、`tests/execution-policy.py`、`tests/provider-runtime-preflight.sh` 與兩個 repositories 的
  `git diff --check` 通過；
- repository suite 的 host-context guard、mock wrapper 與 runtime-preflight sections 通過；完整 suite 後續仍停在
  既有 Phase 2 production manifest exact-inventory ordering mismatch，與本次 guard correction 無關；
- sandbox 內直接呼叫 predicate 會拒絕；經批准在 sandbox 外呼叫同一 predicate 會接受 host
  `/dev/vboxdrv`；兩個 probe 都沒有啟動 Vagrant／VirtualBox process。

### 8.4 Layer 4：Host duplicate／orphan preflight

在 host-context `vm-up` 前，先使用 OS `ps`／`pgrep` inventory，不以 `VBoxManage` 作第一個觀測來源：

- 解析實際 `VBoxHeadless --startvm <UUID>`；
- 每個 expected UUID 必須是 exact allowed count；
- 同 UUID 超過一個時拒絕；
- provider 顯示 `poweroff` 但 process 存在時拒絕；
- unknown／unparseable inventory 時 fail closed；
- 只有 host-context guard 與 process inventory 通過後，才允許呼叫 Vagrant／VBoxManage。

#### 8.4.1 Repository implementation checkpoint（2026-08-27）

Infrastructure commit `531e335` 已把 preflight 接入既有 `make vm-up`，沒有新增另一套 VM lifecycle：

- 共用 library 以 current-user `pgrep` 取得 `VBoxHeadless` command lines，再由獨立 internal parser 將每筆
  `--startvm <UUID>` 正規化為 exact PID／UUID record；missing／multiple／invalid UUID、`pgrep` failure 或
  非預期 record 均拒絕；
- preflight 從既有 `.vagrant/machines/<machine>/virtualbox/id` 取得 project-owned UUID，拒絕 duplicate UUID、
  unexpected machine／provider metadata、invalid records 與同 UUID 多 process；missing ID 只在 Vagrant state
  exact 為 `not_created` 時成立；
- 第一份 OS process inventory 通過後才允許執行 guarded `vagrant status --machine-readable`，接著重取第二份 OS
  inventory；查詢期間 process inventory 有任何改變即拒絕；
- Vagrant `running` 必須對應 exactly one expected `VBoxHeadless` process；`poweroff`／`saved`／`aborted` 必須對應
  zero process；metadata、`not_created`、unexpected state 或反向 mismatch 一律拒絕；
- preflight 與最後的 `vagrant up` 位於同一個 repository lifecycle lock 內；preflight 失敗時不會執行 provider up；
- `tests/provider-runtime-preflight.sh` 以 synthetic process、metadata 與 machine-state fixtures 覆蓋 clean、running、
  duplicate、orphan、reverse mismatch、inventory race、parse／query failure 與 provider-before-preflight rejection；
  `tests/execution-policy.py` 另固定 Make `vm-up` 必須使用 preflight wrapper。

Focused verification 已通過 `tests/provider-runtime-preflight.sh`、`python3 tests/execution-policy.py`、shell／Python
syntax、`git diff --check` 與 isolated incident snapshot 的完整 `make test`；沒有執行 real Vagrant／VirtualBox
process。較大的 Phase 2 working tree 仍有 production manifest exact-inventory ordering mismatch，依 active plan
處理。現存 incident 的 real-host detection acceptance 後續已依第 8.4.2 節取得，且沒有用 synthetic checkpoint
取代。

#### 8.4.2 Real-host duplicate detection checkpoint（2026-08-27）

經使用者批准後，從 sandbox 外只讀取得 current-user Host process inventory；整個 checkpoint 沒有呼叫
`vagrant`／`VBoxManage`，也沒有啟動、停止或清理 VM：

- Host inventory 有六個 `VBoxHeadless` processes；既有 `.vagrant` metadata 的 `core`、`path-a`、`path-b`
  三個 VM UUID 各對應 exactly two processes，沒有 unmatched `VBoxHeadless`；
- parser／stage-1 validator 在第一個 provider query 前回報三組 duplicate UUID 並以 status 1 拒絕，證明現存
  事故可以由新 detector 擋下，不需要以 `vagrant up` 試錯；
- 舊 cohort 由 2026-08-26 21:24:17 啟動的 `VBoxSVC` 擁有，三台 VM 於 21:24:24–21:25:44 啟動；新 cohort
  由 2026-08-27 12:02:32 啟動的另一個 `VBoxSVC` 擁有，三台 VM 於 12:02:39–12:07:28 啟動；Host 同時有
  exactly two `VBoxSVC` 與兩個 `VBoxXPCOMIPCD`；
- 兩組 processes 使用相同的三個 project VM UUID；repository Vagrant disk metadata 與三份 current `.vbox`
  definitions 對每個 VM 只列出同一組 primary／config-drive identities，因此 cleanup 前仍須視為 shared
  machine／disk identity risk，不能各自當成獨立 VM；
- `/dev/vboxdrv` 存在且是 character device，套件 udev rule 將它設為 `root:root 0600`；第一次 integrated probe
  因舊 guard 額外要求 current-user access，在任何 process inventory／provider query 前錯誤回傳 status 126。這是
  guard contract defect，不是 host driver access failure；
- 修正後的 guard 只用 character-device visibility 阻止缺少 host device namespace 的 sandbox path。完整
  `make vm-up` sequence 仍會先因現存 duplicate process inventory 而拒絕，不能為了測試而呼叫 provider。

這個 checkpoint 關閉「現存 duplicate 是否能在 provider 前被 detector 看見」；它不關閉 provider-state compare、
cleanup、單一 control-plane recovery 或 post-cleanup acceptance。任何 termination 前仍須
重取 exact PID／parent／UUID inventory 並取得 cleanup target approval。

#### 8.4.3 Lifecycle lock inheritance remediation（2026-08-27）

第一次 cleanup 後以舊 disk cold-start 時，發現 `provider_vagrant_up` 的 subshell 以 FD 9 持有 repository
`vagrant-up.lock`，但 final `vagrant` exec 讓 `VBoxXPCOMIPCD`／`VBoxSVC` 繼承同一 descriptor。Vagrant command
結束後 long-lived helper 仍持有 lock，下一次 `make vm-up` 因而無限等待 `flock`。這不會建立 duplicate VM，
但會讓正常 lifecycle permanent-block，且原 synthetic tests 未檢查 child FD inheritance。

依 test-first remediation：

- `tests/provider-runtime-preflight.sh` 增加 fake provider child fixture，先 deterministic 證明 FD 9 會洩漏；
- `provider_vagrant` 的 final provider invocation 改為 `command vagrant "$@" 9>&-`，只在 exec child 關閉
  lifecycle lock descriptor；lock owner subshell 在 preflight／up command 完成前仍正常持鎖；
- `tests/execution-policy.py` 的唯一 allowed wrapper line 同步 exact update，維持 direct-provider structural guard；
- shell syntax、focused provider runtime test、execution policy 與 `git diff --check` 通過；
- real-host rebuild 後，long-lived `VBoxSVC` 存在時已確認沒有 process 持有 `vagrant-up.lock`，後續三台 targeted
  up／provision／status／vssh commands 均可依序完成。

此修正是既有 lifecycle lock 的 descriptor-boundary correction，不新增 provider entrypoint，也不改變
duplicate／orphan preflight contract。

#### 8.4.4 Approved cleanup and clean-rebuild checkpoint（2026-08-27）

第 4.4 節的 exact cleanup、destroy、rebuild 與 post-check 已完成。原 duplicated runtime、stale control plane 與
舊 Vagrant machine metadata 均不再作為目前 runtime；新 runtime 已達到單一 control plane、provider running
state、OS process UUID 與 base provisioning artifacts 一致。Incident 不再阻塞後續 Phase 2 real-environment
verification。Mandatory initial review 已完成且沒有 incident-scope defect，user review 亦已通過；獨立 source
commit `531e335` 與 verified record 整理亦已完成。

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
- real provider call 前必須驗證 host context，並確認 `/dev/vboxdrv` 在該 device namespace 中可見且是 character
  device；失敗時在第一個 VirtualBox process 前 fail closed；
- `vm-up` 前先取得 host OS process inventory，再和 provider state 比對；duplicate、orphan、parse failure 或
  provider／process mismatch 一律拒絕；
- provider-reported `poweroff` 不能單獨證明 VM 已停止；runtime acceptance 必須同時核對 OS process inventory、
  provider state 與 active config identity；
- real provider verification 只允許在 approved host context 執行，sandbox tests 必須 mock／isolate provider。

Root `AGENTS.md` 位於目前各 git repositories 之外，不能和 `testbed-docs` commit 混在一起；policy 與本報告則按
`testbed-docs` repository 的 user-review／commit gate 處理。

## 9. Remediation 與 cleanup 順序

1. 縮小版 Codex direct-provider guard 已安裝，並以 no-provider coverage probe 確認目前 shell path 使用新 handler；
   仍禁止從 Codex sandbox 呼叫任何 real `VBoxManage`／Vagrant command。
2. Codex user-layer hook、rules 與 host-only entrypoint 已依獨立 approval 安裝；因沒有可靠 discriminator，direct
   provider calls 採 fail-closed hook＋exact host-only entrypoint，Make／repository scripts 則交由內部 guard。
3. 第 8.6 節的穩定規則已分別加入 root `AGENTS.md` 與 testbed development policy；後者已由
   `testbed-docs` commit `85ce008` 保存。兩者不宣稱取代 executable guard。
4. 經 Phase 2 implementation approval，既有 Infrastructure lifecycle 的 host-context guard、direct-call regression
   與 `vm-up` duplicate-runtime preflight 已實作並通過 synthetic verification。
5. Host namespace 的 fresh `VBoxHeadless`、`VBoxSVC`、`VBoxXPCOMIPCD`、UUID 與 Vagrant／machine definition
   inventory 已只讀取得；stage-1 detector 已拒絕三組 duplicate UUID，且沒有先呼叫 `VBoxManage`。Cleanup 前須
   再重取易變 PID evidence。
6. Exact cleanup targets、順序、recovery 與 residual state 已取得使用者批准；舊 VM 內容確認不需保留。
7. Duplicate／orphan processes 已依 exact inventory 關閉；舊 disk cold-start 無法建立健康 evidence 後，三台 VM
   已 destroy 並 clean rebuild。
8. Approved host-context status、OS process inventory 與 guest artifact checks 已確認單一 control plane、三個新
   VM UUID、三台 running 與 base provisioning 完整。
9. Cleanup 期間發現的 lifecycle lock FD inheritance 已以 failing fixture、最小 wrapper fix 與 real-host no-holder
   evidence 關閉。
10. Incident mandatory initial review 與 user review 已完成；approved source 已由 Infrastructure commit `531e335`
    保存，本文件亦已整理為 verified record。下一步回到 Phase 2，修正 manifest ordering mismatch，再重新建立
    selected-config capacity 與 real three-VM lifecycle evidence；事故前或異常期間 evidence 不得沿用。

## 10. Verification matrix

| Layer | Verification | Acceptance |
| --- | --- | --- |
| Hook context | synthetic capture sandbox／escalated fixtures 與 supported tool paths | 有不可由 command text 偽造的 reliable discriminator；否則明確採 deny-all＋host-only entrypoint |
| Hook blocking | 對 sandboxed direct／absolute／simple wrapped commands 注入 synthetic tool input | direct process 未啟動即 deny；Make／repository entrypoints 放行給內部 guard；parse ambiguity fail closed；specialized paths 不被假定涵蓋 |
| Exec policy | `codex execpolicy check` 驗證 basename／absolute path fixtures | 全部得到 `prompt`，safe non-matches 不受影響 |
| Approval flow | 使用無 VirtualBox side effect 的 controlled fixture 模擬 elevated request | 未批准不執行；批准後只走 outside-sandbox path |
| Soft policy | review root `AGENTS.md` 與 development policy 的 stable clauses | 包含第 8.6 節 invariants，無 incident-specific runtime details，且不宣稱取代 executable guard |
| Repository context | host-context guard unit／shell tests | 已通過 mock／controlled checks；`/dev/vboxdrv` 缺少時，在第一個 VirtualBox process 前 fail closed |
| Direct-call coverage | structural search／test | production scripts 與 tests 無未受保護 direct calls |
| Runtime preflight | synthetic UUID／process inventories | duplicate、orphan、provider mismatch、parse failure 全部拒絕 |
| Cleanup | host fresh inventory、exact approved targets、post-cleanup process check | Passed：單一 control plane、三個新 VM UUID 與 base provisioning 一致；未刪除未批准資料 |
| Phase 2 | 重新執行三 VM lifecycle acceptance | Pending：provider／OS base identity 已一致；selected active config、service、reset／seed evidence 尚待重建 |

Mandatory initial review 未發現 incident-scope defect。分類結果如下：

- confirmed current-scope defect：無；lifecycle lock FD inheritance 已依 test-first flow 關閉；
- integration verification gap：Phase 2 selected-config activation、service lifecycle、reset／seed restoration 尚未執行；
- Phase 2 open work：目前 `make test` 在 provider／identity focused sections 通過後，停於 production manifest
  `guestServices` exact-inventory ordering mismatch；此項在事故前已存在，應由 Phase 2 implementation remediation
  處理，不視為 incident guard／cleanup regression；
- optional hardening：managed sandbox／OS IPC isolation 尚未確認可用機制；
- legacy cleanup：舊 VM 備份與 quarantined IPC artifacts 仍保留，刪除須另行明確批准；
- unconfirmed risk：沒有證據顯示 duplicate runtime 已造成 disk corruption；舊 VM 已依批准範圍 clean rebuild，
  不再用舊 disk state 作 runtime evidence。

不得把「hook script 存在」、「rule syntax 通過」、「soft policy 已寫入」或「repository tests 通過」單獨視為
incident closed。本 record 的結論來自 execution guards、stable policy、host inventory、cleanup、post-cleanup
identity evidence、mandatory review、user review 與獨立 source commit 全部完成；各項 evidence 的適用範圍仍以
本文件明列的 boundary 為準。

## 11. Approval gates 與未決事項

下列工作分別需要明確批准，不能由本報告自動授權；目前 gate 狀態如下：

1. Codex user config 修改已獲批准並完成安裝；先前 fresh-session hook trust／activation 已完成，縮小版 handler
   另以 no-provider behavior probe 確認正在使用；未來若 hook definition hash 改變，仍須重新 `/hooks` review；
2. workspace root `AGENTS.md` 與 testbed development policy 修改已獲批准並完成；兩者已按 filesystem／repository
   boundary 分開處理，沒有假設同一 commit；
3. Repository host-context guard 與 duplicate-runtime preflight 已獲批准，並由 Infrastructure commit `531e335`
   獨立保存；synthetic verification、provider-before-query duplicate detection 與 cleanup 後 provider／process
   compare 已通過；
4. 本次 cleanup／rebuild 所需 host-context VirtualBox／Vagrant commands 已逐次取得批准並完成；後續 real provider
   commands 仍須沿用 sandbox-outside approval flow；
5. 終止舊 duplicated VM／VirtualBox processes、隔離 stale IPC 與 destroy／rebuild exact declared VMs 已取得批准
   並完成；這不授權未來 destructive lifecycle；
6. 若需 managed sandbox／OS policy change，確認 administrator-owned mechanism。

下一步回到 active Phase 2：以新 runtime 修正 manifest ordering mismatch，重新執行 selected-config capacity 與
three-VM lifecycle acceptance。OS-level IPC isolation 持續作 optional hardening 調查，但不授權修改 device 或
sandbox policy。

## 12. Phase 2 關係與退出條件

本 incident 曾是 Phase 2 real-environment verification blocker；guard、cleanup 與 base clean rebuild 已完成。
後續 Phase 2 可在新 runtime 上重新取得 evidence，但：

- 可以執行不會間接啟動 Vagrant／VirtualBox 的 source inspection、render 與 pure tests；
- real VM lifecycle 仍須使用 approved host context，並同步核對 provider state、OS process inventory 與 active
  config identity；
- 目前不得宣稱 Slice 1 selected-runtime capacity evidence、Slice 7 validation gate 或 Phase 2 three-VM lifecycle
  acceptance 已完成。

Incident closure 需要：

1. Codex sandbox command-start guard 已生效並通過 context／command／tool-path fixtures；
2. sandbox-outside approval rules 已生效；
3. root `AGENTS.md` 與 testbed development policy 已保存穩定軟防護，且未被當成 hard enforcement；
4. repository direct paths 與 duplicate-runtime preflight 已 fail closed；
5. exact approved cleanup 完成；已通過；
6. 單一 VirtualBox control plane、provider state 與 actual processes exact match；已通過；
7. post-remediation mandatory initial review 與 user review；已通過。獨立 Infrastructure source commit `531e335`
   與 `records/` 整理；已通過。
