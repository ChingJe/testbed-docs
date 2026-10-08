# Review And Conformance

Code／document review、finding closure 與完成前核對使用本模組。Review-only 工作產出 findings，不自動修改檔案。
已授權的 remediation 另用 [Implementation](implementation.md)；user-review handoff 用 [Delivery](delivery.md)。

## Finding Admission

Review發現只有在下列條件全部成立時，才是current slice必須修正的admitted finding：

1. 有直接source／diff evidence、deterministic reproduction、required evidence缺口，或明確的policy、plan、contract
   contradiction；
2. 問題存在於目前intended implementation diff、current slice支援的production path，或該path依賴的common boundary；
3. 問題違反目前適用的policy、approved plan、既有production contract、safety invariant或acceptance criterion；
4. 問題沒有被明確指派給future slice或approved deferral。

任一條件不成立時，必須分類為`future-phase handoff`、`legacy cleanup`、`optional hardening`、
`integration verification gap`或`unconfirmed risk`，不得把它當成current blocker。Passing test不能否定未被該test
exercise的直接production defect；反之，只有假設性風險而沒有直接evidence時也不能升格為confirmed finding。

Finding admission後，先判斷修正是否保留approved architecture、ownership、data flow、contract、dependency、scope與
verification level。若保留，直接進入[Remediation Loop](implementation.md#remediation-loop-and-scope)；若任一項必須改變，使用入口的[Decision Gates](../development_policy.md#decision-gates)，不得為了
繼續實作而降低finding嚴重度或改寫原acceptance。

## Initial Review

Implementation 與 focused verification 完成後，在 required final verification 與 user-review handoff 之前，
implementer 啟動一個 independent subagent，review production code、設定、tests 與 acceptance evidence，不等待
使用者額外要求。Self-check 不能取代這次 review。只改 tests 或設定的變更同樣需要 independent review；純 prose
變更不需要 subagent，除非使用者要求。

提供 reviewer：task scope、requirement／plan 位置、changed files，以及現有 verification results 與 gaps。Reviewer
直接檢視 primary evidence 並形成自己的結論，不以 implementer 的摘要作為核可依據。Review 是 read-only：reviewer
不修改檔案，不操作 runtime 或 provider，也不再委派另一個 reviewer；修正由 implementer 處理。無法進行
independent review 時回報此缺口，不得以 self-review 宣稱已取代。

沿用仍有效的既有結果；額外執行需要具體問題或缺口，review 本身不自動觸發另一輪測試。Review 至少檢查：

- complete intended slice diff，包括production、tests、config、generated examples與documentation；
- active plan、baseline stage map與working conformance map；
- direct call paths、failure paths、state owners與lifecycle dependencies；
- config provenance、generated artifacts與actual runtime identity；
- destructive scope、capacity、test quality與skipped verification；
- 每個artifact的identity、內容、存在理由與[Change Safety](implementation.md#change-safety)的work-tracking independence；
- 每個permanent test是否符合[Testing](testing.md#evidence-selection-and-permanent-test-admission)的durable regression proposition。

關鍵字、檔名或structure搜尋只能協助發現候選問題，不能取代semantic review。未通過的artifact必須依上方Finding Admission進入
finding admission與remediation，不能只因suite通過就保留。

## Plan Conformance

Active plan中已批准的production behavior、deliverable、acceptance／completion criterion與required command是
normative commitments；尚在探索或明確defer的敘述不因此成為實作驗收義務。Conformance map可合併指向同一結果的
敘述，但必須讓每項可驗收主張追溯至production path、適當的直接證據與結果，並揭露approved deferral或open gap；
不要求每句敘述各有一個deterministic test。

Initial review必須檢查implementation與plan的雙向一致性，以及baseline是否被無意改變。證據是否direct取決於它有無
實際exercise主張指定的owner、boundary、state與outcome；局部或替代性驗證不得推論成未執行的end-to-end保證。
Passing suite不能替代主張本身的證據。Required commitment缺少直接證據、未執行required command或被未經批准地
defer時保持open，並阻止ready／complete claim。

## Follow-up Review

Independent review 產生的 finding，其修正的 targeted follow-up review 交回同一個 independent reviewer；原 reviewer 不可用時，由替代 reviewer
依既有 findings 與 evidence，在相同 focused 範圍內進行。

每次in-scope remediation與focused verification後，立即執行targeted follow-up review，範圍限於修正及其直接依賴、
所用證據、以及可能受影響的production behavior與failure path。

Targeted follow-up通過即關閉該finding。不得為了證明repository沒有其他任何問題而反覆執行open-ended full review。只有
使用者明確要求重新full review，或remediation提供具體evidence顯示存在更廣泛且critical的current-slice regression時，
才重新擴大review範圍。Review中發現但不符合上方Finding Admission的問題依其分類記錄，不重開current finding。

## Final Conformance Check

所有 admitted findings 完成 remediation 與 targeted follow-up review 後，在 required final verification 及
ready／complete claim 之前，依目前已批准的 commitments 核對 claim-level conformance map，將每項主張和 final
production diff 及適當的直接證據重新核對；缺少、過期或只有 indirect evidence 的主張保持 open。

核對使用既有的 working conformance map 與仍有效的證據；policy、plan 或 source 有變更，或 context 有缺口時，依入口的
[Loading And Evidence Reuse](../development_policy.md#loading-and-evidence-reuse) 針對受影響的部分補讀原文與補驗證。
確認本 policy 與 active plan 要求的 final full／integration verification 已在 final production state 完成，然後更新
狀態。已完成且未受後續變更影響的結果不重跑；相關 production change、失敗或新證據缺口則須重新驗證。

Production behavior 已完成但 required direct evidence 仍 open 時，狀態只能是
`Implementation Complete / Verification Incomplete` 或相應較早狀態。Passing suite 不改變這個區分。

## Review Output

Review output必須先列confirmed current-slice findings，再分開blockers、approved deferrals、future-phase handoff、legacy
cleanup、optional hardening、integration verification gaps與unconfirmed risks。每項以白話說明behavior與consequence，
列出實際執行與尚未驗證的boundary，並明確判斷slice是partial、verification incomplete或ready for user review；不能只
以repository-wide suite成功宣稱slice完成。

## Review Documentation

只有使用者要求或active plan要求durable review documentation時才建立或更新review record。需要durable record時，每個
implementation phase／workstream維護一份ledger；每個finding記錄ID、status、owner slice、confirmed evidence、
remediation、verification與closing commit。後續remediation pass只在同一ledger追加簡短iteration，不為每次修正建立新的
完整review文件。只有architecture、product scope或canonical plan實質改變時，才另建獨立文件。
