# Model Monitor Cleanup Runtime Validation

日期：2026-08-13

## Result

以`fl-closure-smoke`完成一次真實GPU runtime驗證。A/B federated learning、ADRF publication、
雙scope cutover皆成功；新版`experiment-stop`在停止Consumer前辨識到兩條active PyMTLF-C
Model Monitor subscriptions，並保持ML與Guest backends運作，直到兩條都出現明確`removed`
evidence才繼續teardown。

本輪驗證通過。第二條cleanup歷經三次`503`後收斂，從Consumer exact DELETE到最後removed約
97秒；舊固定40秒grace確實不足，而目前210秒bounded timeout涵蓋本次觀察結果。這仍不是持續
`503`下的理論保證；timeout維持warning後繼續shutdown的設計。

## Identity and scope

- Infrastructure revision：`47ca02f`
- PyAnLF revision：`08798f15c369`
- PyMTLF revision：`7e8ab7f23bf5`
- Scenario：`fl-closure-smoke`
- Ignored config：`config/local/cleanup-convergence-20260813`
- Effective config hash：`c59ccf14b9157845ad6fb63a15aa661c85c9fd2813b69f14728c2b7f0febeab9`
- Dataset set ID：`2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`
- ML device policy：PyAnLF-A/B與PyMTLF-C使用CPU；PyMTLF-A/B使用`cuda:0`
- WebConsole：disabled

執行前主機約有28 GiB available RAM與225 GiB free storage；swap僅餘2 MiB，依現行policy產生
warning但不阻擋。執行期間沒有其他NVIDIA compute process。

## Preparation

新smoke config首次validation只因content-addressed dataset尚未生成而失敗。執行正式
`dataset-generate`後，完整`experiment-validate`為0 failures、1個low-swap warning；config、
dataset、16個component locks、submodule worktrees、VirtualBox、Docker、GPU CDI/runtime皆通過。

依`reset-show`輸出的`RESET_CONFIRM=fl-closure-smoke`執行scoped reset：

- 清除ADRF `data_store_records` 309筆、`mlmodel_store_records` 1筆；
- 清除1個ADRF model file與1筆scoped NRF URI-list state；
- 清空五個ML named volumes內容；
- 保留containers、images、Compose network、named volumes與三台VM。

## FL closure evidence

兩筆Consumer subscriptions於15:16:44建立，A/B provider、TAC、Location皆不同。23個Guest units
active、五個ML containers healthy；PyMTLF-A/B確認CUDA可用，當時各約使用298 MiB container
RAM。

主要時間線（Asia/Shanghai）：

| Event | Time | Evidence |
| --- | --- | --- |
| Consumer subscriptions active | 15:16:44 | A/B exact Locations saved |
| Two initial Model Monitor subscriptions active | 15:16:46 | distinct subscription/correlation IDs |
| Path A degradation trigger | 15:25:46 | `evaluated=True triggered=True` |
| FL preparation complete | 15:25:46 | participants A/B |
| Rounds 0 and 1 aggregated | 15:25:49 | both global artifacts produced |
| Final validation | 15:25:49 | base WAPE 1.83977, candidate WAPE 0.421642 |
| ADRF publication | 15:25:49 | model ID `1786605949743` |
| Two scopes adopted and cutover complete | 15:25:50 | `complete=False` then `complete=True` |
| Post-cutover evaluated accuracy | 15:27:20 | new model `evaluated=True triggered=False` |

## Teardown convergence evidence

`experiment-stop`於15:29:03確認兩條active PyMTLF-C subscriptions，隨後兩筆Consumer exact
DELETE成功。Cleanup時間線：

| Time | Subscription | Result |
| --- | --- | --- |
| 15:29:03 | `dcaba7b7…` | remote DELETE `204`，立即記錄removed |
| 15:29:33 | `0c13c783…` | remote DELETE `503` |
| 15:30:04 | `0c13c783…` | retry `503` |
| 15:30:37 | `0c13c783…` | retry `503` |
| 15:30:41 | `0c13c783…` | retry `404`，記錄removed |
| 15:30:44 | PyMTLF-C | application shutdown complete |

最後一條subscription是在ML shutdown前明確removed；沒有pending-ID timeout warning。停止後兩筆
Consumer resources均為`deleted`、23個Guest units均inactive、五個ML containers均exited，最後
三台VM graceful poweroff。Infrastructure tracked files、submodule worktrees與gitlink pins未改變。

## Non-blocking observation

PyMTLF-C final validation載入seed artifact時出現scikit-learn
`InconsistentVersionWarning`：artifact使用1.3.2建立，runtime為1.9.0。本輪訓練、validation、
publication與cutover仍成功，因此不影響本次cleanup驗收；但它代表模型artifact與runtime
dependency compatibility尚未被明確固定，應另行評估，不在本次Infrastructure teardown修正範圍。
