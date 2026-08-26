# Phase 2 Static Scenario Common Foundation Detailed Plan

日期：2026-08-27

狀態：Decision baseline confirmed；implementation pending

## 1. 目的

在不增加 VM 的前提下，讓 `5G_NWDAF_Infrastructure` 能由不同 `CONFIG_DIR` 描述並嚴格驗證 static Flat
與 static Hierarchical 的完整 logical runtime inventory。Phase 2 固定 NF identity、八個 UE、四個 data
owners、process placement、endpoint isolation、lifecycle 與 reset contract，但尚不宣稱兩條 static flow 已完成
collection、training 或 publication。

## 2. 已確認決策

1. 沿用 `core`、`path-a`、`path-b` 三台 VM，不新增 VM。
2. production Flat、static Flat 與 static HFL 使用不同 config directories；所有 lifecycle commands 必須使用
   同一 selected `CONFIG_DIR`。
3. static Flat 的 Client 1–4 與 static HFL 的 Leaf 1–4 具有一一對應的 logical data-owner positions，但由各自
   scenario config 啟動。
4. static scenarios 使用八個 UE；每個 Client／Leaf 擁有兩個互斥 UE／SUPI，兩條 data paths 各有四個 UE。
5. 每個 Server／Root、Client／Leaf、Branch 都是獨立 NWDAF NF，具有獨立 NF Instance ID、Go process、
   config、endpoint、NRF registration、state 與 PyMTLF process；只共用 Go binary 與 component revision。
6. scenario switching 維持 explicit clean-start。使用者負責 stop 與 guarded reset，啟動流程不自動清除或
   覆蓋 active scenario。

## 3. Target runtime inventory

| Placement | Static Flat | Static HFL | Data ownership |
| --- | --- | --- | --- |
| `core` VM | 1 Server NWDAF | 1 Root NWDAF＋2 Branch NWDAFs | 無 UE training partition |
| `path-a` VM | Client 1、2 NWDAFs | Leaf 1、2 NWDAFs | 四個 UE；每個 data owner 兩個 SUPI |
| `path-b` VM | Client 3、4 NWDAFs | Leaf 3、4 NWDAFs | 四個 UE；每個 data owner 兩個 SUPI |
| host ML runtime | 5 PyMTLF containers | 7 PyMTLF containers | 每個 NWDAF 對應獨立 PyMTLF state |

Branch 同時啟用 downstream aggregation server 與 upstream client engines，但不配置 private collection、UE
partition 或 autonomous Root orchestration。Client／Leaf 是唯一 static training-data owners。

## 4. 已發現的 implementation gaps

目前 production Flat lifecycle 仍以 A／B／C 固定清單為中心，包括：

- Guest service start／stop／status 與 `service-run.sh` 只認得既有 `nwdaf-a`、`nwdaf-b`、`nwdaf-c`；
- Compose 與 ML start／status／reset 只描述 `pymtlf-a`、`pymtlf-b`、`pymtlf-c` 及兩個 PyAnLF containers；
- renderer、config checker、dataset tooling 與 observability assertions 直接引用 A／B／C names；
- reset 的 Guest unit、ML service 與 volume scopes 是 hard-coded，無法安全涵蓋額外 Branch／Leaf instances；
- 現有六個 UE 與兩條 path 的 service inventory 必須擴充為八個 UE，並建立四份互斥 collection profiles；
- 同一 VM 上啟動多個 Go NWDAF processes 需要明確的 config selection、IP／port binding、NF identity、systemd
  instance dispatch、log 與 runtime-directory isolation。

Phase 2 不以複製 binary、source checkout 或整台 VM 作為新增 NF 的方式；應以同一 pinned binary 啟動多個
隔離的 systemd process instances。

## 5. State 與 seed model contract

現有 lifecycle 的保留／清理語意應延伸至所有 static instances：

- `experiment-stop` 保留 datasets、databases、ADRF state、ML artifacts、model state、containers、images 與
  volumes；
- `experiment-start` 要求 clean process state；任何既有 experiment Guest unit、ML container 或 consumer active
  時拒絕啟動；
- reset 必須由 selected manifest 列出 exact Guest units、host containers、volumes、ADRF collections 與 model
  storage，先提供 plan，再要求 scenario name confirmation 才執行；
- reset 清除 runtime seed copy、trained models、FL workspaces、publication journals 與 ADRF experiment state，
  但保留 container image 內 canonical seed source；
- reset 後第一次 coordinator 啟動重新匯入 deterministic seed，且 artifact key 必須與 config lock 相符；
- 不新增自動 scenario replacement，也不因選擇另一個 `CONFIG_DIR` 隱式 reset。

## 6. Implementation slices

### Slice 1. Capacity、identity 與 endpoint inventory

- 記錄三台 VM 的 CPU、memory、active IP aliases、可用 ports 與 systemd template contract；
- 記錄 host CPU／memory／GPU capacity、Docker network 與 published-port boundary；
- 配置五個 base data/coordinator positions 與兩個 HFL-only Branch positions；
- 驗證每個 NF 的 NRF profile、SBI、internal API、callback 與 PyMTLF endpoint 可唯一表示且互相可達；
- 若既有資源不足，停止並回報，不新增 VM 或合併 NF identity。

### Slice 2. Scenario manifest 與 config layout

- 新增 static Flat 與 static HFL config directories；
- 讓 manifest 明確列出 Guest machines、NWDAF NFs、ML processes、UEs、data-owner mapping 與 volumes；
- production Flat default config 保持三 NWDAF／六 UE 原狀；
- renderer 與 strict checker 由 selected manifest 驗證 exact topology，而不是以 A／B／C 固定清單推斷。

### Slice 3. Independent NWDAF NF lifecycle

- 讓同一個 Go binary 能依 systemd instance 選取正確 config；
- 每個 NF 使用獨立 NF Instance ID、bind endpoint、runtime directory 與 log identity；
- start／status／stop 依 manifest inventory 操作，不遺留未宣告或 stale NF process；
- HFL Branch capability 與 paired PyMTLF server＋client engines 必須一致。

### Slice 4. Dynamic host ML inventory

- 讓 Compose render、start、status、stop 與 health checks 支援五個或七個 PyMTLF services；
- 每個 PyMTLF 使用自己的 config、published endpoint、volume 與 containing-NWDAF internal API root；
- 不為 pure Branch 配置 PyAnLF 或 training-data collection ownership；
- resource limits 與 device policy 必須在現有 host capacity 內通過 validation。

### Slice 5. Eight-UE data-owner mapping

- 將 static scenario 的兩條 paths 各配置四個 UE，共八個唯一 SUPIs；
- 四份 Client／Leaf collection profiles 各包含兩個 SUPIs，彼此無重疊且全集恰為八個；
- Flat Client N 與 HFL Leaf N 使用相同 logical data partition；
- subscriber data、SMF session resolution、UPF stimulus dataset 與 service inventory 保持一致。

### Slice 6. Manual reset and seed restoration

- reset plan／apply／verify 全部改用 selected manifest inventory；
- reset 拒絕任何 declared 或 unexpected experiment process 仍 active 的狀態；
- 驗證新增 NF／ML volumes 與 workspaces 均被納入 exact scope；
- 驗證 runtime state 清空後，coordinator startup 能重新產生相同 seed artifact key；
- containers、images、volume objects、VMs 與 canonical seed source 必須保留。

### Slice 7. Static validation gate

- render／validate production Flat、static Flat 與 static HFL 三組 configs；
- 使用 pinned NWDAF／PyMTLF native config validation；
- 驗證 NF identities、ports、callbacks、topology edges、UE ownership、volumes 與 cleanup scopes 無衝突；
- Phase 2 只宣稱 common foundation ready，不以 host-only validation 取代 Phase 3／4 full flow evidence。

## 7. Verification matrix

| Layer | Verification | Gate |
| --- | --- | --- |
| Capacity | three-VM 與 host runtime inventory | 五／七個 logical NFs 與 ML processes 可配置；不足時明確阻擋 |
| Config | 三條 config lines render、parent checks、native loaders | exact mode／role／topology／collection ownership 全部通過 |
| NF identity | NF Instance ID、SBI／internal endpoints、NRF profiles | 所有同時執行的 NFs 唯一且可達 |
| Data owners | 八個 SUPIs 與四份 profiles | 每個 owner 恰兩個、無交集、無遺漏 |
| Lifecycle | manifest-driven start／status／stop inventory | 不使用 A／B／C hard-coded topology assumptions |
| Reset | plan／apply／verify 加 seed re-import check | exact state 清空，canonical seed 可重建為相同 identity |
| Repository | focused tests、`make test`、Compose checks、`git diff --check` | 全部通過 |

## 8. Phase completion criteria

只有同時滿足下列條件才完成 Phase 2：

1. production Flat default config 與現有 runtime contract 未被改成 static scenario。
2. static Flat 可 render／validate 一 Server、四 Clients、八 UE 與五組獨立 NF／PyMTLF identities。
3. static HFL 可 render／validate 一 Root、兩 Branches、四 Leaves、八 UE 與七組獨立 NF／PyMTLF identities。
4. 三台 VM／host capacity 與所有 endpoints 通過 inventory gate，未新增 VM。
5. lifecycle 與 reset scopes 由 selected manifest 產生，不遺漏新增 instances。
6. reset 後 canonical seed restoration identity 驗證通過。
7. Diff 經 user review、verification 通過，並使用獨立 commit。

## 9. 明確不包含

- static Flat full collection／FedAvg／publication runtime acceptance；
- static HFL Leaf fitting／Branch aggregation／Root aggregation runtime acceptance；
- Flat FedProx、controlled performance comparison 或 Branch latency attribution；
- 自動停止、清理或覆蓋 active scenario；
- 新增、重建或刪除 VM；
- production Flat 從六個 UE 改為八個 UE；
- dynamic hierarchy、hot reload 或 arbitrary-depth topology。
