# Deployment, Lifecycle And Experiment Integrity

設計或修改 deployment source、generation／validation pipeline、operational lifecycle、capacity 或 experiment
comparison 時使用本模組。實際操作 provider、VM 或執行 destructive operation 時另用
[Runtime Safety](runtime-safety.md)。

## Configuration-source And Single-pipeline Gate

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

Common scenario validation 只限制有 authoritative contract 或可證實 production failure 作為依據的共通不變條件；
其他預期值須由 selected input 推導。單次實驗的設定與驗收值不因此成為跨實驗的合法性限制。修正缺乏依據的限制時，
仍須保留真正必要的語意、安全與失敗防護，不得繞過共用 pipeline。

新增驗證前，必須追蹤若省略該驗證，輸入在既有責任邊界會得到什麼實際結果。若它仍會在正確邊界被正常拒絕，
且此前不會造成錯誤的 production state 或有害副作用，僅為提早報錯不得在上游重複驗證。驗證所需的預期值
必須取自其 authoritative source，不得為了提早拒絕而另建一份配置真相。

## End-to-end Operational Lifecycle Gate

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

## Capacity And Experiment Integrity

Capacity gate必須從selected deployment、runtime inventory與實際execution policy推導完整resource demand，並和每個
resource owner可用的bounded capacity比較；不得只檢查預先列出的resource種類、固定Host reserve或單一runtime domain。
Deployment引入任何新的resource owner、competition或dependency時，其capacity responsibility必須立即進入同一分析，
而不是等policy補上具體名稱後才受檢查。

容量不足是 decision blocker。不得在未重新決策下新增 VM、合併 logical NFs、共用 identity／state、降低必要
isolation，或悄悄縮小 acceptance criterion。

Controlled comparison必須識別、固定或明確記錄所有可能影響結果、但不是本次independent variable的因素。不能因某個
factor未列在既有experiment template就忽略；第一輪只跑通流程時，也不得把混有其他差異的結果歸因於單一設計變因。
特定實驗的設定與acceptance由approved experiment definition指定，從selected input與run evidence核對。
Pilot可以調整仍符合共通contract的參數，但必須記錄實際設定；不得因此放棄配對與可重建性。
