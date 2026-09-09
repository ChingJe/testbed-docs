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
3. actual running state：以實際 runtime owner／provider 的直接觀測，以及 Section 7 定義的 selected／active identity
   evidence 為準；
4. 歷史背景：舊 README、archive 與 reports 只作 provenance，不覆蓋目前 contract。

## 2. Continuous Work Unit And Mandatory Re-read

`continue`、`fix the findings`、`review again`、`implement the plan` 或同一目標的 remediation 都屬於同一個
continuous work unit，除非使用者改變 objective、phase、repository boundary、architecture 或 verification
scope。

新的 continuous work unit 在第一次讀寫檔案、執行 command／tool、操作 runtime／Git，或開始 implementation、
review、verification 前，必須從 disk 讀取：

1. workspace root `AGENTS.md`；
2. 本 policy 中和目前 action 直接相關的 sections；
3. active phase plan 的 status、decisions、current slice、acceptance／completion criteria 與 conformance map。

同一 work unit 的後續 turn 不必機械式重讀。純討論、名詞解釋或不需查證目前 filesystem／repository／runtime
state 的澄清不觸發重讀。若 objective、phase、repository boundary、architecture 或 verification scope 改變，
已知 policy／active plan 在 disk 上更新，對話經過 context compaction、summarization、handoff，或 session
continuity 不確定，則必須在下一個 action 前重新讀取完整 `AGENTS.md`、本 policy 與 active plan；不能假設 root
`AGENTS.md` 會由 runtime 自動重新注入，也不能以對話摘要或記憶代替 disk 上的目前內容。

在宣告 implementation slice／phase ready、complete，或準備 user-review／commit checkpoint 前，必須再次完整
讀取本 policy 與 active plan，從目前文字重建 final conformance check。這是 completion gate，不是新的 work
unit。

## 3. Implementation Slice Definition

重大實作開始前，active plan 至少必須明確列出下列責任，並補充實際 flow 具有但本清單未命名的 boundary：

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

## 5. Configuration-source And Single-pipeline Gate

一套 deployment 必須有單一 authoritative high-level source，負責 plan／design 指定的 topology、identity、
placement 與 ownership；experiment input 負責 timing、traffic、monitoring、training 或其他 run behavior；
generated runtime directory 保存 processes 實際使用的 native artifacts。具體 source name、schema 與欄位 ownership
由 active plan／design 定義，不由本 policy 固定。不得把同一事實同時維護於多個人工高階來源。

任何新增的人工 deployment truth、operator selection state 或平行 generation／validation／lifecycle path，不論採用
何種檔案格式、儲存位置或工具形式，都是 architecture decision。提案前至少必須回答：

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
- topology-specific branch 可以存在，但所有由 shared source、contract 或 lifecycle 產生的 common validation 必須
  始終執行，不能依 topology 名稱或 branch 形態選擇性略過。

## 6. End-to-end Operational Lifecycle Gate

設計不能只證明中間 artifact 可以產生或單一 happy path 可以啟動。Implementation-ready plan 必須從最早的
authoritative input與artifact preparation，一直追蹤到最終observation、stop、cleanup與recovery；實際存在的每個
state transition、owner及跨process／machine／trust boundary都必須納入，不能用固定流程圖取代實際trace。

每個實際步驟都必須定義 selected intent、actual state 的authoritative observation、兩者不一致時的failure
behavior，以及partial failure、timeout、retry、rollback、restart與recovery。任何缺少、無效、過期或無法完整解析的
必要state，都不得被解讀成空操作成功或擴大成未界定的scope。

Selected deployment／generated runtime intent 必須能和 deployed actual state 及 resource inventory
相互核對。Wrong-config stop／reset 不得部分執行；status 不得只過濾 selected inventory 而隱藏 unexpected
runtime。Inventory 解析失敗或空清單不得退化成「不做任何事並成功」，也不得讓空 target arguments 擴大成
操作完整 runtime catalog。

Config activation 若跨多台 VM，必須定義 partial activation 的 detection 與 recovery。不得在部分 Guests 已換
config、其他 Guests 尚未完成時宣稱 selected topology active。

## 6.1 Provider Host-context And Process-inventory Gate

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

## 7. Runtime Identity, State And Destructive Safety

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

## 8. Capacity And Experiment Integrity

Capacity gate必須從selected deployment、runtime inventory與實際execution policy推導完整resource demand，並和每個
resource owner可用的bounded capacity比較；不得只檢查預先列出的resource種類、固定Host reserve或單一runtime domain。
Deployment引入任何新的resource owner、competition或dependency時，其capacity responsibility必須立即進入同一分析，
而不是等policy補上具體名稱後才受檢查。

容量不足是 decision blocker。不得在未重新決策下新增 VM、合併 logical NFs、共用 identity／state、降低必要
isolation，或悄悄縮小 acceptance criterion。

Controlled comparison必須識別、固定或明確記錄所有可能影響結果、但不是本次independent variable的因素。不能因某個
factor未列在既有experiment template就忽略；第一輪只跑通流程時，也不得把混有其他差異的結果歸因於單一設計變因。

## 9. Change Safety And Decision Gates

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
Hash-like mechanism 另受 Section 10 的更嚴格special gate約束；符合本段的一般derived-evidence條件，不代表已取得
新增或擴張hash機制的授權。

移除既有derived evidence仍是contract change。必須先trace其producer、consumer與failure effect，並以semantic、
lifecycle或destructive-safety invariant保留必要保障，不能只因其形式看似冗餘就刪除。

當繼續工作必須取得新的authority，或必須實質改變已批准的architecture、ownership、contract、dependency、scope或
acceptance時，停止並請使用者決策。下列是常見情況，不是decision gate的完整字面定義：

- agreed architecture、ownership、data/state flow 或 operator contract 必須改變；
- core assumption 為 false，必須替換 approved implementation strategy；
- 需要新增 VM、external dependency、service、persistence 或 config source；
- 必須弱化、刪除或延後 acceptance／verification；
- 需要實作 future phase behavior；
- 缺少必要 specification、component revision、permission、tooling 或 environment。

Blocker report 必須包含原假設、contradiction、可行選項、建議與 tradeoff，以及是否必須先更新 plan。

發現的工作若不阻塞 current slice，分類為：`future-phase handoff`、`legacy cleanup`、`optional hardening`、
`integration verification gap` 或 `unconfirmed risk`，不得偷偷拉入目前 diff。

## 10. Hash-like Mechanism Special Gate

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

## 11. Finding Admission And Remediation

### 11.1 Finding Admission Gate

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
verification level。若保留，直接進入本節remediation loop；若任一項必須改變，使用Section 9的decision gate，不得為了
繼續實作而降低finding嚴重度或改寫原acceptance。

### 11.2 Evidence Form And Test-first Remediation

Production behavior defect若能deterministic reproduction，應先建立或識別會因正確原因失敗的behavioral test，再做
最小production修正。Test必須直接exercise authoritative owner、entrypoint、lifecycle state與contract-defined outcome；
只證明helper被呼叫、literal token存在、structure剛好相同或相鄰operation成功，不構成behavioral regression evidence。

Artifact identity、work-tracking leakage、一次性cleanup assertion、documentation status或其他policy-conformance defect，
可以用直接source／diff contradiction作為admission evidence，不得為了遵守test-first而建立第二層meta-test。尤其不得
建立只驗證某檔案、名稱、token、欄位、hash、implementation helper或cleanup結果存在／不存在的permanent test。若此類
finding同時暴露durable production regression risk，才將最小behavior-oriented case併入既有owning suite。

Permanent repository test只有在能陳述一個durable regression proposition時才成立：給定supported owner、entrypoint與
state，特定操作必須產生contract-defined result或failure。Test的identity、pass／fail cause與維護理由不得依賴目前
work item、review history或偶然implementation representation；plan結束後仍須能獨立解釋它保護的behavior。只有distinct
owner、runner或contract boundary無法由既有suite合理承載時，才新增獨立test artifact，並在review中證明該必要性。

### 11.3 Remediation Loop And Scope

每個admitted in-scope finding依下列順序處理：

1. 記錄confirmed evidence、受影響owner與預期修正；
2. 對production behavior defect建立或識別failing behavioral test；不適合測試的policy-conformance defect保留直接
   source／diff evidence；
3. 只修改關閉finding及其direct behavioral dependency所需的最小範圍；
4. 執行focused verification；
5. 立即執行Section 12.3定義的targeted follow-up review；
6. 若finding仍未關閉，在相同邊界重複此loop；若出現architecture、contract、scope或verification變更，停止並走
   decision gate。

只要remediation留在approved boundary內，就在同一continuous work unit直接繼續；一般progress update不是新的approval
gate。不得順手加入adjacent cleanup、future behavior或speculative resilience。Hash-like mechanism相關修正仍須同時遵守
Section 10，不能藉remediation名義新增未經准入的hash或hash-specific permanent test。

### 11.4 Verification Boundary

新topology／lifecycle的tests必須從實際flow、state owners、failure paths與plan commitments推導完整coverage。下列是
常見verification dimensions，不是可用來省略未列出boundary的封閉清單：

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

Passing repository tests不證明未覆蓋的production lifecycle沒有問題。任何synthetic、mock、static或局部測試，都只能
支持它實際執行到的owner與boundary；不得據此宣稱未被執行的external system、runtime environment或end-to-end flow已
通過。若active plan要求real environment，缺少該evidence時狀態應為
`Implementation Complete / Verification Incomplete`或保持更早的open state。

## 12. Review Closure Protocol

### 12.1 Initial Review

Implementation與focused verification後，必須在不中斷的下一步完成一次initial review，不等待使用者額外要求；它應在
final full／integration verification與implementation commit checkpoint之前進行。Initial review至少檢查：

- complete intended slice diff，包括production、tests、config、generated examples與documentation；
- active plan、baseline stage map與working conformance map；
- direct call paths、failure paths、state owners與lifecycle dependencies；
- config provenance、generated artifacts與actual runtime identity；
- destructive scope、capacity、test quality與skipped verification；
- 每個artifact的identity、內容、存在理由與Section 9的work-tracking independence；
- 每個permanent test是否符合Section 11.2的durable regression proposition。

關鍵字、檔名或structure搜尋只能協助發現候選問題，不能取代semantic review。未通過的artifact必須依Section 11進入
finding admission與remediation，不能只因suite通過就保留。

### 12.2 Plan-conformance Gate

Active plan中任何承諾production behavior、deliverable、acceptance／completion criterion或required command的敘述都是
normative item，除非明確標為background、non-goal、optional或approved deferral。Conformance map對每個item記錄：

- production path；
- direct deterministic test；
- verification command／result；
- approved deferral／plan change；
- open gap。

Initial review必須同時檢查implementation-to-plan、plan-to-implementation與baseline-to-plan。Test只有在直接exercise
criterion指定的production owner、entrypoint、lifecycle state與outcome時才是direct evidence。Mock取代受驗boundary、
只測相鄰phase／operation、只證明call發生但未驗證downstream effect，或把多個tests推論組合成未被實際執行的end-to-end
guarantee，都只能算indirect evidence。

Passing suite不能替代criterion-specific evidence。Required item缺少production path、只有indirect evidence、未執行
required command或被未經批准地defer時保持open，並阻止ready／complete claim。

### 12.3 Targeted Follow-up Review

每次in-scope remediation與focused verification後，立即執行targeted follow-up review，只檢查：

- remediation diff；
- original finding的direct dependencies；
- existing或newly added owning tests；
- remediation能直接影響的production behavior與failure path。

Targeted follow-up通過即關閉該finding。不得為了證明repository沒有其他任何問題而反覆執行open-ended full review。只有
使用者明確要求重新full review，或remediation提供具體evidence顯示存在更廣泛且critical的current-slice regression時，
才重新擴大review範圍。Review中發現但不符合Section 11.1的問題依其分類記錄，不重開current finding。

### 12.4 Final Fresh-read Conformance Gate

所有admitted findings完成remediation與targeted follow-up review後，但在final full／integration verification及
ready／complete claim之前，必須：

1. 從disk重新完整讀取目前development policy與active plan；
2. 從目前文字重建final conformance map，包括所有normative statements、verification matrix、review checklist、
   completion criteria、required commands與approved deferrals；
3. 將每個item和final production diff、exact test path及verification result重新核對；
4. 將缺少、過期或只有indirect evidence的item保持open；
5. 執行required final full／integration verification，再更新最終狀態。

先前摘要、記憶中的requirements、舊conformance map或之前通過的test result都不能取代本gate。Production behavior已完成但
required direct evidence仍open時，狀態只能是`Implementation Complete / Verification Incomplete`或相應較早狀態。

### 12.5 Review Output

Review output必須先列confirmed current-slice findings，再分開blockers、approved deferrals、future-phase handoff、legacy
cleanup、optional hardening、integration verification gaps與unconfirmed risks。每項以白話說明behavior與consequence，
列出實際執行與尚未驗證的boundary，並明確判斷slice是partial、verification incomplete或ready for user review；不能只
以repository-wide suite成功宣稱slice完成。

### 12.6 Review Documentation

只有使用者要求或active plan要求durable review documentation時才建立或更新review record。需要durable record時，每個
implementation phase／workstream維護一份ledger；每個finding記錄ID、status、owner slice、confirmed evidence、
remediation、verification與closing commit。後續remediation pass只在同一ledger追加簡短iteration，不為每次修正建立新的
完整review文件。只有architecture、product scope或canonical plan實質改變時，才另建獨立文件。

## 13. Documentation And Status Discipline

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

## 14. User Review, Commit And Push Gates

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

## 15. Common Workflow

1. 重讀 root `AGENTS.md`、本 policy 與 active plan；若發生 context compaction，完整重讀三者。
2. 確認 active slice、repository owners、source contract、acceptance evidence 與 deferred work。
3. 若擴充既有 flow，完成 baseline stage disposition map。
4. 將所有 normative plan items 建立 working conformance map。
5. 追蹤 current production path、failure path、state 與 direct dependencies。
6. 確認使用現有 authoritative source／pipeline；新 config source、entrypoint 或 architecture 先經 decision gate。
7. 若工作涉及hash-like mechanism，先完成Section 10的special gate；未取得准入不得實作或建立permanent test。
8. 對可測的production behavior建立characterization／failing tests，再完成最小完整slice；非行為型
   policy-conformance問題使用直接evidence，不建立meta-test。
9. 執行 focused verification。
10. 在final full verification前完成一次Section 12.1的initial review，依Section 11.1 admission所有findings。
11. 對admitted in-scope findings執行remediation、focused verification與targeted follow-up review，直到關閉或遇到
    decision gate；不為每次修正重新執行open-ended full review。
12. 依需要更新同一份phase／workstream review ledger，不建立per-remediation完整review文件。
13. 重新完整讀取本policy與active plan，依Section 12.4重建final conformance map。
14. 執行required final full／integration verification，將indirect或unavailable evidence保持open。
15. 完成documentation language-consistency pass。
16. 保持changes unstaged／uncommitted，提出user-review handoff並停止。
17. Review confirmation後提出commit proposal，再等待explicit commit approval。
18. 只建立approved commits；另行取得push approval。
