# Planning And Implementation Slices

定義重大 implementation slice，或擴充既有 production flow 時使用本模組。Config source、pipeline、lifecycle 與
capacity 的設計規則見 [Deployment](deployment.md)；plan 文件本身的寫法見 [Documentation](documentation.md)。

## Slice Definition

重大實作開始前，active plan 至少必須明確列出下列責任，並補充實際 flow 具有但本清單未命名的 boundary：

1. operator-visible behavior 或 vertical flow；
2. authoritative inputs 與 generated artifacts；
3. VM、Guest process、Host container、network、storage 與 state owners；
4. external、component-native 與 private contracts；
5. start、status、logs、stop、reset、restart、recovery 與 failure paths；
6. 與變更風險相稱的驗證方式及 required real-environment evidence；
7. explicitly deferred behavior。

一個 named phase 可以包含多個 slices。每個 slice 的詳細計畫先與使用者討論確認；使用者指定本次工作要執行的
slice 後，在指定範圍內連續完成，並讓每個 slice 的範圍與證據可以分開 review。未被指定的 slice 不自動開始，
即使它屬於同一個 phase。指定多個 slice 時，每個 slice 先完成 focused verification 與
[independent review](review.md#initial-review)，再開始下一個指定的 slice；不得刻意累積整個 phase 成為難以 review 的
大型 working-tree diff。

「完成」只表示目前 approved slice 的 behavior 和 acceptance criteria 已關閉，不自動授權 future phase、
speculative hardening、unrelated cleanup、額外 VM 或 component architecture change。

## Existing-flow Extension

新增 deployment mode、topology、role、config version、lifecycle procedure 或 experiment type 前，active plan 必須
先命名 canonical existing flow，並從 authoritative inputs 到最終 cleanup／recovery 追蹤實際 end-to-end path。任何
會產生、轉換、傳遞、保存、觀測或清除 runtime-relevant state 的 owner 或 boundary 都必須納入，不能因未出現在
既有 checklist 就省略。

盤點至少要涵蓋 input selection、generation／validation、build／provision、stage／activation、infrastructure 與
process lifecycle、stimulus／trigger、observation／evidence，以及 stop／reset／recovery。這些是 routing dimensions，
不是封閉的 stage 清單；實際 flow 有額外 boundary 時必須擴充。

每個 baseline stage 必須標記為：

- reused without semantic change；
- adapted，並說明 owner、data flow 或 invariant 的改變；
- explicitly replaced；
- approved for deferral；
- not applicable，並附理由。

Baseline stage 不得因新 topology 看似簡單而被省略。若沿用既有名稱、selector、state、status、manifest field
或 operation，它必須保留原本 preconditions、postconditions、transition invariants 與 operator interpretation；
語意不同時必須使用不同 representation，或先完成 explicit contract decision。

## Out-of-scope Work

發現的工作若不阻塞 current slice，分類為：`future-phase handoff`、`legacy cleanup`、`optional hardening`、
`integration verification gap` 或 `unconfirmed risk`，不得偷偷拉入目前 diff。

| 分類 | 意義 |
| --- | --- |
| `future-phase handoff` | 已指派給具名後續 phase／slice 的工作 |
| `legacy cleanup` | 沒有造成目前功能失效的過時或未使用內容 |
| `optional hardening` | 超出目前 required contract 的 resilience |
| `integration verification gap` | 尚未在 real environment 中 exercise、且目前 acceptance 未要求該環境的 behavior |
| `unconfirmed risk` | 合理但沒有直接 evidence 的疑慮 |

Current slice 的 admitted defect 與更大範圍 remediation 的決策，依 [Review](review.md#finding-admission) 與入口的
[Decision Gates](../development_policy.md#decision-gates) 處理。
