# Protocol-driven Branch Replacement Validation Record

執行日期：2026-09-12 至 2026-09-13

User review：2026-09-13 confirmed

狀態：Completed / Verified Record

## 1. 目的與宣稱邊界

本 record 保存四台 VM、十一組 NWDAF↔PyMTLF runtimes 上，MNIST 與 CIFAR-10 各一次
protocol-driven hierarchical FL Branch replacement flow 的實際證據。每個 run 均執行：

```text
2 normal accepted rounds
→ hard fail-stop Area A primary Branch
→ Root 以 Area B／C 接受 degraded rounds
→ Root 依 priority 準備 Area A replacement
→ replacement 從後續 cohort 恢復貢獻
→ 共完成 8 accepted rounds
→ 保存 final Root model
→ independent held-out evaluation
→ exact stop／reset
```

本次只證明 topology、GPU training、failure detection、priority replacement、structured evidence、final-model
handoff、collection retry 與 cleanup flow 在指定 revisions 和環境上閉合。不宣稱模型品質 benchmark、收斂保證、
MNIST／CIFAR-10 優劣、Branch failure 的因果效果，或 hierarchical FL 優於 Flat FL。

實驗契約由
[Multi-host Branch Replacement Experiment Plan](../../plans/protocol-driven-hierarchical-fl-experiment/multi-host-branch-replacement-experiment-plan.md)
與
[Slice 3 Detailed Plan](../../plans/protocol-driven-hierarchical-fl-experiment/slices/slice-3-dual-dataset-branch-replacement-acceptance-detailed-plan.md)
管理。

## 2. Source、config 與 runtime identity

兩個正式 run 使用相同 source 與 runtime baseline：

| 項目 | 實際 identity |
| --- | --- |
| Infrastructure branch／run-time HEAD | `feat/hierarchical-fl-protocol-extension`／`8495f4af5b4106a68fa9b34fdeddb1aa02de67d7`，包含本次未提交 working-tree implementation |
| Go NWDAF | `be3fa576a22c6b787c9ce578c79621b486641e6d` |
| PyMTLF | `bdbd2a9591e47d59b690f0031dfebc77f757bb91` |
| NRF | `0dd4024d4ab75b6630e04901968228b9b9718cf5` |
| ADRF | `905f0599f68fe389bba14ed56db0ef9abeab5ccd` |
| Selected testbed | `testbed.protocol-hierarchical.yaml` |
| Generated config directory | `config/local/protocol-hierarchical` |
| PyMTLF image | `sha256:ddd6002995d211b5693b361784bd12f277da7cadc12f50f1502dec02ecebde02` |
| OCI revision | `bdbd2a9591e47d59b690f0031dfebc77f757bb91` |

Infrastructure 在執行時標記為 dirty，因為本 record 驗證的正是尚待 commit 的 Slice 3 working-tree
implementation；各 run 的 `run.json` 保存當時完整 repository revision 與 dirty flag。Component repositories
均為 clean，PyMTLF image revision 與 checked-out source一致。

Physical topology為 `core`、`path-a`、`path-b`、`path-c`四台VM。Logical topology為一個Root、三個初始
active Branches、Area A一個inactive replacement Branch，以及六個Leaves；十一個Guest Go NWDAF processes
和十一個Host PyMTLF containers一對一對應。

## 3. Workload 與 GPU admission

兩個run皆使用seed `42`、每Leaf 8,000筆training samples、Root 200筆validation samples、200筆獨立held-out
samples、batch size 16、learning rate 0.001、local epochs 32及8 accepted rounds。

Host GPU為一張NVIDIA GeForce RTX 3080，driver `535.183.01`，總記憶體10,240 MiB。Admission時free memory為
9,988 MiB，高於8,192 MiB門檻。Root與六個Leaves共七個participants使用`cuda:0`；四個aggregation-only
Branches維持CPU。兩個正式run均未出現GPU OOM或silent CPU fallback。

## 4. Branch failure 與 replacement evidence

Controller從selected topology的Area A candidate pool解析priority `100` primary，先凍結Guest與Host兩個
application owners，再以`SIGKILL`終止，並確認systemd與container均未自動重啟。Controller沒有對Root指定replacement、
修改priority或控制preparation timing。

Root在primary失效後以Area B／C接受round，並從同一candidate pool選出priority `50` replacement。兩個run均自然
形成兩個degraded rounds；replacement從下一個尚未dispatch的cohort開始貢獻，之後完成四個restored rounds。

| Dataset | Primary stop→failure detected | Stop→replacement ready | Stop→first contribution |
| --- | ---: | ---: | ---: |
| MNIST | 256.134秒 | 257.002秒 | 373.649秒 |
| CIFAR-10 | 256.226秒 | 257.106秒 | 373.606秒 |

## 5. Round 與模型表現

下表使用Root node-local JSONL中的`MODEL_EVALUATION`。Accuracy以200筆per-round validation samples計算；階段
依accepted `ROOT_ROUND_OUTCOME`與replacement contribution分類。

### 5.1 MNIST

| 階段 | Round | Validation loss | Accuracy |
| --- | ---: | ---: | ---: |
| Initial | — | 2.3086 | 10.0% |
| Normal | 1 | 1.3796 | 51.0% |
| Normal | 2 | 0.7109 | 79.0% |
| Degraded | 3 | 0.5259 | 84.0% |
| Degraded | 4 | 0.4331 | 86.0% |
| Restored | 5 | 0.3736 | 88.0% |
| Restored | 6 | 0.3283 | 89.5% |
| Restored | 7 | 0.2952 | 92.5% |
| Restored | 8 | 0.2705 | 92.5% |

獨立held-out evaluation為185／200，accuracy 0.925。Final Root artifact為74,634 bytes。

### 5.2 CIFAR-10

| 階段 | Round | Validation loss | Accuracy |
| --- | ---: | ---: | ---: |
| Initial | — | 2.3032 | 11.0% |
| Normal | 1 | 1.9725 | 22.0% |
| Normal | 2 | 1.6900 | 39.0% |
| Degraded | 3 | 1.6318 | 43.0% |
| Degraded | 4 | 1.5770 | 46.5% |
| Restored | 5 | 1.5251 | 48.5% |
| Restored | 6 | 1.4760 | 50.5% |
| Restored | 7 | 1.4296 | 51.0% |
| Restored | 8 | 1.3878 | 51.0% |

獨立held-out evaluation為100／200，accuracy 0.5。Final Root artifact為76,695 bytes。

兩個dataset的validation loss在本次run內皆持續下降，但沒有paired no-failure baseline、multi-seed repetition或
統計設計，因此不能把變化歸因於failure／replacement，也不能用兩個accuracy數值作跨dataset模型比較。

## 6. Run identity 與 evidence preservation

| Dataset | `RUN_NAME` | `requestId` | `planId`／`mlCorreId` | Phase counts |
| --- | --- | --- | --- | --- |
| MNIST | `mnist-replacement-20260913-b` | `2aad3ece-cd8e-4dcf-b10f-cbb661fe7ec8` | `768e68dd-e375-4e92-9e2d-3f2519b28fc2` | 2 normal、2 degraded、4 restored |
| CIFAR-10 | `cifar10-replacement-20260913-a` | `3a090f6f-bb27-4f89-8569-d71c8279f5a5` | `a6520a53-edc1-47cf-abda-0404012c6137` | 2 normal、2 degraded、4 restored |

每個ignored run directory保存`events.jsonl`、`run.json`與`final-root-model.tar.gz`；一般文字log只在需要時以
bounded diagnostics保存。Raw evidence與model不進Git。

MNIST training完成後，第一次held-out collection因run-local artifact權限而失敗。修正collector後，以相同
`RUN_NAME`、`requestId`、`planId`和Root procedure record執行collection-only成功；沒有重新提交training request，
成功finalization後才執行reset。CIFAR-10隨後由default training→collection path一次完成。

## 7. Cleanup 與 verification

兩個run都在evidence finalization後停止selected processes、恢復Guest restart policy，並exact reset NRF／ADRF／
volume experiment state。Reset verification顯示selected NRF profiles、ADRF records與model files為空。最後四台VM均經
guarded lifecycle關閉並觀測為`poweroff`；datasets、images、VM disks與ignored run evidence保留。

| 驗證 | 結果 |
| --- | --- |
| Branch replacement focused behavioral suite | PASS |
| Infrastructure full `make test` | PASS；synthetic provider fixtures，未啟動real provider或service process |
| MNIST `config-create DEVICE=gpu`、dataset與`experiment-validate` | PASS |
| MNIST real multi-host GPU replacement flow | PASS；collection-only retry後finalized |
| CIFAR-10 `config-create DEVICE=gpu`、dataset與`experiment-validate` | PASS |
| CIFAR-10 real multi-host GPU replacement flow | PASS |
| Two-file evidence、held-out、process stop與exact reset | PASS |
| Mandatory initial／targeted review | PASS；無尚未處理的current-slice finding |
| User review | PASS；2026-09-13 confirmed |

保留但不維護的legacy Flat／static HFL `make test-containers`因舊seed-artifact digest與目前PyMTLF seed不一致而
失敗。主計畫已將legacy experiment regression列為approved deferral；canonical eleven-container GPU boundary已由上述
兩個real runs直接驗證，因此沒有加入legacy compatibility workaround。

## 8. 限制與後續

- 本次是bounded flow acceptance，不是模型品質或效能benchmark。
- Validation與held-out集合都只有200筆；accuracy不代表完整dataset benchmark結果。
- 沒有paired baseline或multi-seed evidence，不能量化Branch failure的模型影響。
- 本次不驗證retained-result recovery、old Branch handback、Leaf replacement、多Branch failure或Root restart recovery。
- 本次不包含`5g-viz`、Prometheus／Grafana、communication cost、GPU utilization或energy instrumentation。
- Legacy Flat／static HFL regression保持不維護狀態，不影響本record的canonical flow結論。
