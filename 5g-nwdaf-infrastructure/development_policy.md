# 5G NWDAF Infrastructure Development Policy

本入口依任務把新版 `5G_NWDAF_Infrastructure` 的 planning、implementation、deployment、review 與 delivery 工作路由到
適用規則。Repository map 與 workspace boundaries 在 workspace root `AGENTS.md`。每條詳細規則只有一個 owner 模組；
plan 定義 task-specific behavior 與 acceptance。Plan 內的通用 workflow 描述以現行 policy 為準，plan 自己的
task-specific acceptance criteria 仍然有效。通用 workflow 指 review 方式、重讀、user-review handoff、commit／push 的
批准步驟，即使它們寫在 plan 的 completion 清單內；plan 中的 runtime safety、provider、destructive scope、required
evidence 與 verification 要求不屬於通用 workflow，不因本句而放寬。

## Scope And Common Principles

本 policy 適用於：

- `5G_NWDAF_Infrastructure/` 的程式、設定、infrastructure、deployment、runtime lifecycle 與 tests；
- `testbed-docs/5g-nwdaf-infrastructure/plans/` 下的 implementation-oriented plans；
- 會改變 testbed topology、runtime identity、state、reset、experiment execution 或 evidence interpretation 的工作。

當工作同時改變 NWDAF、PyAnLF 或 PyMTLF component behavior 時，另讀 `nwdaf-docs/docs/development_policy.md`；
本 policy 只擁有 testbed orchestration、deployment 與 experiment runtime boundary，不取代 component development
policy。

在已同意的邊界內完成被要求的工作。優先使用既有 owner 與既有工具；unrelated refactoring 與 speculative hardening
不擴張任務。使用者目前的指示優先於 plan 或 skill 中較舊的 workflow 要求。

討論、診斷與 review 只授權檢視與回報；實作請求授權範圍內的修改與驗證。Git 授權與文件狀態轉換屬於
[Delivery](development-policy/delivery.md)。

## Source Ownership And Repository Boundaries

- `5G_NWDAF_Infrastructure/` 擁有 VM、network、deployment definitions、native config generation、process
  placement、Host ML runtime、dataset staging、lifecycle 與 reset tooling。
- `testbed-docs/5g-nwdaf-infrastructure/` 擁有 active plans、confirmed design、lab-specific operations、experiment
  definitions 與 verified records。
- `nwdaf-docs/` 擁有 Go NWDAF、PyAnLF、PyMTLF 的 current architecture、component contracts 與 Release 18
  evidence。
- `5G_Infrastructure/` 與 `testbed-docs/5g-infra/` 是舊 testbed 的歷史／遷移材料，不是新版 runtime contract。
- 各 repository 分別檢查 status、diff、branch 與 remote；不得在一個 commit 混合多個 repositories。

當 source、plan、generated config 與 actual runtime 互相矛盾時，先區分問題類型：

1. component behavior：以 component source、tests、`nwdaf-docs` active plan／verified record 為準；
2. intended testbed deployment：以 authoritative deployment source、active testbed plan、generated-artifact contract
   與 validators 為準；
3. actual running state：以實際 runtime owner／provider 的直接觀測，以及 [Runtime Safety](development-policy/runtime-safety.md#runtime-identity-state-and-destructive-safety) 定義的 selected／active identity
   evidence 為準；
4. 歷史背景：舊 README、archive 與 reports 只作 provenance，不覆蓋目前 contract。

## Task Routing

| 關注事項 | 模組 |
| --- | --- |
| Slice 範圍、既有 flow baseline、範圍外工作分類 | [Planning](development-policy/planning.md) |
| Config source、generation／validation pipeline、lifecycle 設計、capacity、experiment 比較 | [Deployment](development-policy/deployment.md) |
| Real provider／VM 操作；撰寫或修改可能啟動 provider process 的 code、tests、Make targets 或 host scripts；runtime identity；stop／reset 等 destructive scope | [Runtime Safety](development-policy/runtime-safety.md) |
| 程式／設定變更、defect remediation、derived evidence、hash-like mechanism | [Implementation](development-policy/implementation.md) |
| 證據選擇、permanent test、verification 範圍 | [Testing](development-policy/testing.md) |
| Independent review、findings、plan conformance | [Review](development-policy/review.md) |
| 文件分類、狀態、語言 | [Documentation](development-policy/documentation.md) |
| User-review handoff、commit、push | [Delivery](development-policy/delivery.md) |

## Loading And Evidence Reuse

讀取與目前決定相關的段落。主題適用時才跟隨連結；連結本身不代表整份被連結的文件都是必讀。同一工作的後續 turn
沿用已建立的 context，不必機械式重讀。

### Recovery After Context Compaction

用摘要恢復 objective、scope、decisions、progress 與 open items。Context 缺漏、不確定或已改變時，回到原始指示、
plan 或 evidence 查證。恢復不需要新的 plan 或 record。

### Evidence Applicability

Verification 依變更的 behavior 與 acceptance criteria 決定。既有結果涵蓋未變更的相關內容與條件；相關內容、
輸入或條件改變，或失敗未解時，須依影響範圍重新驗證。

### Runtime State

以上沿用規則不適用於 runtime state。任何 real provider、VM lifecycle 或 destructive operation 前，必須依
[Runtime Safety](development-policy/runtime-safety.md#current-runtime-observation) 重新取得當下的 host 與 runtime
observation。對話經過 context compaction、summarization 或 handoff 後，執行這類 operation 前必須先從 disk 讀取
Runtime Safety 模組與 root `AGENTS.md` 的 Provider Execution Safety，不得只依摘要中的規則行動。

## Decision Gates

在使用者的範圍內，一般 implementation 選擇直接繼續。

當繼續工作必須取得新的authority，或必須實質改變已批准的architecture、ownership、contract、dependency、scope或
acceptance時，停止並請使用者決策。下列是常見情況，不是decision gate的完整字面定義：

- agreed architecture、ownership、data/state flow 或 operator contract 必須改變；
- core assumption 為 false，必須替換 approved implementation strategy；
- 需要新增 VM、external dependency、service、persistence 或 config source；
- 必須弱化、刪除或延後 acceptance／verification；
- 需要實作 future phase behavior；
- 缺少必要 specification、component revision、permission、tooling 或 environment。

Blocker report 必須包含原假設、contradiction、可行選項、建議與 tradeoff，以及是否必須先更新 plan。

只有 optional cleanup、speculative hardening 或 future work 本身，不阻塞目前工作；它們依
[Planning](development-policy/planning.md#out-of-scope-work) 分類。
