# Slice 2：E0–E2b 情境與故障生命週期初始計畫

日期：2026-09-21

狀態：Plan Review Confirmed／Implementation Pending；已確認共用故障設定、短跑參數及執行／事後分析分工，尚未實作或執行真實環境

上層依據：[實作順序與 Slice 安排](../Testbed%20Implementation%20Sequence%20and%20Slices.md)與
[Testbed 實驗就緒盤點](../Testbed%20Experiment%20Readiness%20Inventory.md)。
[Slice 1 計畫](./Slice%201%20Component%20Integration%20and%20Evidence%20Compatibility%20Initial%20Plan.md)
已處理新版 component 設定、E0 接線、原始證據收集及共用 run lifecycle；本文件只討論下一步的 testbed 故障控制與
四情境短程驗收。Go NWDAF／PyMTLF 的協定與訓練行為仍以 owning component source 和 `nwdaf-docs` 為準。

## 1. 目標與已確認邊界

沿用現有四台 VM、`TESTBED`、scenario、generated config、共用 renderer／checker／runner、逐節點原始紀錄及
stop／reset 路徑，讓同一條 runner 流程能執行下列情境。不新增平行的 deployment source 或第二套 runner。

| 情境 | 故障與預期修復方向 | A* 當次啟動 |
| --- | --- | --- |
| E0 | 不注入故障；健康對照 | 否 |
| E1 | 在指定 accepted-round barrier 後停止 A，由 A* 接手 | 是 |
| E2a | 停止 A，讓 A1／A2 嘗試直接接到 Root | 否 |
| E2b | 停止 A，並在 Root 新訂閱前讓 A2 不可用；由 A1 嘗試直接接到 Root | 否 |

E2b 依 `nwdaf-resources` 的情境順序，在共同觸發點停 A、緊接著停 A2；testbed 對每個目標仍使用既有
Guest／Host 受保護停機流程，不直接搬用本機 `kill()`。兩者須分別保存實際停止時間與操作結果。
實際 degraded rounds 與修復結果由執行時序決定，不預設固定值；runner 不在訓練期間判定修復是否達標，
而是保存原始資料，留待事後分析。

本 slice 的交付是四個短程 MNIST 情境在實驗室 testbed 各執行一次，確認啟動、故障、accepted-round 延續、
逐節點收集、final model、停止與 guarded reset。這些短跑不計入 Slice 3 的五 paired seeds 正式實驗。

## 2. 現有流程靜態盤點與 baseline disposition

本節依目前 `5G_NWDAF_Infrastructure` source、其選用的 PyMTLF source，以及
`nwdaf-resources/deployments/hierarchical_fl/` 的本機流程盤點；沒有執行 VM、provider 或訓練。
後者只作協定事件與情境語意參考，不是 testbed 的部署或停機實作來源。

| End-to-end 邊界 | 現況與 disposition | Slice 2 建議 |
| --- | --- | --- |
| Scenario 選擇、生成、檢查 | **Adapted**。`configlib.py` 已允許 `reparent_leaves_to_root` 且 renderer 能傳入 native topology；但非空 `fault` 只接受 `branch-replacement`，`config-check.py` 對任何 fault 都呼叫 replacement target resolver。目前沒有 E2 scenario。 | 在同一 scenario／renderer／checker pipeline 重構共用的觸發時機與停機目標，並一次遷移既有 E1 scenario；Root 修復策略仍由 `topology.onBranchFailure` 擁有，不長期保留兩套 fault 格式或新增人工設定來源。 |
| Build、provision、stage、activation | **Reused without semantic change**。Slice 1 的 selected config、component revision、A* 情境啟動、runtime identity 與 activation 路徑可沿用；E2 不需要新 VM。 | 只用實際生成與啟動的 process inventory 核對 selected runtime；不增加每階段重複的 A* 不存在驗證。 |
| Process start、ready、training request | **Reused without semantic change**。現有 `fl-experiment-run.py` 會走共同啟動、ready、GPU admission 與 Root 訓練請求。 | 四情境共用 `fl-experiment-run`；舊的 branch-replacement 專用 Make alias 不作 E2 入口。 |
| Accepted-round barrier 與 fault trigger | **Adapted**。`PhaseTracker.ready_for_fault()` 已依完成輪數及下一輪 in-flight 狀態判斷時機；目前 contract、runner loop 只有一次 `fault_injected` 與單一 A target。Scenario checker 要求 `acceptedRounds >= normalAcceptedRounds + restoredAcceptedRounds + 1`，`PhaseTracker.finalize()` 也要求至少一輪 degraded accepted round。 | 保留共同 barrier，從 selected scenario 解析明確的停機目標；E2b 在停 A 後緊接停 A2，逐一記錄有效時間。移除為修復輪數預留空間的 schema 限制與 runner 的 degraded／repaired 達標判定，只要求 fault barrier 落在總輪數內。 |
| Round timeout | **Adapted**。`TESTBED.operations.roundTimeoutSeconds=300` 會進入 Root／Branch PyMTLF 的 round 等待與回報期限，也用作 artifact download timeout；歷史 replacement runs 的首個 degraded accepted round 均在停 A 後約 298 秒才出現。`preparationTimeoutSeconds=300` 是另一個階段。 | 將同一 `TESTBED` 來源的 `roundTimeoutSeconds` 調為 60 秒供四情境共用，保留既有 renderer／checker 管線；preparation timeout 暫不因 round 的舊數據而調整。短跑確認健康輪次與故障等待，若 workload 變大再評估。 |
| Guest／Host fail-stop、partial failure | **Adapted**。現有 `fail_stop_primary()` 對單一 Branch 核對 Guest／Host identity，先凍結 Guest，再停止 Host backend、mask 並 kill Guest，回報 `effectiveAt` 與 `hardStoppedAt`；`_faulted_guest` 及 cleanup 只追蹤一個 Guest。 | 延伸同一個受保護的停機操作與復原登記，支援 scenario 選定的多個精確 target；保留先凍結 Guest 的順序、失敗補救、每 target 操作結果及 cleanup ownership。 |
| Root 修復與逐輪判讀 | **Adapted**。目前 `PhaseTracker` 只容許原三 Branch 與 A*，要求 `BRANCH_FAILURE_DETECTED`、`BRANCH_REPLACEMENT_READY`；選用的 PyMTLF source 實際輸出 `EDGE_UNAVAILABLE`、`REPAIR_SELECTION`、`EDGE_CONFIRMED`、`TOPOLOGY_ACCEPTANCE` 等事件，而不是這兩個舊事件。E1 也不能以現有判讀直接驗收。 | Runner 只用 Root status／accepted-round 資訊安排故障與確認訓練完成，保存 component-native JSONL，不以 topology 或 participant 組合判定修復達標；實際關係與貢獻留待事後分析。 |
| 原始證據、final model、official-test、stop／reset | **Common path reused；fault evidence adapted**。runner 已在完成後保存 final model，停止 process 後收集逐節點 JSONL 與 held-out 結果，collection 可補跑；失敗 run 先停機並保留可取得的原始紀錄，成功後才 guarded reset。`check_evidence()` 目前只接受恰好一個 A 的 stop event 與 replacement phase。 | 沿用收集與 exact reset 邊界，只核對 selected fault targets 確實完成停機、訓練及必要檔案已收集；不以 participant／修復結果決定能否保存 final model、held-out 或進行安全 reset。 |

`nwdaf-resources` 的四個本機 profile 可對照 E0／E1／E2a／E2b：它在首輪完成、下一輪等待時先 kill 本機
Branch PyMTLF／Go processes，E2b 隨後 kill 一個 Leaf 的 PyMTLF／Go processes；並從 Root／舊 Branch JSONL
核對修復嘗試、訂閱、accepted topology 及各輪貢獻。這提供事後分析的事件語意參考，但其本機 `kill()`、固定四輪、
單一 Branch stop event 與等待 replacement preparation 的代理控制，不符合 testbed 的 Guest／container lifecycle
和自然 degraded-round 觀測，不得直接複製。若日後需要額外時序控制，應以實際失敗證據另行決策。

## 3. Source 發現與已確認方向

1. **共用 fault 契約要重構**：在既有 scenario 的 fault 資料中表達觸發時機與一或多個 runtime target；
   E1／E2a 停 A，E2b 停 A 與 A2。`onBranchFailure` 仍只決定 Root 的協定修復策略。既有 E1 scenario
   一次遷移至共用格式，不長期維護舊的 `branch-replacement` fault 契約或繞過 `config-check.py`。
   具體欄位、target 解析及 manifest／checker／runner 的對應方案見第 4–7 節。
2. **Root round policy 保持不變**：`nwdaf-resources` 四 profile 共用 Root `fraction_train=1.0`、
   `min_train_nodes=2`、`accept_failures=true`、`min_completion_rate=0.66`，目前 `TESTBED` 與其一致。
   不為 E2 設另一個 accepted-round 門檻。PyMTLF 每輪從實際 active Branch
   與 direct Leaves 選取 participants，completion 分母是當輪 `selectedNfInstanceIds`，因此 degraded 的 B／C、
   E1 修復後的 A*／B／C、E2a 的 A1／A2／B／C、E2b 的 A1／B／C 不必預先寫成不同固定門檻。
   四情境維持相同 Root policy，原始紀錄保留逐輪 selected／successful／failed identities、分母與 accepted
   結果。0.66 允許四選三或三選二的輪次被接受，`accepted=true` 本身不能證明所有預期節點皆有貢獻；
   E2a／E2b 是否修復，事後才由實際 topology 與 A1／A2 的參與判斷，不在 runner 設另一個門檻。
3. **E2b 沿用參考情境的觸發與目標順序**：在 selected accepted-round barrier 後先停 A、緊接停 A2；
   分別保存 Guest、container、`effectiveAt`、`hardStoppedAt` 與失敗結果，供事後核對 A2 在 Root 新訂閱前
   是否已不可用。
   `nwdaf-resources` 的觸發是在首輪後；testbed 的實際輪數仍由 selected scenario 決定。
   不移植本機 `kill()` 或 replacement preparation gate。若真實 VM 時序搶先，保留該次結果，再決定是否調整控制。
4. **事後判讀以原始事件串聯**：Root 的修復選擇、`MODEL_TRAINING_OPERATION`、`EDGE_CONFIRMED`、
   `TOPOLOGY_ACCEPTANCE.realizedTopology`、`ROUND_AGGREGATION` 與 A Branch／Leaf 逐節點 JSONL，分別證明
   requested、realized、accepted 關係、receiver-assigned `subscriptionId`、`mlCorreId`、首次故障後貢獻與
   B／C 持續參與。這些是事後分析的輸入，不是 runner 的 repair gate。E2b 的 A2 未接上不能只靠「找不到
   confirmed event」推斷，須結合停機、Root 修復選擇、可得的訂閱結果、最終 accepted topology 與 accepted
   rounds。A2 在 discovery 哪一步失敗不是本 slice 的必收證據，不為此要求 component 新事件。
5. **Round timeout 依歷史 run 縮短**：`5G_NWDAF_Infrastructure/runs/history/protocol-hierarchical/`
   的 `run.json` 顯示，MNIST 4 local epochs 的相鄰正常 accepted rounds 約 8–9 秒；
   CIFAR-10 5 local epochs 約 11–12 秒。三次歷史 replacement runs（`mnist-formal-replacement-20260913-b`、
   `cifar10-formal-replacement-20260914-a`、`cifar10-skew-replacement-20260914-a`）停 A 到首個 degraded
   accepted round 約 298 秒，幾乎耗盡目前 300 秒的 round timeout。`nwdaf-resources` 本機流程設 60 秒；
   本 slice 將 testbed `roundTimeoutSeconds` 設 60 秒，正常輪次仍有歷史觀測值數倍的空間，故障等待預期
   降至約一分鐘。這是短跑要驗證的預期，不是已測得的新結果；delay extension 可另行延長等待。
   `preparationTimeoutSeconds=300` 與觀測到的 round 等待不同，暫不調整。
6. **測試只守住會失效的共同邊界**：在現有 owning suite 聚焦 scenario target 解析、多 target 停機／cleanup、
   accepted-round barrier 與收集 checkpoint，再以四個短程 MNIST run 取得 real boundary evidence；不把
   profile 名稱、固定 degraded 輪數或修復成功寫成 runner invariant。短跑仍須包含 final model、held-out、
   原始紀錄、停止與 selected exact reset；成功的 E0 不能替 E1／E2 證明 fault path。

目前沒有發現需要新增 VM、service、人工 config source 或改 PyMTLF contract 的直接依據。第 4–7 節是
根據本節方向形成的具體方案；共用 fault 設定與線上／離線分工已由使用者確認，但尚未實作或通過 runtime 驗收。

## 4. 具體方案：單一 scenario 與 target truth

沿用 scenario schema version 2、現有 `fault`／`observation` 位置和 generated manifest，不新增人工來源。
已確認的共用 fault 形狀如下；`stopNodes` 使用 `TESTBED.analytics` 中的 NWDAF unit 名稱，順序即操作順序：

```yaml
topology:
  onBranchFailure: reparent_leaves_to_root # E1 改為 replace_branch
training:
  acceptedRounds: 8
  localEpochs: 4
fault:
  normalAcceptedRounds: 2
  stopNodes:
    - nwdaf-branch-a-primary
    - nwdaf-leaf-a2                 # 只在 E2b 出現；E1／E2a 只停第一項
observation:
  pollIntervalMilliseconds: 250
  heartbeatSeconds: 30
```

E0 沒有 `fault`／`observation`；`onBranchFailure=replace_branch` 只是 native topology 必填策略，不會觸發停機。
`normalAcceptedRounds` 只決定 accepted-round barrier：完成兩輪後，在下一輪 in-flight 時注入故障；
`training.acceptedRounds=8` 決定總共要完成的 accepted rounds。Fault scenario 只要求
`0 < normalAcceptedRounds < acceptedRounds`，不規定故障後哪一輪必須修復，也不設 `repairedAcceptedRounds`。
`observation` 保持現有 poll／heartbeat 契約，不因 fault 格式更動。短程 MNIST 的 4 local epochs、8 accepted
rounds、2 輪後注入故障是本次已確認的接線測試配置，不是所有 scenario 的通用合法性限制。

解析 selected `stopNodes` 時，第一項必須是所屬 Branch group 當次最高 priority、已啟動的 Branch；其餘項目若有，
必須是同一 group 中當次已啟動的 Leaves，且不得重複。這是為了在開始操作前確定每個受保護停機目標與
同一故障 group，並非以 E1／E2b 名稱或固定 node set 限制通用 schema。從 `TESTBED` 的 node definition／placement／
backend mapping 一次解析每項的 `nfInstanceId`、Guest machine／unit、Host PyMTLF service；由第一項解析 group
及候選 Branches。`topology.onBranchFailure` 仍由 Root 使用來選修復策略，testbed 不在 scenario 複製
「應修復到哪些節點」的期望。A* 是否列入當次 active Guest／Host inventory，由 selected group＋修復策略
導出；E0／E2 不啟動 A*。不把 `mode`、`branchGroup` 或預期 A* ID 當第二份人工設定。

共用資料流為 scenario → `configlib` 解析與 selected runtime inventory → `config-render.py` 產生 native config／
manifest → `config-check.py` 核對 selected scenario 與 generated artifacts → `FLExperimentContract.build` 建立
runner 停機目標與 barrier → `fl-experiment-run.py` 操作與 checkpoint → `fl_experiment.py` 核對執行及收集
結果。現有五份 E1 scenario 的舊 `mode`／`branchGroup`／`restoredAcceptedRounds` 一次遷移；`resolve_branch_replacement()`
的責任由共用 target resolver 取代，checker 不再對每個 fault 強制呼叫 replacement resolver。`run.json` 的
selected scenario 保留輸入，另存從它解析出的停機 target identity；不建立雙格式相容分支。`kind` 仍不參與此決策。

## 5. 具體方案：故障操作、觀測與失敗復原

1. **共用觸發與停機**：四情境仍走 `fl-experiment-run`。當 Root 已完成指定正常輪數且下一輪處於
   `ROUND_DISPATCH`／`ROUND_WAITING`，依 `stopNodes` 順序逐一呼叫同一 guarded fail-stop primitive；E2b 先 A
   後 A2，中間不增加 sleep、preparation gate 或第二個 runner。每項沿用 selected／active config、Guest service／PID、
   Host Compose service／container 與 provider Host-context guard；先凍結 Guest，再凍結 Host backend，之後
   runtime-mask 並 kill Guest、kill Host backend。每項記 `effectiveAt`（Guest 已凍結）與 `hardStoppedAt`
   （雙 owner 停止確認），連同實際 identity／結果寫入 `events.jsonl` 和 `run.json`。現有單數
   `BRANCH_PROCESS_STOPPED`／`primaryStoppedAt` checkpoint 改為每項一筆 controller
   `NODE_PROCESS_STOPPED` event，以及 `run.json.faultStops` 清單；每筆以 selected `nfInstanceId`、Guest unit、
   Host service 對應同一 target，保存原有 Guest／container 停機證據與兩個時間。只有該 target 的雙 owner
   停止確認後才寫 confirmed stop；第一項成功、第二項失敗時保留第一筆與失敗紀錄，不產生第二筆假成功。
2. **停機中的部分失敗**：每個成功凍結或 mask 的 Guest 都登記到 cleanup ownership；若後續 target 失敗，
   停止注入，不繼續跑成「成功 E2b」，並依現有選定 runtime 的 stop 路徑及每 target unmask／resume 補救。
   不能把第一項已停、第二項未停的 run 標成完整故障；補救失敗要留在 run failure 記錄與現場狀態中，
   不自動擴大清理其他 VM／container。`stop_all()` 和 collect-only 必須能處理多個已受影響 target，
   而非目前的單一 `_faulted_guest`。
3. **時間序與 E2b 競態**：provider 操作期間 Root JSONL 仍可能前進；runner 要以來源時間合併全部 pending
   Root records 與每個 stop event，再更新故障 barrier 與記錄，不以讀取時間宣稱先後。A2 的實際
   `effectiveAt`、Root 修復選擇及任何新訂閱的 `startedAt` 都按原樣保存；若沒有對 A2 發出訂閱，
   不虛構一個訂閱時間。A2 是否真的在 Root 新訂閱前停下是事後根據時間線判讀的實驗結果，
   不是 runner 的成功條件。若時序不符，保留該次資料並回報；不反推時間、加入等待代理或重寫故障設定。
   `roundTimeoutSeconds=60` 只縮短根據舊 300 秒觀察到的故障等待；若正常輪誤逾時，以短跑證據再決策。
4. **完成與失敗資料保留**：只要 Root 達到 selected `acceptedRounds` 且 `COMPLETE`，不論是否修復，
   都走同一條 final model、held-out、逐節點 JSONL 與 controller events 收集路徑；資料保存成功後再走
   現有 guarded exact reset。`run.json` 的執行狀態只表示訓練、故障操作與收集流程是否完成，
   不新增 `not-recovered` runtime status／outcome，也不因未出現 degraded／repaired round 而拒絕 final model。
   訓練未完成或停機失敗時沿用先 stop、再盡力補收可得原始資料／diagnostics 的路徑，不做假成功 reset；
   collection 中斷則保留可重試 checkpoint。現有 `PhaseTracker.finalize()` 的修復門檻必須移出執行／收集路徑。

## 6. 執行中檢查與事後分析的邊界

Runner／`check_evidence()` 只檢查執行契約：selected scenario／runtime identity 一致、故障 barrier 已達、
每個 selected target 的受保護停機操作與 checkpoint 完整、Root 最終 `COMPLETE` 且完成 selected
`acceptedRounds`、final model／held-out／逐節點 JSONL 與 controller events 已保存，以及 stop／reset
符合原有安全邊界。這些檢查保護操作和資料完整性，不能因少了修復證據而拒絕一個已完成且已收集的 run。
E0 沒有 fault events 也是正常結果；故障操作失敗、訓練未完成或檔案缺失則仍分別記錄為執行／收集問題。

`PhaseTracker` 只保留觸發 barrier 與完成輪數所需的資訊；現有 `BRANCH_FAILURE_DETECTED`／
`BRANCH_REPLACEMENT_READY`、固定 normal／degraded／restored cohort、至少一輪 degraded，以及
`restoredAcceptedRounds` 的要求均不得留在 runner 成功條件中。`ROUND_AGGREGATION`、
`MODEL_EVALUATION`、`EDGE_UNAVAILABLE`、`REPAIR_SELECTION`、`MODEL_TRAINING_OPERATION`、
`EDGE_CONFIRMED`、`TOPOLOGY_ACCEPTANCE` 等 component-native records 按原樣保存；
不在執行中把某種 selected／successful participant、accepted topology 或修復時間當作必過門檻。

事後才從 `run.json`、`events.jsonl`、Root／Branch／Leaf 原始 JSONL、final model 與 held-out 結果判讀：
故障後有幾輪只由 B／C 貢獻、E1 的 A* 是否接手、E2a 的 A1／A2 是否直掛、E2b 的 A2 是否及時停下、
實際 accepted topology／participant、正式 `subscriptionId`（需連同接收端身分）與 `mlCorreId`，以及
是否 `not recovered`。`TOPOLOGY_ACCEPTANCE.accepted` 不等於 `ROUND_AGGREGATION.accepted`；
Root 發出修復要求也不等於已形成新訂閱或成功貢獻。Slice 2 短跑後可人工檢視這些現象並報告，
但不把判讀規則寫進 runner；正式的跨 seed 離線分析留 Slice 3。四情境仍使用同一 run directory，
不新增專用 renderer、report 或平行結果庫。

## 7. 實作順序、聚焦驗證與停止點

1. 先調整共用 scenario parser／resolver／inventory／renderer／checker，遷移 E1 scenarios，加入 E0／E2a／E2b
   的短程 MNIST selected scenarios。四情境採 4 local epochs、8 accepted rounds；E1／E2a／E2b 在
   2 輪完成後注入故障，E0 不注入。將同一 `TESTBED` 的 `roundTimeoutSeconds` 調至 60，保留
   `preparationTimeoutSeconds=300`。確認 native topology、active A* selection 與 manifest 從 selected input 一致
   產生；不改 component contract、VM、service 或 dataset source。
2. 再擴充共用 runner 的 target contract、逐 target fail-stop／checkpoint／cleanup 與 source-time event merge；
   保持 Host guard、selected identity、精確停機與 reset scope。將 tracker／checker 中的修復達標判定移出
   執行路徑，讓所有已完成訓練且操作／收集完整的 run 都能保存 final model 和原始資料。
3. 在既有 `tests/runtime-inventory.py`、`tests/fl-experiment.py` 等 owning suite 只修正與新契約直接衝突的
   行為檢查：從 selected scenario 解析 target 與 A*，多 target 停機及 partial cleanup，來源時間順序，
   以及 Root `COMPLETE` 後即使沒有修復仍可完成收集。測試的 target、輪數與期望值取自 fixture／selected
   input；不建立專以 E0／E1／E2 名稱、固定 repaired rounds 或某個 log 字串為真理的永久測試，
   也不執行 real provider。
4. 經初次 review、修正與 focused checks 後，在 approved Host context 每次重新核對容量／selected runtime，
   依 E0、E1、E2a、E2b 先後各跑一個短程 MNIST run；每次保留結果並完成 stop／收集／exact reset 後才切情境。
   短跑的執行檢查是 A* 只在 E1 啟動、指定節點依序完成停機、正常輪次對 60 秒 timeout 有餘裕，
   並取得 8 個 accepted rounds、final model／held-out／逐節點原檔及 cleanup。E2b 實際停機與新訂閱
   的先後、故障後貢獻及修復情形在收集後檢視並報告，不作 runner 成功條件。

本輪只是計畫細化，沒有實作或真實環境驗收。共用 schema、事後才分析修復，以及短跑參數已由使用者
確認；計畫已通過 review，下一步是依本計畫實作與驗證。若實作盤點發現需改 component contract、新增 config source／VM／service／dependency，或
擴大 destructive scope、弱化驗收，先更新計畫並再請使用者決策。`preparationTimeoutSeconds`、正式五 paired
seeds 與離線統計均留在各自的後續判斷／Slice 3，不在此輪預設解法。
