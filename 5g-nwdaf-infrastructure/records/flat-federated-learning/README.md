# Flat Federated Learning Records

本分類保存 `5G_NWDAF_Infrastructure` 採用 flat federated learning 架構時，已完成的
testbed foundation 與 validation evidence。這些文件描述的是 HFL adaptation 之前的
three-NWDAF、two-TAI、two-UPF baseline，不代表目前更新後的 component revisions 已完成
相同層級的 runtime 驗證。

## 內容

| 分類 | 內容 |
| --- | --- |
| [foundation/](foundation/plan.md) | Repository、VM、network、configuration、runtime 與 full-core testbed 的初始建置歷史 |
| [validation/](validation/README.md) | 依日期保存的 provisioning、runtime、E2E、cleanup 與 regression evidence |

閱讀 individual record 時，應保留其原始日期、revision 與環境脈絡。後續 HFL 工作若沿用其中
的 network、VM 或 operational assets，仍需在新的 validation record 中重新確認，不能直接把
flat FL 結果視為 HFL evidence。
