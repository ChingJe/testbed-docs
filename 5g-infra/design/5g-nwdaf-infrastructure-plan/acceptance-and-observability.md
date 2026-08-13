# Fresh Acceptance、Cleanup 與 Observability

[返回主計畫](../5g-nwdaf-infrastructure-plan.md)

本文件保存原計畫第 16.33–16.35 節的驗收結果、cleanup 修正與後續觀測規劃。

## 16.33 Fresh User Full-core Black-box Acceptance

2026-08-13 將三台 VM、production Compose resources/images、ignored local/generated configs與
datasets清至fresh state後，交由不繼承對話且只能讀取Infrastructure repository的subagent依新版
文件操作。它只使用公開Make commands，成功在10m55s內從零provision三台VM，2m59s內啟動
aggregate stack，並完成六UE registration/PDU Sessions、雙NWDAF subscription/callback、Path A
degradation、A/B兩輪local training、C sample-count-weighted FedAvg、ADRF publication、A/B
reprovision、generation cutover與post-cutover evaluated accuracy。從config準備到VM halt約47分鐘。

這證明root README與topic guides足以支援fresh full-core business E2E，但也發現五項observability
落差：status未顯示UE/PDU summary、focused logger無法匹配Consumer特殊unit、單一subscription
snapshot只保留最後callback、缺少closure summary/watcher，以及Host/component log時區未說明。

正常`experiment-stop`確定刪除兩筆Consumer resources並執行完整40秒grace；internal Model Monitor
cleanup中，一條DELETE 503後retry 404並標記removed，另一條在shutdown前DELETE 503後沒有成功
retry/removed evidence。因此business closure通過，teardown cleanup只部分確認，需優先診斷
fixed grace後未驗證收斂及shutdown retry ownership。現場保留為三台VM poweroff、五個containers
exited、volumes/network/images/state retained，Git與submodules clean。完整證據見
[Fresh User Full-core Black-box Acceptance](../../reports/5g-nwdaf-infrastructure/fresh-user-full-core-black-box-acceptance-2026-08-13.md)。

## 16.34 Log-correlated Model Monitor Cleanup

Fresh black-box run顯示固定40秒grace可能在第二條internal Model Monitor DELETE尚未retry完成前就
停止ML containers。為避免修改PyAnLF／PyMTLF，teardown ownership留在Infrastructure：

1. `experiment-stop`在Consumer DELETE前讀取PyMTLF-C logs，以`active`與`removed`事件重建目前
   active的subscription ID集合。
2. 在DELETE前掛接單一PyMTLF-C log follower，避免反覆執行`docker logs --since`掃描Docker
   `local` logging driver。
3. Consumer exact DELETE完成後，每兩秒解析本機暫存stream；預期IDs全部出現`removed`即提前
   結束等待。
4. 預設timeout為210秒；持續`503`、log follower中止或timeout只產生明確warning及pending IDs，
   不永久阻塞WebConsole、ML與Guest teardown。
5. PyAnLF的public registration ID與PyMTLF backend resource ID不同，不作跨層ID等值關聯。

實作新增`ML_CLEANUP_TIMEOUT_SECONDS`（預設210）與內部poll interval，舊
`ML_CLEANUP_GRACE_SECONDS`只保留deprecated fallback。Parser與convergence wait已整合到既有
repository runner，完整`make test`通過。

同日下午以GPU `fl-closure-smoke`完成production runtime驗證。兩條cutover後subscriptions在
Consumer DELETE後分別立即removed與約97秒後removed；較慢者歷經三次`503`，最後retry得到
`404`並在PyMTLF-C shutdown前留下明確removed evidence。兩筆Consumer resources、23個Guest
units、五個ML containers及三台VM依序正常停止，沒有pending-ID timeout warning。這證實log
correlation能處理本次實際延遲，且210秒涵蓋本次觀察值；持續`503`仍由bounded timeout避免永久
阻塞。完整證據見
[Model Monitor Cleanup Runtime Validation](../../reports/5g-nwdaf-infrastructure/model-monitor-cleanup-runtime-validation-2026-08-13.md)。

## 16.35 Observability Contract Audit and Remediation Plan

Fresh-user black-box acceptance與後續runtime cleanup validation證明主要流程可重現，但也顯示目前
公開status/log介面有「process active不等於business-ready」、「特殊unit無法查詢」、「A/B資訊
被最後一筆callback覆蓋」等落差。2026-08-13對Infrastructure command surface、systemd units、
Compose labels、Consumer state與文件承諾進行read-only audit；本節只規劃，不代表已實作。

### Confirmed findings

1. `logs.sh`把所有VM service固定映射成`5g-nwdaf@<service>.service`。一般NF、UE與WebConsole
   使用template unit，因此可查；Consumer實際為`5g-nwdaf-consumer.service`，Network實際為
   `5g-nwdaf-network.service`，focused query會查錯unit。
2. 預設`make logs`使用`--service '*'`，只匹配`5g-nwdaf@*.service`，因此文件所稱「all current
   VM journals」實際漏掉Consumer與Network logs。
3. VM logs使用`journalctl -o cat`，Docker logs未加`--timestamps`；跨Host、Guest與container
   對照時缺乏一致timezone evidence。`--since`也分別由Guest journal與Host `date`解讀，相同的
   `today`或相對時間不保證代表同一absolute instant。
4. `services-status`只顯示23個systemd unit state，但command reference聲稱包含registration/session
   state。UERANSIM process為`active`不代表六個UE已完成registration與PDU Session。
5. Consumer只保存global `notificationCount`與最後一個callback request的correlations；A/B報告
   交錯時，單一snapshot無法證明兩路都持續送達。
6. Consumer callback由`ThreadingHTTPServer`處理；現有lock只各自包住單次read與write，沒有包住
   完整read-modify-write。兩個callback threads可能遺失increment。`subscriptions-stop`另啟一個
   Consumer CLI process修改同一state file，process-local lock也無法避免callback與DELETE state
   互相覆蓋。
7. `subscriptions-status`把Consumer status錯誤導向`/dev/null`並以`|| true`吞掉失敗；可能只顯示
   `consumer=active`卻沒有state body，command仍回傳成功。
8. `observe --once`對VM、Guest、WebConsole、ML與subscription查詢採best-effort suppression；VM
   查詢失敗甚至可能只留下空白。`experiment-status`直接呼叫它，因此完整snapshot缺段時仍可能
   exit 0，與「complete experiment state」語意不符。
9. Full-core closure仍需人工跨PyMTLF-C與A/B logs辨識trigger、preparation、rounds、publication、
   adoption、cutover與post-cutover accuracy；尚無current-run milestone summary。

以下不是同類錯誤：WebConsole本來就是template unit且現有查詢正確；subscription lifecycle scripts
已明確使用Consumer特殊unit；Network unit不列入23個experiment process summary是刻意的lifecycle
邊界；ML logs依Compose project/service labels選取，不受systemd名稱影響。修正不改unit naming、
不新增user-facing Make command，也不修改NWDAF、PyAnLF、PyMTLF或其他submodule。

### Target behavior

公開介面應形成一致的三層觀測：

- `services-status`回答Guest processes與六UE目前這次invocation的registration/PDU結果；
- `subscriptions-status`回答兩個exact resources與A/B各自callback進度；
- `experiment-status`誠實彙整VM、Guest、WebConsole、ML、subscriptions及current-run FL milestones；
- `logs.sh`提供完整或focused的所有owned units/containers，並以一致absolute time語意輸出。

Status只陳述可證明的狀態。保存於Consumer state的`active`代表local saved resource state，不宣稱
remote GET驗證；目前NFs沒有提供適合的GET/list介面，不能用health endpoint替代resource existence。

### Implementation plan

**A. Log source correctness and time normalization**

1. 在Host shared helper集中定義owned VM log units：template services、Core-only Consumer、三台VM
   Network。Focused `consumer`／`network`映射實際unit；其他名稱保持template mapping。
2. `--service '*'`在Core選取template、Consumer與Network，在Path VMs選取template與Network；不向
   明知不擁有Consumer的Path VMs發出無意義query。
3. Host先把`--since`解析成單一absolute RFC3339 instant，再將同一instant交給journald與Docker；
   不讓Guest與Host各自解讀`today`。
4. Journald使用帶UTC timestamp的output，Docker加入engine timestamp；stream prefix仍保留VM或
   Compose service identity。文件以UTC `Z`為canonical log time，並說明Host status仍可顯示
   Asia/Shanghai local time。

2026-08-13本項已在Infrastructure實作：owned-unit resolver明確涵蓋template
services、Core Consumer及三台VM的Network unit，`--since`由Host timezone解析一次後轉為UTC，
journald與Docker均輸出timestamp。完整`make test`與disposable CPU `make test-containers`通過，後者
實際確認ML focused logs含UTC engine timestamp。後續短暫啟動既有三台VM，在不啟動experiment
services的情況下完成實機spot check：Core Consumer特殊unit、三台Network特殊unit、一般NRF
template unit與預設wildcard均成功查詢並輸出UTC timestamp；Ubuntu 22.04 journald不接受RFC3339
`T...Z`輸入的相容性也在測試中發現並改用等值的`YYYY-MM-DD HH:MM:SS UTC`表示。測試後三台VM
graceful halt。

**B. UE business readiness**

1. `services-status`保留23-unit表，另加六UE summary。
2. 對每個UE讀取systemd `InvocationID`，只查該次invocation journal，避免上一輪成功log污染目前
   判斷。
3. 以UERANSIM既有明確訊息辨識registration與PDU Session；輸出`successful`、`pending`、
   `failed`、`inactive`。若service active但尚無成功證據，必須是`pending`而非成功。
4. Parser採fixture tests，不改UERANSIM或Guest service runtime。

2026-08-13本項已在Infrastructure實作：`services-status`保留23-unit表並加入六UE readiness表，
只以active service的systemd `InvocationID`查詢本次journal；InvocationID或journal查詢失敗會使
command失敗，不會降級成看似正常的`pending`。Fixtures涵蓋inactive、fresh pending、registration-
only、完整成功、reject、procedure failure及失敗後恢復，完整`make test`通過。實際running UE的
current-invocation輸出保留到下一次正常`experiment-start`驗證，不為本項單獨啟動full stack。

**C. Consumer state integrity and per-path callbacks**

1. 將state操作改為Linux file lock保護的跨thread／跨process atomic transaction；lock涵蓋完整
   read-modify-write，state本體仍以temporary file、fsync與atomic replace提交。
2. 保留既有global fields以相容retained state，新增以既有subscription correlation映射到Path A/B
   的callback summary；每路保存request count與last callback UTC time。
3. 明確定義count為callback HTTP requests，不把一個list payload中的items混稱為多次requests。
   未知correlation不得歸入A/B，應分開顯示或記錄warning。
4. `subscriptions-status`輸出A/B表格，包含path、provider NF instance ID、TAC、resource status、
   correlation、callback count、last callback與exact Location；舊state缺少per-path fields時顯示
   `unknown`，不要求migration。
5. 移除吞錯行為；Core unreachable、Consumer CLI失敗與state parse失敗需明確輸出原因並回傳非零。

2026-08-13本項已在Infrastructure實作：Consumer state以process-local reentrant lock加Linux
`flock`保護完整transaction，並維持fsync、atomic replace及directory fsync；create、callback與
delete均以merge update避免覆蓋並行通知。State保留legacy global fields，新增Path A/B request count
與last callback UTC time，未知correlation獨立告警。`subscriptions-status`不再吞錯，並明確標示
resource state是local saved ownership而非remote GET。Regression以threads及processes並行寫入80次、
early callback、DELETE overlap、legacy state、unknown correlation和malformed status驗證，完整
`make test`通過；實際雙Path counters留到最終runtime acceptance驗證。

**D. Aggregate status honesty**

1. `observe`continuous mode保留best-effort特性，但每個失敗section必須印出明確`unavailable`原因；
   不再讓VM section空白。
2. `observe --once`與`experiment-status`在VM顯示running但SSH不可達、Docker daemon不可達或狀態
   parser失敗時回傳非零。VM正常poweroff屬可觀測狀態，Guest sections顯示`not-running`而非誤報
   command failure。
3. `experiment-status`只組合各domain canonical status，不複製UE或Consumer邏輯。

2026-08-13本項已在Infrastructure實作：canonical Guest、WebConsole與Subscription status先辨識
Vagrant machine state，poweroff／not-created顯示`not-running`且不嘗試SSH；running backend查詢失敗、
Docker失敗或parser失敗則保留原因並回傳非零。Continuous `observe`在section failure後繼續下一個
interval，`observe --once`與`experiment-status`則拒絕incomplete snapshot。Fixtures驗證failure
propagation及poweroff語意，完整`make test`通過；另在三台VM實際poweroff、retained ML containers
exited的現場執行`experiment-status`成功且所有Guest-related sections誠實顯示`not-running`。

**E. Current-run FL milestone summary**

1. 在前四項穩定後，再從目前PyMTLF-C container的`StartedAt`與config identity界定current run，
   不掃入上次container start的milestones。
2. 以fixture-tested parser顯示兩monitor scopes、degradation trigger、FL process、preparation、各round、
   final validation、publication、adopted scopes、cutover與post-cutover evaluated accuracy。
3. Summary只讀既有logs並標示`not-seen`／timestamp／identity，不把缺少log推論為component failure，
   也不新增獨立smoke/watcher command。

2026-08-13本項已在Infrastructure實作並整合至既有`ml-status`及aggregate status，沒有新增公開
command。查詢以各PyMTLF container的`StartedAt`限制Docker logs，並要求A/B/C的config-set與hash
identity一致；輸出monitor lifecycle、degradation、process、preparation、A/B local rounds、aggregation、
validation、publication、adoption、cutover、post-cutover evaluated accuracy及明確failure signature。
Fixture tests涵蓋完整里程碑、cutover前後accuracy排序、空logs、精確`StartedAt`傳遞與mixed-config
拒絕；完整`make test`及disposable CPU `make test-containers`通過。另對保留的
`cleanup-convergence-20260813` production containers執行唯讀查詢，成功還原雙scope、A/B rounds
0/1、validation、publication、兩scope adoption、cutover及post-cutover evaluated accuracy。

同日晚間以新的GPU `fl-closure-smoke` config完成live runtime acceptance。Current-run summary從兩筆
active monitor開始，依序顯示degradation、雙scope process、A/B rounds 0/1、validation、publication、
兩scope adoption、cutover及post-cutover evaluated accuracy；六UE、雙Path callback counters與focused
special-unit logs亦通過。正常teardown成功刪除Consumer resources，但internal Model Monitor DELETE
持續503並在210秒後留下兩筆pending IDs；另發現current-run完整log重掃與並行Vagrant logger的效能
問題。完整證據、資源數字與後續邊界見
[Observability Runtime Acceptance](../../reports/5g-nwdaf-infrastructure/observability-runtime-acceptance-2026-08-13.md)。

### Validation and delivery

1. 擴充現有`make test`，加入unit resolver、wildcard expansion、absolute-time normalization、UE current-
   invocation parser、Consumer old-state compatibility、A/B accounting，以及thread/process concurrent
   update tests。
2. 對status section failure建立fixture或stub tests，確認`observe --once`與`experiment-status`不會把
   incomplete snapshot回報為成功；continuous observe仍可在下個interval恢復。
3. 執行shell/Python syntax、完整`make test`與`make test-containers`；container test驗證ML focused
   logging未回歸。
4. Runtime驗證啟動既有三台VM與`fl-closure-smoke`：確認六UE均registration/PDU successful、A/B
   callback counters獨立增加、Consumer/Network focused logs可查、default logs包含特殊units、
   current-run FL summary完成cutover。之後使用已驗證的log-correlated stop並halt VMs。
5. 更新Infrastructure README、operations、commands、troubleshooting與validation；在`testbed-docs`
   新增dated result report。按repository boundary分別review與commit，使用者確認前不push。

建議實作順序為A→B→C→D，review後再做E。A至D修正錯誤與status可信度；E是較高階的便利摘要，
不應阻擋前四項交付。
