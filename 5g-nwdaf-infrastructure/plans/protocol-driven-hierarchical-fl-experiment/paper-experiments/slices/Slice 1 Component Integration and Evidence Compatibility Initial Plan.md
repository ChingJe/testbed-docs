# Slice 1：Component 整合與證據相容性計畫

日期：2026-09-21

狀態：Review Confirmed／Commit Pending；component 整合與完整 run lifecycle 優化已實作並完成短程 E0 驗證，待提交

上層依據：[實作順序與 Slice 安排](../Testbed%20Implementation%20Sequence%20and%20Slices.md)與
[Testbed 實驗就緒盤點](../Testbed%20Experiment%20Readiness%20Inventory.md)。本 slice 處理盤點 3.1、3.2、3.4、3.5
的 testbed 整合；故障注入與四情境真實驗收屬 Slice 2，五 paired seeds 與離線統計屬 Slice 3。

## 1. 目標與既定決策

以現有四 VM、`TESTBED`、scenario、generated config、共用 renderer／checker／runner 及 stop／reset 路徑，接通新版
Go NWDAF／PyMTLF。E0／E1／E2 使用同一批 VM；A* 可安裝於 `path-a`，但只在 E1 啟動其 Guest NWDAF 和 Host PyMTLF，
E0／E2 不啟動。情境啟動完成時對 A* 實際未運行狀態觀測一次，不建立跨階段重複的負面證明。

E0 不注入故障；其 native topology 仍需填 `on_branch_failure: replace_branch`，但健康 run 不會走到修復分支。
E1 也用 `replace_branch`；E2a／E2b 用 `reparent_leaves_to_root`。故障目標和 Root 修復策略是兩個不同責任，
不能繼續讓舊 `fault.mode: branch-replacement` 同時代表兩者。Slice 1 可生成／解析 E2 topology，不等於 E2 runner 已能注入故障。

已確認在現有 scenario 內明確記錄 `topology.onBranchFailure`：E0／E1 為 `replace_branch`，E2a／E2b 為
`reparent_leaves_to_root`；E0 不設 `fault`。此欄位只決定 Root 故障時的 topology 行為，
`fault` 只記錄注入目標與時機。`TESTBED.analytics.protocolTopology` 仍擁有節點、分組與 placement；
不從 scenario 名稱、`kind` 或 `fault.mode` 隱含推斷 Root 策略，也不新增另一個人工設定來源。

逐節點 PyMTLF `observations.jsonl` 是訓練／協定原始證據；Root 檔在 run 中供 controller 即時判讀，所有實際運行節點的
原檔在 process 停止後、reset 前從既有 volume 保存。文字 log 只作 diagnostics。保留 final model 和 official-test
evaluation 的既有獨立來源，不把它們混成 JSONL。短程無故障 run 僅驗證接線，不列入正式五 seed 實驗。

2026-09-21 的優化前短程 E0 run 顯示 `startedAt` 到 `RUNTIME_READY` 約 4 分 48 秒，而兩輪訓練請求到 final artifact 收集約
5 秒；這只定位優化範圍，沒有各內部操作的獨立計時，不能把延遲歸因於某一項檢查或連線。使用者已將現有
Guest 反覆短連線、重複 ready 查詢與可獨立操作的串行化納入本 slice 處理；不為追求速度降低必要安全或
實驗驗收門檻。

## 2. Source-level 現況與需要適配的邊界

以下是實作前 source 的直接盤點，用於對照本 slice 的改動，不是已驗證 runtime 的宣稱。

| 邊界／owner | 實作前行為 | Slice 1 處置 |
| --- | --- | --- |
| `Vagrantfile`／四 VM provisioning | 從 `TESTBED` placement 安裝所有 Guest units，並依 `components.lock.yaml` 記錄 component revision | reused；不改 VM 數或 placement，A* 保留可安裝但依 scenario 不啟動 |
| `configlib.image_scenario_contract` | 無 fault 或唯一 `branch-replacement` fault；E2 不能通過 | adapted；在既有 scenario 驗證明確的 `topology.onBranchFailure`，與 `fault` 分開；E2 具體故障注入契約留 Slice 2 |
| `configlib.protocol_topology`、`config-render` | 產生舊 `admission: complete_required`，缺少新版必填 `on_branch_failure`；Root artifact origins 只有 Branch／ADRF | adapted；同一 renderer 產生 native E0／E1／E2 topology，E2 允許 Root 存取可能直掛的 Leaf artifact |
| `render_protocol_nwdaf` 的既有 template 路徑 | 新版 NWDAF 已移除 repository 內的 `config/` templates；原路徑讀取會在 render 時失敗 | adapted；沿用 testbed 已有的 `config/default/nwdafcfg-*.yaml`，與既有 static renderer 共用來源，不新增人工設定來源；仍須以新版 NWDAF parser／實際啟動確認相容性 |
| `expected_runtime_inventory`、manifest、`config-check` | 全部 11 個 NWDAF／PyMTLF 被視為同一 selected runtime；容量、NRF IDs、Compose、reset 由它衍生；checker 寫死 11 | adapted；區分完整安裝／保留範圍與當次應啟動 process；所有 consumer 使用同一 generated selection |
| `experiment-start`、`services-start`、`ml-start`、registration／status | Guest 與 Host 啟動皆取 manifest 全清單；NRF check 的 IDs 來自 `resetScope` | adapted；只啟動 active process；registration 和 ready 判斷使用 active identities，status 仍揭露 unexpected actual runtime |
| `vssh`、Guest upload、啟動／ready 路徑 | 每個 `vssh` 都另執行 `vagrant ssh -c`；工具與設定的 upload、Guest unit start／poll、最終 snapshot 分別呼叫；Compose 已能同時啟動 selected containers | adapted；在共用 Host transport 內複用每台 VM 的連線，依依賴屏障並行獨立 VM／Host 操作，合併同邊界的重複查詢；不為此新增 deployment source 或平行 lifecycle |
| `experiment-stop`、`services-stop`、`ml-stop`、`experiment-reset` | stop 停整個 ML project，但 Guest stop 取 selected 清單；reset 對 selected containers／volumes 作 exact comparison，拒絕 unexpected residue | adapted；跨 E1→E0／E2 的 retained A* state 不得被誤判成可忽略，也不能靠擴大破壞範圍解決；保留 wrong-config 與 exact reset 防護 |
| `fl-experiment-run.py`／`fl_experiment.py` | Root JSONL live reader 與 final model volume copy 已存在；`PhaseTracker`、resource log parser 和 evidence checker 仍依舊事件、文字 log、固定 Branch cohort | adapted；共用事件判讀改讀新 JSON 欄位與實際 direct children；E1／E2 fault outcome 完整驗收在 Slice 2 |
| `collect-only` checkpoint | 只接受 `collection-pending`／`collection-failed`，並要求 terminal `COMPLETE` 和 final model 已保存 | adapted；完成 run 的 collect-only 沿用；中途失敗 run 另走最小的原始紀錄補收，不要求不存在的 final model |

若只改 start controller 而不改這些衍生 consumer，會出現「設定選了不啟動 A*，manifest 卻仍要求它健康、
NRF 仍等它註冊、reset 又把舊 A* volume 當 unexpected」的矛盾。這是單一配置管線內的適配，不是額外 runner。

## 3. End-to-end 實作順序與 baseline disposition

1. **版本與輸入選擇（adapted）**：確認 parent gitlinks、nested NWDAF／PyMTLF commits、`components.lock.yaml`，
   並遵守現有 preflight 對 selected component worktree 的 clean 要求；若 component 版本未提交或不相符，
   先停下，不以 dirty nested HEAD 當可重建版本。沿用既有 `TESTBED` 作
   topology／placement truth，scenario 作當次 condition、training、fault 與 observation truth。
2. **生成／檢查（adapted）**：在現有 `config-render.py`／`configlib.py`／`config-check.py` 內生成 native topology、各節點
   config、Compose、manifest；將 `scenario.topology.onBranchFailure` 原樣轉為 native `on_branch_failure`，
   並保存 selected scenario 的值供 checker 核對，不從 fault 推導。依 scenario 導出 active process selection，
   保留完整安裝／reset owner。E2 Root artifact
   allowlist 包含 direct Leaves，Branch／ADRF 原路徑不退化。應有數量從 selected input 推導，不寫成跨情境硬規則。
3. **建置／stage／activate（adapted）**：沿用 Guest binary 與 Host image build、provisioning、
   config activation 和 identity 檢查；只調整共用傳輸與獨立 VM 工作的排程，不新增 VM、image 類型或平行部署
   流程。生成設定須由選定 PyMTLF native parser 接受。所有 VM 完成 stage 後才能 activate，部分 Guest
   activation 失敗時沿用既有 fail-closed recovery，不宣稱 selected config 已全域生效。
4. **啟動／ready／訓練（adapted）**：四 VM 照常運行；僅啟動 active Guest／Host process。啟動後核對 active
   Guest、Host、NRF、component／image identity 與容量，並對未選 A* 的實際停止狀態觀測一次；不要求它沒有
   provisioned unit、config、stopped container 或 volume。使用現有 request／status 與 Root JSONL poll 進行
   一次短程 E0 訓練；E1／E2 多故障控制不在這一步驗收。
5. **觀測／停止／收集（adapted）**：Root live reader 按新版 `MODEL_EVALUATION`、`ROUND_AGGREGATION` 和
   `MODEL_ARTIFACT_SAVED` 判讀 accepted rounds、`loss`／`accuracy` 與 final artifact；不因其他合法
   structured records 出現就直接失敗。先 stop process，再依每個 active／曾啟動節點的 native config 和
   Compose mount 找到對應 volume、`mlCorreId` 原檔；包含已 fail-stop 的 A／A2，但不要求從未啟動的 A* 有檔。
   以既有 final-model read-only volume-copy 方法延伸收集，不改原始 JSONL bytes；controller events／run
   metadata 仍由 runner 保存。
6. **評估／reset／復原（adapted）**：final model 與 official-test evaluation 保持可獨立重試；必要原檔保存成功
   後才允許 guarded reset 清空 volume。collection 不完整則保留 checkpoint 與現場資料，不清成成功 run。
   訓練中途失敗時先盡力停止 process，再盡力收集已有原檔；收集錯誤只記入 failed run，不阻止停止或
   強制 reset。情境切換時先明確 stop／guarded reset；不因 A* 本次未啟動而遺漏先前 E1 留下的 state。

已安裝與本次啟動是不同概念。已確認沿用現有 generated manifest 的欄位分工：`runtime.guestServices`、
`runtime.hostContainers` 和 `runtime.mlVolumes` 表示當次 active process／volume；`runtime.resetScope`
保留從 `TESTBED` 可重建的完整 retained／cleanup 範圍。E0／E2 的前者不含 A*，後者仍含 A*；E1 兩者均含 A*。
`runtime.nwdafs` 繼續描述完整邏輯節點定義；NRF ready 預期 IDs 從 active Guest services 對應的節點推導，
不得再直接拿 `resetScope.nrf.nfInstanceIds` 當啟動完成清單。現有
`load_runtime_manifest` 要求 active 與 reset 清單完全相等的限制需據此適配；status／reset 仍對完整實際
project inventory 作 exact comparison，不能只看 active subset。所有清單由 `TESTBED`＋scenario 生成，
不增加人工 selector，也不因 A* 未啟動就自動清理或切換情境。

## 4. Structured evidence 與 collection contract

- PyMTLF 在各節點的 `experiment_recording.directory/<mlCorreId>/observations.jsonl` 追加紀錄；Root 啟動 procedure
  時建立目錄，其他節點可能在首筆事件後才有檔。收集名單依 selected／actual process 和記錄存在性解釋，
  不能用「所有安裝節點都必須有 JSONL」當規則。
- Root `MODEL_EVALUATION` 有 `evaluationStage`、`loss`、`accuracy`、`sampleCount`；`ROUND_AGGREGATION`
  有 `roundInd`、`accepted` 和 selected／successful／failed NF IDs。`MODEL_ARTIFACT_SAVED` 只有
  `artifactFile` 與 `roundInd`；artifact identity／大小仍依 terminal status、既有 final model copy 與實際
  檔案判斷，不能從 JSONL 虛構 digest 欄位。
- `CANDIDATE_SELECTION`、`EDGE_CONFIRMED`、`EDGE_UNAVAILABLE`、`REPAIR_SELECTION`、`TOPOLOGY_ACCEPTANCE`
  是拓樸事件；各端 `MODEL_TRAINING_OPERATION` 記錄 operation、direction、outcome、可取得的 peer／
  `subscriptionId`／message。正式 subscription resource 依接收端與其 `subscriptionId` 對回；JSONL
  未保存完整 `resourceLocation` URL，不再要求從 Docker text log 擷取 URL 才能成立協定證據。
- `events.jsonl` 可保留 runner 已寫入的 controller／live Root 衍生事件；每個 node 原始 JSONL 另存，
  不以轉換後的 events 檔替代。Slice 1 建立保存 layout 與最小 metadata；跨節點統計摘要和繪圖屬 Slice 3。
- 現有 `collect-only` 只覆蓋 terminal `COMPLETE` 且 final model 已保存。Slice 1 把「已完成訓練後補收」與
  「訓練中途失敗後保留部分證據」分開處理。後者保留 `failed` 狀態：若已知 `mlCorreId`，process 停止後
  盡力複製當時存在的逐節點原檔；若收集本身失敗，允許同一 run identity 再補收，不啟動訓練。只有定位
  正確 run、selected config 與 volume owner 所必需的檢查；不以 terminal `COMPLETE`、final model、
  全部節點都有檔或完整輪數作為部分原檔收集門檻。若失敗早於取得 `mlCorreId`，只保留可得的 controller
  events／diagnostics；volume 已 reset 則明確報資料不可恢復。任何部分收集都不把 run 改判成功。

## 5. 聚焦驗證與完成條件

不為情境名稱、檔案集合或 source 文字形狀另建永久測試。優先沿用 owning tests，以最少直接證據核對：

1. selected scenario 產生 E0／E1／E2a／E2b 所需 topology／active inventory，native PyMTLF parser 可接受；
   E0 與 E2 不啟動 A* 的設定不會讓 registration、Compose、容量或 reset 各自採用不同真相。
2. 共用 lifecycle 在 A* active／inactive 兩種清單下仍識別 unexpected actual runtime；重用 VM／保留舊
   container／volume 的切換不跳過 wrong-config 與 exact reset fence。
3. 以新版 PyMTLF 事件 fixture 或受控輸入檢查 Root round／metric／terminal 解讀、完成 run 的收集重試，
   以及一次中途失敗後從 stopped volume 補收已有原檔；確認它仍標為 `failed`，不要求不存在的節點檔案。
   只測直接失效結果，不追加情境專屬或純欄位形狀測試。
4. 在 approved Host context 做一個短程無故障 run，核對 selected component revision、Guest／Host active
   identity、逐輪 evaluation、final model、official-test、各實際節點原始 JSONL、stop 與 guarded reset。
   Real provider 不在 sandbox 內調用；執行前重做當日容量與安全 preflight。
5. 完整 run lifecycle 優化後，先以不啟動 real provider 的受控檢查核對共用傳輸、連線失效、跨 VM 啟用部分失敗
   的聚合與回滾、訓練完成後 stop／收集／評估的依賴，以及 Host volume 操作失敗仍不會提前 reset；再於 approved
   Host context 重跑一次短程 E0，核對單次連線複用、active／retained identity、原始證據、held-out evaluation、
   stop 與 exact reset。失敗 run／collect-only 以聚焦受控檢查確認仍可停機、保留部分證據及補收；E1／E2 的
   真實故障與修復驗收留在 Slice 2。不以大量情境組合或效能基準測試取代上述直接驗證，也不將原短跑的成功
   當成優化後已驗收。

Slice 1 完成只代表新版設定、E0 接線、原始證據收集及本次 lifecycle 優化可用；不證明 E1／E2 修復、故障清理、多 target fail-stop、
五 seed 配對或正式論文結果。若 E2 native config 可解析但 runner 仍拒絕 E2 fault，這是 Slice 2 handoff，
不得誤報為 Slice 1 real-environment 驗收成功。

## 6. 已確認的實作邊界

1. **已確認：Manifest 的 active／reset 分工**：沿用上述既有欄位分別表達當次啟動和完整清理範圍；
   E0／E2 的 A* 不啟動，但可被識別為先前情境保留的資源。ready／registration／容量只要求 active process，
   stop／reset 不隱藏 unexpected actual runtime，也不改變 explicit scenario-switching 與 guarded reset。
2. **已確認：失敗 run 的最小補收**：沒有 terminal model 時，僅盡力保存已有 JSONL／controller events；
   收集失敗可在同一 run identity 補收，不重訓、不阻擋停止、不以完整性驗證封鎖部分資料，也不把該
   run 轉成 successful。補收不能跳過必要的 run／config／volume owner 定位與既有 reset 安全邊界。
3. **已確認：scenario 內的 Root 策略**：以 `topology.onBranchFailure` 明確選兩種 native 支援值；
   `fault` 獨立描述故障注入，E0 沒有 fault。Slice 1 更新既有 scenario／renderer／checker 並核對兩種
   native 值；E2a／E2b 的故障 target、時序與執行驗收仍屬 Slice 2。

若 source 或聚焦驗證顯示上述做法需要改變已確認 component contract、擴大 destructive scope、新增 VM／service／
external dependency，先回報 contradiction 並更新計畫取得決策，不在 Slice 1 偷加 workaround。Production code 的
修改已依使用者確認開始；正式實驗執行仍不在本 slice 範圍內。

## 7. 完整 experiment-run lifecycle 優化：已實作並驗證

優化前的 `fl-experiment-run.py` 呼叫 `experiment-start.sh`，再由共用 `services-start.sh`、`ml-start.sh` 啟動實際
runtime。Guest 命令經 `lib.sh` 的 `vssh` 逐次執行 `vagrant ssh -c`；工具、設定與 reset helper 的
`vagrant upload` 另開傳輸。四台 VM 的工具同步、provisioning 查詢、config stage／activate、clock check，
以及各 Guest unit 的 start／poll 多為依序呼叫。Host Compose 已一次啟動 selected containers，但
`ml-start` 的 GPU probe／`ml-status` 與 runner 的 ready snapshot 有重疊讀取。`RUNTIME_READY` 後，runner
提交訓練請求並以 HTTP status 和 Root JSONL 輪詢；故障情境還會在指定 barrier 進入既有 fail-stop。正常完成後
先複製 final model，再呼叫 `experiment-stop.sh`；該 stop 在有訂閱資源的情境先執行訂閱清理及既有等待，再
停止 Host ML project、依序停止 Guest units。接著逐一檢查並複製每個 active 節點 volume 的原始 JSONL，執行
held-out evaluation，最後以 `experiment-reset.sh apply` 和 `verify` 兩次完整檢查與清理收尾。這是同一條現有
lifecycle 的成本盤點，不建立第二套 runner。短程 E0 的 final artifact 到 evaluation 約 115 秒，evaluation
到 cleanup 約 83 秒；現有 milestone 不足以把這些時間分配給各子步驟。

實作按下列依賴與責任進行：

1. **共用連線（adapted）**：在 approved Host context、完成 provider／process inventory preflight 後，由
   runner 為 selected VM 取得 Vagrant 提供的 SSH 設定，建立每台 VM 一條僅限本次 run 的 OpenSSH
   multiplexing master。現有 Guest 命令與 upload 由 `lib.sh` 共用受保護的傳輸入口使用相同連線；涵蓋
   start、stop、reset 及其失敗路徑，不能只改 `vssh` 而遺漏 `vagrant upload`。各次遠端命令仍分別回報結果，
   不是讓 VM 自行無條件跑完整套流程。獨立呼叫現有 Make lifecycle target 仍沿用相同安全入口，
   不依賴 runner 專用的平行實作。暫時 SSH 設定與 control socket 由 Host runner 擁有，放在非共享且僅
   owner 可存取的位置；成功、失敗與中斷時均關閉／清理。master 中斷時明確失敗，不隱式退回新直連；
   所有 provider 相關呼叫及 Guest 傳輸仍先通過既有 Host-context guard。正常 run 的目標是每台 VM 一條
   實體 SSH 連線，不宣稱連線失效時仍可維持此數量。
2. **VM 平行化（adapted）**：各 VM 的工具同步、provisioning 讀取與 config staging 可在 VM 之間並行；
   保留「全部 stage 完成 → activate → 全部 active identity 核對」的屏障，部分 activate 失敗仍依既有
   rollback／fail-closed 規則處理；同一 VM 的 mutation 仍維持順序與既有 machine lock。同一 VM 內把可
   共用遠端上下文的命令與 unit 狀態讀取合併。核心服務
   依實際相依順序啟動；核心前置條件成立後，互不相依的 `path-a`／`path-b`／`path-c` 工作可並行，
   但不預設單一路徑內 Branch／Leaf 的相依性可取消。Host 只有在所有必要 Guest 結果成功後才宣稱 ready。
3. **訓練、故障與停止（adapted）**：保留 request／status 與 Root JSONL 的對時、accepted-round barrier 和
   terminal checkpoint；輪詢不是 VM SSH 工作，不因共用 Guest 連線而自動變快。若輪詢成本確實顯著，才在
   既有 reader／poll 邊界減少重複 process 啟動，不新增常駐 collector。既有 fault fail-stop 的 Guest／Host
   凍結、停止和時間戳具有先後語意，不能為了並行而改序；本 slice 只確保新 transport 不破壞既有路徑，
   E1／E2 真實驗收仍屬 Slice 2。正常完成先將 final model 安全複製到 run，再停止 process；保留有訂閱時
   的 cleanup 與等待語意、Host ML stop 後才停 Guest 的既有順序。同一 VM 的 unit 仍按相依順序停止；
   `path-a`／`path-b`／`path-c` 互不相依的 Guest stop 可並行，依賴它們的 core stop 仍在屏障後執行，
   並彙總任何失敗；避免 stop wrapper 重做上游已取得的相同
   selected／actual 狀態查詢，但獨立執行 stop 仍須自行核對必要 precondition 和完成狀態。
4. **收集、評估與 reset（adapted）**：process 確認停止後，各 active 節點從各自 volume 複製原始 JSONL，
   不再用逐節點串行 `docker run` 存在性查詢加 `docker create`／`cp`／`rm` 的總耗時作預設；在不改原始 bytes、
   volume owner 或缺檔語意的前提下，以 bounded Host 並行處理獨立節點。final model 已保存且 Host 容量允許時，
   held-out GPU evaluation 可與這些唯讀收集並行；兩者及 evidence check 都成功後，才能執行 guarded reset。
   Reset 的 `apply` 和 `verify` 是清除前後不同狀態，不可當成重複檢查直接合併；各階段先以完整 actual project
   inventory 和 selected `resetScope` 作 exact compare、核對 process 已停，之後才可受控並行清理或檢查互不
   相干的 volumes。Guest DB／ADRF reset 仍在原 owner 執行，任何清理或驗證失敗均不得回報 reset verified。
5. **失敗與補收（adapted）**：現有 exception path 先抓 diagnostics 才 `stop_all()`；改為先盡力停止並恢復
   故障 Guest 的 restart 狀態，再抓可從已停止 container／Guest 取得的 diagnostics 和部分 JSONL；diagnostics
   失敗不阻止 stop。若 stop 未完成，保留失敗資訊與現場資料，不進入 reset。`collect-only` 不重新訓練；
   已完成 run 只重做缺少的收集／評估／reset，失敗 run 只補收當時存在的原始紀錄，兩者均沿用相同
   volume owner、run identity 與 Guest transport。共用連線在成功、失敗和補收結束時都須清理，不因中途
   例外留下本次 run 的 control socket。
6. **驗證收斂（adapted）**：找出同一 run 中重複的 Guest config、NF registration、ML identity／CUDA
   讀取，讓已有直接結果供上層消費，而不是每層重查；獨立 `services-start`／`ml-start` 入口仍須維持自身
   成功語意。保留啟動前 provider Host guard、完整 process／capacity scope、跨 VM 啟用後 selected／active
   config 核對、啟動完成時 active Guest／Host／NRF ready，以及 stop／reset 的 wrong-config 和 exact-scope
   fence。移除任何查詢前，先說明省略後的實際失效結果與保留的 owning check，不把重複次數當成安全性。

已盤點的重複查詢與處置如下。重複情形來自優化前呼叫鏈，不是各步耗時測量；精簡時優先重用同一狀態邊界內的
直接結果，不新增持久 cache 或讓獨立 lifecycle 指令依賴 runner 的記憶。

| 查詢／階段 | 優化前的重複與處置 |
| --- | --- |
| 啟動後 Guest identity／unit 狀態 | `services-start.sh` 在 stage 後確認 Guest identity，`experiment-start.sh` 結尾再確認；runner 的 `active_guest_identity()` 和 `guest_runtime_snapshot()` 又各自先確認 identity，再另外逐 VM／unit 讀狀態。保留跨 stage／啟動邊界必要的確認，將 runner 連續取得的 identity、active target 和 unit 狀態合成一次可供紀錄的 snapshot。 |
| Host ML identity／health／CUDA | `ml-start.sh` 先測 image CUDA，啟動後 `ml-status.sh` 查實際容器；runner 的 `runtime_snapshot()` 又逐容器查 health／CUDA，送訓練前 `verify_runtime_identity()` 再查 ML identity。保留 image 啟動前探測、獨立 `ml-start` 的啟動後成功判斷與 runner 最終 ready 證據，因三者跨越 build／Compose／整體啟動的不同狀態邊界；`experiment-start` 結尾已核對 exact-scope／config，因此 runner 不再於同一 ready 邊界透過 controller 重查 ML identity。獨立執行的 controller 指令仍自行核對。 |
| NRF registration | `services-start.sh` 等待 selected registration，runner 形成 ready snapshot 時又執行一次單次查詢。兩者相隔 Host ML 啟動，不能只憑「查了兩次」刪除；若收斂，保留 final ready 的直接證據與獨立 `services-start` 的成功語意。 |
| 停止 Host ML | `ml-stop.sh` 在停止前後各執行完整 `ml-status.sh`，中間另有等待 no-running 的檢查。停止前改用足以保護 project／container identity 與範圍的精確檢查，停止後沿用 no-running 結果，不為同一動作產生第二份完整狀態報告。 |
| Reset volume | `experiment-reset.sh apply`、`verify` 都先對每個 volume 執行 `volume_state` 內容／大小掃描，之後又分別清除或確認為空。`plan` 保留內容預覽；`apply` 保留清除前完整 inventory／identity fence 和實際清除，`verify` 保留清除後的空值證據，移除兩個執行階段中沒有決策 consumer 的額外內容掃描。 |

Provider／VM state、selected config 和 reset scope 也在多層 wrapper 查詢；只在同一操作、狀態尚未改變且
結果的使用者明確時共用。啟動前、停止前、reset `apply` 前與 `verify` 時跨越不同狀態邊界，不能把舊
snapshot 當成新狀態；獨立執行的指令也仍須自行執行其必要安全檢查。

這項工作已沿用同一條 experiment lifecycle 實作，並以優化後短程 E0 驗證；原本的 E0 成功紀錄只證明
修改前的接線。此次沒有新增 VM、Guest service、deployment selector、人工 config source 或 hash-like
evidence，也沒有改變 provider guard、destructive scope 或 component contract。後續若需改變這些邊界，
仍須先更新計畫並取得使用者決策。

## 8. 實作符合性與驗證現況

| 本 slice 承諾 | 直接證據與現況 |
| --- | --- |
| 選定版本與 native config | Parent gitlinks／component lock 已對齊選定 Go NWDAF、PyMTLF commits；四台 Guest provisioning manifest 核對一致。正式 MNIST E0／E1 設定通過共用 checker 與 PyMTLF native parser；`reparent_leaves_to_root` topology 以同一 `TESTBED`／scenario contract 由 PyMTLF native model 解析。 |
| Active／retained process 分工 | E0 真實短跑只啟動 10 個 Host PyMTLF 與對應 Guest NWDAF；停止的 A* container 可被辨識為 retained，不被當成 active。啟動後的 Guest、Host、NRF identity 檢查與 guarded reset 均通過。 |
| 新版事件與原始紀錄 | 2026-09-21 的 MNIST 2-round 無故障短跑 `protocol-mnist-smoke-20260921-d` 得到兩個 accepted normal rounds、final model、official-test 結果與 10 份逐節點原始 JSONL；run 已標為 `successful`，reset 已驗證。完成 run 與失敗 run 的補收路徑另由受控測試覆蓋。 |
| 完整 run lifecycle 優化 | `vssh` 與 Guest upload 在 runner 內共用每台 VM 的 SSH master；各 VM 的工具同步、provisioning、config stage、path 啟停及 Host 原始紀錄收集依相依屏障並行。Host ML stop 與 reset 移除無決策用途的重複掃描，runner 不再重做 `experiment-start` 結尾已完成的 ML identity 檢查，失敗路徑先停止 process。受控測試覆蓋 upload 失敗不得 activate、部分 activate 回滾、master 遺失不退回 provider、原始 volume 收集失敗不提前 reset，以及完成和失敗 run 的補收。最終版本的 MNIST E0 `protocol-mnist-smoke-20260921-opt2` 在 approved Host context 成功完成兩個 accepted normal rounds、10 份原始 JSONL、final model、official-test 評估與 guarded reset，runner 暫存 SSH socket 已清理。 |
| 工作邊界 | E1／E2 的真實故障與修復、四情境驗收、五 paired seeds 與正式論文結果維持後續 slice；本次短跑不計入正式比較。 |

優化前與最終版本兩次短程 E0 的 run `startedAt` 到 `RUNTIME_READY` 約為 4 分 48 秒與 1 分 51 秒；final artifact 收集
到 held-out 評估約為 115 秒與 39 秒，評估到 cleanup 約為 83 秒與 30 秒。這只是兩次執行的觀察，沒有各內部操作
的獨立計時，不作效能基準或單一變更的因果歸因。優化後 process 已停止且 reset 已驗證；四台既有 VM 已正常關機，
Host-context provider／process inventory 再次確認皆為 `poweroff`。

`tests/repository.sh` 的 provider／config stage 等受控檢查已通過，但完整執行仍在 `runtime-inventory.py` 的
replacement-smoke fixture 因未產生 split manifest 而停止；沒有為了這項與短程 E0 無關的 fixture 額外生成
資料。正式選定設定的共用 checker、PyMTLF native parser、owning tests 與優化後真實 E0 路徑均已通過。
