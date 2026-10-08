# Implementation And Remediation

已授權的程式／設定變更與 defect remediation 使用本模組。重大 slice 另用 [Planning](planning.md)；deployment 與
runtime 規則見 [Deployment](deployment.md) 與 [Runtime Safety](runtime-safety.md)；證據與測試選擇見
[Testing](testing.md)；independent review 與 finding closure 見 [Review](review.md)。閱讀 remediation 規則不授權在
review-only 或診斷工作中修改檔案。

## Change Safety

保留既有characterized production behavior，除非approved plan明確標示replaced。任何implementation strategy都不得
繞過、弱化或重新解釋approved contract、owner、validation、external boundary、destructive scope或required evidence。
下列是常見違規形式，不是可用來推論其他形式獲准的封閉清單：

- 新增平行 workaround；
- 弱化 validation；
- 以 mock／fake dependency 取代 required real boundary；
- 改變 component architecture ownership；
- 擴大 destructive scope；
- 將 required evidence 改寫為 optional。

Implementation repository中的artifact必須能只靠durable product／domain／contract context解釋其名稱、內容與存在
理由。若plan、issue、review iteration或其他work-tracking context被改名、歸檔或移除，而product behavior沒有改變，
implementation artifact也不應需要改名或修改。Project-management identity只屬於以planning、review、migration
history或verified result為目的的文件；某個詞若同時具有runtime domain meaning，是否允許取決於它在該處描述的實際
system semantics，而不是字面allowlist。

任何額外產生、傳遞或保存、只為證明identity、integrity或provenance的derived evidence，不論representation，都視為
新的contract。引入前必須證明authoritative source、每個direct consumer的必要性、failure behavior、
lifecycle scope，以及為何既有semantic validation或direct runtime observation不足；假設性的corruption、tampering或drift
不足以成立。Trusted local input優先由parse、schema、reference、ownership與domain semantics驗證；external
supply-chain contract與component-native identity則留在其原本boundary，不得由testbed複製另一套expected proof。
Hash-like mechanism 另受下方 [Hash-like Mechanism Special Gate](#hash-like-mechanism-special-gate) 的更嚴格special gate約束；符合本段的一般derived-evidence條件，不代表已取得
新增或擴張hash機制的授權。

移除既有derived evidence仍是contract change。必須先trace其producer、consumer與failure effect，並以semantic、
lifecycle或destructive-safety invariant保留必要保障，不能只因其形式看似冗餘就刪除。

## Hash-like Mechanism Special Gate

Hash（包括 SHA-256）、checksum、digest、fingerprint與其他以輸入內容推導值來證明相等性、完整性、identity或
provenance的機制，因為容易被視為低成本防護而在沒有真實failure consumer時擴散，必須視為獨立的高風險設計選擇。
本規則按機制語意判斷，不依賴演算法名稱、欄位名稱、編碼、檔案副檔名或工具allowlist；改名、改用其他演算法或
包進另一種artifact不會避開此gate。

Hash-like mechanism不是trusted local config、generated artifact、dataset、temporary transfer、cache、runtime status、
logging、provenance或test evidence的預設解法。一般性的「避免corruption／drift」、「增加可追蹤性」、「方便除錯」或
「未來可能有用」，以及只證明兩份bytes相同但沒有定義相同bytes為何是正確system state，都不足以成立新機制。

只有下列兩類boundary可使用hash-like value：

1. existing standard、external supply-chain／transport contract或component-native contract已定義該value，且testbed只在
   原boundary驗證或傳遞它；不得因此重新計算、複製或擴張成testbed-owned expected proof；
2. testbed-owned production contract有具體且直接的content-derived identity需求，現有較簡單的representation無法合理
   滿足，而且在實作前已於active plan逐項記錄並取得使用者明確決策。

第二類准入必須同時證明：authoritative producer；每個direct consumer；protected trust／transport／lifecycle
boundary；mismatch時會阻止或改變的具體production action；value的lifetime與cleanup owner；semantic validation、
structured identity或direct runtime observation不能提供同一invariant的理由；以及如何限制為單一最小representation，
不向無關config、manifest、filename、label、receipt、cache、log、status或test擴散。缺少任一項即不得新增。

不得只為hash-like mechanism建立存在性、格式、值相等、mismatch或移除結果的permanent repository test；只有該機制已
通過上述准入、而且test直接保護其durable production contract時才可測試。一次性inventory、cleanup或plan conformance
應留在active plan／review evidence，不得以新的hash-specific helper、fixture、test file或structural assertion固化。

既有hash-like mechanism不因本規則自動保留或自動刪除。修改前仍須trace完整producer、consumer與failure effect：有
外部或已批准contract者留在原boundary；無consumer、重複既有proof或只回應假設性風險者，依active plan移除並以必要的
semantic、lifecycle或destructive-safety invariant取代。Review必須逐一揭露本次新增、擴張、保留與移除的機制及其
contract依據；文字搜尋只能發現候選項，不能取代semantic trace。

## Remediation Loop And Scope

每個admitted in-scope finding依下列順序處理：

1. 記錄confirmed evidence、受影響owner與預期修正；
2. 依[Testing](testing.md#evidence-selection-and-permanent-test-admission)選擇直接證據；需要permanent behavioral test時，先確認它會因正確原因失敗；
3. 只修改關閉finding及其direct behavioral dependency所需的最小範圍；
4. 執行focused verification；
5. 立即執行[Review](review.md#follow-up-review)定義的targeted follow-up review；
6. 若finding仍未關閉，在相同邊界重複此loop；若出現architecture、contract、scope或verification變更，停止並走
   decision gate。

只要remediation留在approved boundary內，就在同一工作內直接繼續；一般progress update不是新的approval
gate。不得順手加入adjacent cleanup、future behavior或speculative resilience。Hash-like mechanism相關修正仍須同時遵守
[Hash-like Mechanism Special Gate](#hash-like-mechanism-special-gate)，不能藉remediation名義新增未經准入的hash或hash-specific permanent test。
