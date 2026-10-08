# Review Handoff And Git Delivery

本模組擁有 review confirmation、文件狀態更新、commit message 與 Git 授權規則。

## User-review Handoff

Implementation 可交付 user review 時，回報 affected repositories、diff summary、verification results、remaining
gaps 與 active plan conformance。Intended changes 保持 unstaged／uncommitted，讓 IDE 直接顯示；active plan 保持 `Ready for User Review` 或
相符的 open state，除非使用者已要求 commit。

## Review Confirmation And Document Status

使用者明確要求 commit 目前成果，即代表 review 確認，並授權 stage 與 commit 該成果。準備 commit message 並完成
commit，不需另提 proposal 或等待 message 批准；只有在無法從 context 合理判斷預期範圍時才詢問。詢問 commit
方法、假設情境或討論未來提交，不視為對目前成果的 review 確認。

Commit 前，更新相關的既有 plans、implementation／review records 與 indexes，使其反映 user review 與實際進度，並把
這些狀態修改納入其 owning repository 對應的 commit；對 implementation repository 的 commit 請求同時涵蓋文件
repository 中這些相應的狀態修改。保留歷史 review 與 verification 事實；仍有 required implementation 或
real-environment acceptance 未完成時，plan 保持 open。Commit 請求不會讓未完成的工作變成完成。一般變更不需要
新的狀態文件。

## Commit

Commit 本次工作的變更，保留無關的既有修改，各 repository 分開 commit。Stage 後先檢查 staged diff，確認只包含本次
工作的內容，再建立 commit。工作涵蓋不同性質的變更時，選擇合理的
split。回報產生的 commit hashes 與 remaining work。

## Push And History Operations

同時要求 commit 與 push 才授權兩者；只要求 commit 不授權 push。使用使用者指定或既有的 remote 與 branch；目標
不明確時詢問。

Amend、rebase、reset、cherry-pick 與其他 history operation 需要針對該操作的明確授權。

## Commit Message Format

每個 commit 有英文 title 與非空的 description，中間空一行：

```text
<type>(<scope>): <summary>

<description>
```

Scope 可省略，`<type>: <summary>` 同樣有效。Type 為 `feat`、`fix`、`docs`、`refactor`、`test` 或 `chore`。Summary
使用祈使、簡潔、具體的寫法。Description 以適合該變更的短段落或條列，說明變更內容與其目的或結果。

Implementation commit message 描述技術差異與理由；`phase`、`batch`、`round`、`priority`、finding ID 或
remediation iteration 等內部 project-management／review 標籤不寫入。Documentation commit 在該識別字就是被修改的
文件時可以提及。

Commit message 不加任何 agent 或工具署名，包括 `Co-Authored-By`、session 連結與「generated with」類 trailer。
