# 5G NWDAF Infrastructure Development Policy

本文件定義新版 `5G_NWDAF_Infrastructure` 的 planning、implementation、deployment、review 與 verification
規則。它適用於：

- `5G_NWDAF_Infrastructure/` 的程式、設定、infrastructure、deployment、runtime lifecycle 與 tests；
- `testbed-docs/5g-nwdaf-infrastructure/plans/` 下的 implementation-oriented plans；
- 會改變 testbed topology、runtime identity、state、reset、experiment execution 或 evidence interpretation 的工作。

先讀 workspace root `AGENTS.md` 以確認 repository boundaries、source routing、continuous work unit、user review
與 Git approval gates。當工作同時改變 NWDAF、PyAnLF 或 PyMTLF component behavior 時，另讀
`nwdaf-docs/docs/development_policy.md`；本文件只擁有 testbed orchestration、deployment 與 experiment
runtime boundary，不取代 component development policy。

---

## 1. Source Ownership And Repository Boundaries

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
3. actual running state：以 Guest active config hash、container labels、process inventory 與 runtime evidence 為準；
4. 歷史背景：舊 README、archive 與 reports 只作 provenance，不覆蓋目前 contract。

## 2. Continuous Work Unit And Mandatory Re-read

`continue`、`fix the findings`、`review again`、`implement the plan` 或同一目標的 remediation 都屬於同一個
continuous work unit，除非使用者改變 objective、phase、repository boundary、architecture 或 verification
scope。

在每個 follow-up turn 開始時，必須從 disk 重新讀取：

1. workspace root `AGENTS.md`；
2. 本 policy 中和目前 action 直接相關的 sections；
3. active phase plan 的 status、decisions、current slice、acceptance／completion criteria 與 conformance map。

若對話經過 context compaction、summarization、handoff 或 session continuity 不確定，必須重新讀取完整
`AGENTS.md`、本 policy 與 active plan；不能假設 root `AGENTS.md` 會由 runtime 自動重新注入，也不能以對話
摘要或記憶代替 disk 上的目前內容。

在宣告 implementation slice／phase ready、complete，或準備 user-review／commit checkpoint 前，必須再次完整
讀取本 policy 與 active plan，從目前文字重建 final conformance check。這是 completion gate，不是新的 work
unit。

## 3. Implementation Slice Definition

重大實作開始前，active plan 必須明確列出：

1. operator-visible behavior 或 vertical flow；
2. authoritative inputs 與 generated artifacts；
3. VM、Guest process、Host container、network、storage 與 state owners；
4. external、component-native 與 private contracts；
5. start、status、logs、stop、reset、restart、recovery 與 failure paths；
6. focused tests、full checks 與 required real-environment evidence；
7. explicitly deferred behavior。

一個 named phase 可以包含多個 slices。每個 slice 應先完成 focused verification、mandatory initial review 與
user-review handoff，再擴張到下一個 cross-cutting slice。不得刻意累積整個 phase 成為難以 review 的大型
working-tree diff。

「完成」只表示目前 approved slice 的 behavior 和 acceptance criteria 已關閉，不自動授權 future phase、
speculative hardening、unrelated cleanup、額外 VM 或 component architecture change。

## 4. Existing-flow Extension Gate

新增 deployment mode、topology、role、config version、lifecycle procedure 或 experiment type 前，active plan 必須
先命名 canonical existing flow，並逐階段盤點：

1. authoritative deployment selection；
2. experiment／traffic inputs；
3. config generation／rendering；
4. strict validation 與 component-native loaders；
5. generated runtime artifacts／manifest；
6. infrastructure definition 與 machine lifecycle；
7. config stage／activate 與 network reconciliation；
8. deployed service start／status／logs／stop；
9. auxiliary runtime build／start／health／status／logs／stop；
10. subscriber、dataset 與 traffic stimulus；
11. subscription-driven 或 explicit trigger；
12. reset、seed restoration、restart 與 cleanup evidence。

每個 baseline stage 必須標記為：

- reused without semantic change；
- adapted，並說明 owner、data flow 或 invariant 的改變；
- explicitly replaced；
- approved for deferral；
- not applicable，並附理由。

Baseline stage 不得因新 topology 看似簡單而被省略。若沿用既有名稱、selector、state、status、manifest field
或 operation，它必須保留原本 preconditions、postconditions、transition invariants 與 operator interpretation；
語意不同時必須使用不同 representation，或先完成 explicit contract decision。

## 5. Configuration-source And Single-pipeline Gate

一套 deployment 必須有單一 authoritative high-level source，負責 plan／design 指定的 topology、identity、
placement 與 ownership；experiment input 負責 timing、traffic、monitoring、training 或其他 run behavior；
generated runtime directory 保存 processes 實際使用的 native artifacts。具體 source name、schema 與欄位 ownership
由 active plan／design 定義，不由本 policy 固定。不得把同一事實同時維護於多個人工高階來源。

新增任何人工維護的 YAML source、top-level directory、selector、renderer、checker、Compose catalog 或
lifecycle entrypoint 都是 architecture decision。提案前必須回答：

1. 現有 authoritative source／entrypoint 為何無法合理擴充；
2. 新來源或路徑的 owner 與 lifetime；
3. 與既有資訊是否重複，以及如何防止 drift；
4. operator 需要增加哪些選擇、同步或記憶負擔；
5. migration、validation、rollback 與 removal plan。

未記入 active plan 並取得使用者確認前，不得新增。由 authoritative source 生成的 native config、manifest 或
runtime-specific orchestration artifact 是 runtime output，不是另一份人工 source。

對同一 lifecycle 的新 topology：

- 優先擴充既有 renderer、checker 與 lifecycle；
- common behavior 必須只有一條 pipeline；
- 不得先生成另一 topology 的 artifacts、刪除後再重建；
- 不得以第二套 checker shortcut 跳過 common validation；
- topology-specific branch 可以存在，但 common Core／RAN／network／identity／scenario／native checks 必須
  始終執行。

## 6. End-to-end Operational Lifecycle Gate

設計不能只證明 config 能 render。Implementation-ready plan 必須追蹤完整方向：

```text
authoritative deployment + experiment input
→ render
→ validate
→ stage
→ activate
→ start
→ status／logs
→ stop
→ reset
→ seed／state recovery
```

每一步必須定義：

- authoritative selected identity；
- actual active identity 與 observation path；
- declared／actual mismatch behavior；
- partial failure、timeout、rollback 與 restart behavior；
- unexpected process／container／volume 的呈現與處置；
- empty、missing、invalid 或 stale inventory 的 fail-closed behavior。

Selected deployment／generated-config identity 必須能和 deployed active config、runtime labels 及 process inventory
相互核對。Wrong-config stop／reset 不得部分執行；status 不得只過濾 selected inventory 而隱藏 unexpected
runtime。Inventory 解析失敗或空清單不得退化成「不做任何事並成功」，也不得讓空 target arguments 擴大成
操作完整 runtime catalog。

Config activation 若跨多台 VM，必須定義 partial activation 的 detection 與 recovery。不得在部分 Guests 已換
config、其他 Guests 尚未完成時宣稱 selected topology active。

## 7. Runtime Identity, State And Destructive Safety

- 每個 NWDAF NF 必須有獨立 NF Instance ID、process、config、endpoint、NRF registration、log identity、
  runtime state 與對應 ML process／state。
- 共用 binary、component revision 或 physical VM 不代表共用 NF identity 或 writable state。
- Manifest inventory 必須能由 trusted authoritative deployment source 完整重建並 exact compare，不只驗證欄位
  形狀。
- Guest units、Host containers、ports、volumes、subscription mode、coordinator 與 reset scope 必須一一對應。
- Destructive scope 必須先以 read-only checks 解析 exact target，再和 selected topology 及 actual runtime 比對。
- Reset 必須同時防止漏清 stale state 與跨 topology 誤清；Host 與 Guest destructive guards 都要涵蓋 actual
  topology，不能依賴舊角色 hard-code 或只有外層檢查。
- Scenario switching 保持 explicit：不自動停止、覆蓋或 reset active scenario；需要使用者執行明確的 stop
  與 guarded reset。
- Reset 後若 acceptance 要求 deterministic seed restoration，必須驗證重新匯入後的 artifact identity；只驗證
  volume 為空不構成完成。

## 8. Capacity And Experiment Integrity

Capacity gate 必須根據 selected runtime inventory 計算，而不是只檢查固定 Host reserve：

- VM CPU、memory 與 disk；
- selected Guest process count；
- selected Compose CPU／memory limits；
- image build 與 Docker overhead；
- GPU participant count、GPU availability 與必要的 GPU memory；
- IP aliases、published ports、networks、volumes 與 filesystem headroom。

容量不足是 decision blocker。不得在未重新決策下新增 VM、合併 logical NFs、共用 identity／state、降低必要
isolation，或悄悄縮小 acceptance criterion。

Controlled comparison 必須區分 topology、algorithm、data partition、traffic stimulus、seed、training effort 與
timing。第一輪只跑通流程時，文件不得把不同 algorithm 的結果宣稱為純 topology effect。

## 9. Change Safety And Decision Gates

保留既有 characterized production behavior，除非 approved plan 明確標示 replaced。不得因 planned approach
不方便而：

- 新增平行 workaround；
- 弱化 validation；
- 以 mock／fake dependency 取代 required real boundary；
- 改變 component architecture ownership；
- 擴大 destructive scope；
- 將 required evidence 改寫為 optional。

只有下列情況停止並請使用者決策：

- agreed architecture、ownership、data/state flow 或 operator contract 必須改變；
- core assumption 為 false，必須替換 approved implementation strategy；
- 需要新增 VM、external dependency、service、persistence 或 config source；
- 必須弱化、刪除或延後 acceptance／verification；
- 需要實作 future phase behavior；
- 缺少必要 specification、component revision、permission、tooling 或 environment。

Blocker report 必須包含原假設、contradiction、可行選項、建議與 tradeoff，以及是否必須先更新 plan。

發現的工作若不阻塞 current slice，分類為：`future-phase handoff`、`legacy cleanup`、`optional hardening`、
`integration verification gap` 或 `unconfirmed risk`，不得偷偷拉入目前 diff。

## 10. Test-first Remediation And Verification

Confirmed defect 若能 deterministic reproduction，應先建立或識別 failing test，再進行最小 remediation。每次
remediation 後執行 focused verification 與 targeted follow-up review；只要不改變 approved architecture、
contract 或 verification level，可在同一 work unit 持續進行。

新 topology／lifecycle 的 tests 至少依適用範圍涵蓋：

- production baseline regression；
- complete selected deployment render 與 common／topology-specific validation；
- invalid、missing 與 empty manifest／inventory；
- wrong selected config 與 active identity mismatch；
- unexpected Guest unit／Host container／volume；
- partial activation、start failure、rollback 與 partial stop；
- exact reset scope 與 seed restoration；
- topology switching；
- capacity rejection；
- active plan 要求的 real infrastructure／container／accelerator integration evidence。

Passing repository tests 不證明未覆蓋的 production lifecycle 沒有問題。Host-only、mock 或 config tests 不得
宣稱 real VM、5GC、UE、UPF、ADRF、MongoDB、Docker GPU 或 end-to-end experiment acceptance。若 active plan
要求 real environment，缺少該 evidence 時狀態應為 `Implementation Complete / Verification Incomplete` 或保持
更早的 open state。

## 11. Mandatory Review And Plan Conformance

Implementation 與 focused verification 後，必須在不中斷的下一步完成一次 initial review，不等待使用者額外
要求。Review 至少檢查：

- complete slice diff；
- active plan、baseline stage map 與 normative conformance map；
- direct call paths、failure paths 與 lifecycle dependencies；
- config provenance、generated artifacts 與 actual runtime identity；
- destructive scopes、capacity、tests 與 skipped verification。

Active plan 中任何承諾 production behavior、deliverable、acceptance／completion criterion 或 required command 的
敘述都是 normative item，除非明確標為 background、non-goal、optional 或 approved deferral。Conformance map
對每個 item 記錄：

- production path；
- deterministic test；
- verification command／result；
- approved deferral／plan change；
- open gap。

Initial review 必須同時檢查 implementation-to-plan、plan-to-implementation 與 baseline-to-plan。Passing suite
不能替代 criterion-specific evidence。Required item 只有 indirect evidence 時仍保持 open。

Review output 必須分開 confirmed current-slice defects、deferred work、legacy cleanup、hardening、integration
gaps 與 unconfirmed risks，並明確說明執行了什麼、尚未驗證什麼，以及 slice 是 partial、verification
incomplete、ready for user review 或 completed。

## 12. Documentation And Status Discipline

- stable workflow／review rules 放在本 policy；workspace routing 放在 root `AGENTS.md`；phase-specific decisions
  與 conformance map 放在 active plan。
- 未完成工作與 findings 放在 `plans/`；confirmed design 放在 `design/`；experiment definition 放在
  `experiments/`；只有具備 required evidence 並通過 user review 的結果放入 `records/`。
- Prototype 能 render、focused tests 通過或 implementation 存在，不構成完成 record。
- 文件狀態應使用能反映 evidence 的 open state，例如 `Implementation`、`Review Pending`、
  `Implementation Complete / Verification Incomplete`、`Ready for User Review` 或 `Completed`。
- 不在 README 複製容易失效的 current progress；README 只提供穩定入口、責任與 navigation。

文件 prose language 依序由 explicit user instruction、existing document dominant language、sibling series
language、repository default 決定。Code identifiers、paths、schema fields 與 API names 保持 English。交付前必須
重新閱讀完整 changed document，並和至少一份 current sibling 比較，完成獨立 language-consistency pass；
不得只做 diff spot check。

## 13. User Review, Commit And Push Gates

Implementation、review、verification 或 plan conformance 完成，只授權準備 user-review handoff，不授權 staging
或 commit。User review 前：

- intended changes 保持 unstaged／uncommitted，讓 IDE 直接顯示；
- active plan 保持 open state；
- 回報 affected repositories、diff summary、verification results 與 remaining gaps；
- 停止並等待使用者確認 review result。

Review confirmation 不等於 commit approval。之後必須提出 read-only commit proposal，列出：

- 每個要 commit 的 repository；
- included files 與 change summary；
- verification results／remaining gaps；
- proposed split 與完整 commit messages；
- excluded unrelated／pre-existing changes。

只有使用者明確批准目前 proposal 後才能 stage approved files、檢查 staged diff 並建立 approved commits。
Proposal materially 改變時必須重新批准。Commit approval 不授權 amend、rebase、reset、cherry-pick 或 push；
push 需要另外明確批准。

## 14. Common Workflow

1. 重讀 root `AGENTS.md`、本 policy 與 active plan；若發生 context compaction，完整重讀三者。
2. 確認 active slice、repository owners、source contract、acceptance evidence 與 deferred work。
3. 若擴充既有 flow，完成 baseline stage disposition map。
4. 將所有 normative plan items 建立 working conformance map。
5. 追蹤 current production path、failure path、state 與 direct dependencies。
6. 確認使用現有 authoritative source／pipeline；新 config source、entrypoint 或 architecture 先經 decision gate。
7. 建立 characterization／failing tests，再完成最小完整 slice。
8. 執行 focused verification。
9. 立即進行 mandatory initial review，分類 findings。
10. 對 admitted in-scope findings 執行 test-first remediation 與 targeted follow-up review，直到關閉或遇到 decision
    gate。
11. 重新完整讀取本 policy 與 active plan，重建 final conformance map。
12. 執行 required full／integration verification，將 indirect 或 unavailable evidence 保持 open。
13. 完成 documentation language-consistency pass。
14. 保持 changes unstaged／uncommitted，提出 user-review handoff 並停止。
15. Review confirmation 後提出 commit proposal，再等待 explicit commit approval。
16. 只建立 approved commits；另行取得 push approval。
