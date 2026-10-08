# Runtime Safety

操作 real infrastructure provider、VM lifecycle，設計／執行 stop、reset 等 destructive operation，或撰寫、修改可能
啟動 provider process 的 production code、tests、Make targets 與 host scripts 時使用本模組。
Workspace 層級的 provider execution 邊界在 root `AGENTS.md` 的 Provider Execution Safety；兩者同時適用。

## Current Runtime Observation

Runtime state 不適用入口的[證據沿用規則](../development_policy.md#loading-and-evidence-reuse)。任何 real provider、
VM lifecycle 或 destructive operation 前，必須重新取得當下的 host 與 runtime observation；對話摘要、先前輸出、
記憶或既有 record 都不能代表目前 runtime state。

Observation 的取得本身受下方 gate 約束：先完成不啟動 provider 的 host-context 檢查與 host OS process inventory，
不得以 provider query 作為第一個 runtime observation。對話經過 context compaction、summarization 或 handoff 後，
執行這類 operation 前必須先從 disk 讀取本模組與 root `AGENTS.md` 的 Provider Execution Safety，不得只依摘要中的
規則行動。

## Provider Host-context And Process-inventory Gate

任何 real infrastructure provider operation，包括 operator 語意上看似唯讀的 status、list、show、validate 或
inventory query，都必須視為可能改變 provider control-plane state。Production code、repository tests、Make
targets 與 host scripts 不得留下繞過共用 guard 的 direct `vagrant`／`VBoxManage` call path。

對使用 VirtualBox 的 lifecycle：

- 共用 guard 必須在第一個 Vagrant／VirtualBox process 啟動前確認 execution 位於 approved host context，並確認
  `/dev/vboxdrv` 在 approved host device namespace 中可見且是 character device；此檢查用來阻止缺少 host
  VirtualBox device namespace 的 sandbox process 接觸共享 host IPC，不得先呼叫 provider，再以成功或錯誤輸出
  反推 context。
- Real provider verification 只能在 approved host context 執行；sandboxed repository-local tests 與一般 CI tests
  必須使用 synthetic fixtures、mock 或完全隔離的 provider substitute，不得啟動 host VirtualBox client。若有
  host-only integration test，必須明確分類並使用同一 approval／guard boundary。
- `vm-up` 或等價 startup 前，必須先從 host OS process inventory 解析實際 provider processes，再與 declared VM
  identity 及 provider state exact compare；不得以 provider query 作為第一個 runtime observation。
- Duplicate identity、process 存在但 provider state 不可見、provider／process mismatch、empty／invalid inventory
  或解析失敗都必須讓正常 lifecycle fail closed，不能繼續 start、halt、destroy 或 reset。事故 cleanup 必須改走
  fresh exact inventory、explicit targets 與獨立使用者批准，不得由正常 lifecycle 自動推斷或擴大 scope。
- Provider-reported `poweroff`、`not created` 或等價狀態不能單獨證明 VM 已停止；runtime acceptance 與 destructive
  scope 必須同時核對 OS process inventory、provider state、selected deployment 與 active config identity。

Command-start hook、approval rule、repository guard 與 OS process preflight 是不同防護層；任一層存在或 tests
通過，都不能被當成其他層已受保護或 real runtime acceptance 已完成。

## Runtime Identity, State And Destructive Safety

- 每個由selected deployment宣告或由runtime建立的logical entity與resource都必須有明確owner、identity、lifetime、
  observation path及cleanup responsibility。共用binary、component revision、physical machine或其他implementation
  resource，不代表可以共用logical identity或writable state。
- Runtime inventory必須能由authoritative deployment source完整重建，並與actual runtime及destructive scope作exact
  comparison；只驗證欄位形狀、只觀測selected subset或依賴固定role清單都不充分。
- 為比較selected、generated與active runtime而使用derived identity時，只能有一個authoritative representation，且
  必須直接服務mismatch detection與lifecycle fencing。不得為相同state建立平行expected identity、巢狀provenance或
  沒有failure consumer的衍生證明。
- Destructive operation必須先解析完整actual scope，再和selected intent及active identity核對。Reset必須同時防止漏清
  stale state與跨deployment誤清，且每個runtime owner都受相同scope invariant約束。
- Scenario switching 保持 explicit：不自動停止、覆蓋或 reset active scenario；需要使用者執行明確的 stop
  與 guarded reset。
- Reset 後若 acceptance 要求 deterministic seed restoration，必須驗證重新匯入後的 artifact identity；只驗證
  volume 為空不構成完成。
