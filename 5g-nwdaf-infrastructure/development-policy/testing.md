# Testing And Verification Evidence

選擇證據、決定是否新增 permanent test，以及界定 verification 範圍時使用本模組。Remediation 流程見
[Implementation](implementation.md#remediation-loop-and-scope)；test code 的 independent review 見
[Review](review.md#initial-review)。

會啟動或可能啟動 provider process 的 tests 受
[Runtime Safety](runtime-safety.md#provider-host-context-and-process-inventory-gate) 約束；hash-like mechanism 的
permanent test 受 [Hash-like Mechanism Special Gate](implementation.md#hash-like-mechanism-special-gate) 約束。

## Evidence Selection And Permanent-test Admission

修正finding前，先界定受影響的supported behavior、authoritative owner與實際失效結果，再選擇能直接區分失效和正確
行為的最小證據。對可重現且值得長期防止的production behavior defect，優先識別既有的failing behavioral test；
現有suite不足時，才建立必要的測試。非持久的policy-conformance或一次性修正可由直接source／diff與受控重現證明，
不得為了符合test-first形式而新增permanent meta-test。

Permanent repository test必須保護獨立於本次work item與偶然implementation representation的durable contract，直接
exercise受驗owner、entrypoint及其可觀測結果。若合法的設定或等價實作變更會讓common test失敗，或test只反映
implementation representation而無法指出production behavior的錯誤，它就不能作為permanent regression evidence。
先複用既有owning suite；只有不同的owner或execution boundary確實無法由它承載時，才新增獨立test artifact。
實驗專屬的精確值須對照approved definition與selected run核對，不得由common test推斷為跨情境不變條件。

## Verification Boundary

Verification scope由本次變更實際影響的supported flow、owner、failure path與approved acceptance推導，使用足以支持
各項主張的最小直接證據；不得把所有可想像的edge case或既有test inventory當成每次變更的必測清單。
已適用的安全不變條件與active plan要求的real-environment evidence仍須保護，不能以縮小測試範圍為由弱化。
Focused verification通過後，除required final verification外，只有相關production change、失敗或具體未解疑點才擴大
或重跑驗證。

Passing repository tests不證明未覆蓋的production lifecycle沒有問題。任何synthetic、mock、static或局部測試，都只能
支持它實際執行到的owner與boundary；不得據此宣稱未被執行的external system、runtime environment或end-to-end flow已
通過。若active plan要求real environment，缺少該evidence時狀態應為
`Implementation Complete / Verification Incomplete`或保持更早的open state。
