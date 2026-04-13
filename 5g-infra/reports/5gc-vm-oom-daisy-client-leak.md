# 5GC VM OOM / Daisy Client Leak Investigation Report

**Date:** 2026-04-09
**Component:** 5GC VM — Daisy FL framework (`daisy/examples/07_MTLF_training`)
**Symptom:** VM 長時間運行後某些 process（主要是 ML Service）被 OOM killer 砍掉，系統記憶體持續攀升。

---

## 根本原因

每次 NWDAF 觸發 retrain，Daisy master 都會 spawn 新的 FL client process，但舊的 client **永遠不會自動退出**，導致每輪 retrain 在記憶體中留下 2 個 Python process（各 ~240 MB RSS）。

---

## 問題鏈

### 1. Daisy client 不清除（主因）

`server_api_handler.py` 的 `publish_task` handler 每次被呼叫都無條件 spawn 新 client：

```python
for i, gid in enumerate(distinct_groups):
    subprocess.Popen(cmd)   # handle 直接丟棄，無法追蹤
```

Client 在 FL training 完成後不會自己退出（`start_client_numpy` 持續阻塞），導致每輪 retrain 累積 2 個殭屍 client process。

### 2. shutdown.py 無效

`shutdown.py` 只對 `nodes.yaml` 中的固定地址（`0.0.0.0:8887/10087/10088`）送 gRPC shutdown signal。由於 client 是**主動連入** aggregator（非自己 listen），shutdown signal 打到 node 本身，不會廣播給所有已連入的 client。當 master 被 OOM kill 時，`shutdown.py` 的 gRPC 連線全部失敗，`except: pass` 靜默吞掉，什麼都沒清到。

### 3. VM 無 swap

VM 設定 `Swap: 0B`，記憶體一滿立即觸發 OOM killer，沒有任何緩衝空間。

### 4. ML Service 被 kill 的時機

Model swap 期間，ML Service 同時持有 old + new 兩個 PyTorch model（`SwapModel` 先 load 新的再 unload 舊的），瞬間記憶體多出 ~200-300 MB。系統若已被 daisy client 壓到臨界，此峰值即觸發 OOM，ML Service 首當其衝。

### 5. Subscription 刪除時的 model leak（次要）

`eventssubscription.go` 刪除 subscription 時，只從 SharedModelRegistry 移除 entry，未呼叫 `mlClient.UnloadModel()`：

```go
if remaining == 0 {
    ctx.DeleteSharedModel(modelUrl)
    // ← 沒有呼叫 mlClient.UnloadModel(modelId)
}
```

ML Service 的 `_models` dict 永遠不會清到這個 model。目前 subscription 為長駐型，此 bug 影響有限，但仍屬真實 leak。

---

## 事發當時的 VM 狀態（2026-04-09 ~03:00）

- 總記憶體：3.8 GiB，可用：328 MiB，Swap：0
- Daisy client 累積數：12 個，佔用約 2.7 GB RSS

### 殘留 client 分布（按 TID 分組）

| 建立時間 | TID (前8碼) | group-test-001 | group-test-002 |
|---------|------------|:--------------:|:--------------:|
| Apr08 ~07:46 | `a0422d77` | ✓ | — |
| Apr08 ~08:52 | `3358af33` | — | ✓ |
| Apr08 ~08:58 | `93206a04` | ✓ | — |
| 01:34 | `af6ff5ce` | — | ✓ |
| 01:37 | `7eefc5e0` | — | ✓ |
| 01:46 | `e4876b26` | ✓ | ✓ |
| 02:43 | `934d6721` | ✓ | ✓ |
| 02:46 | `05f314cf` | ✓ | — |
| 02:50 | `8656e39b` | ✓ | ✓ |

多數 TID 只剩一個 group 的 client，另一個已被 OOM kill。正常 pair（兩個 group 都在）只有 3 輪（01:46、02:43、02:50）。

### dmesg OOM 紀錄

```
Killed process 1033830 (python3) anon-rss:331520kB
Killed process 1503157 (python3) anon-rss:330212kB
Killed process 2328191 (python3) anon-rss:331160kB
Killed process 2328227 (python)  anon-rss:244132kB
Killed process 2420597 (python3) anon-rss:331020kB
```

---

## 修復方向

| 優先級 | 項目 | 方法 |
|--------|------|------|
| 立即 | 啟動前清殭屍 client | `pgrep -f 'daisy.*client.py' && pkill -f 'daisy.*client.py'` |
| 高 | Daisy client lifecycle | `server_api_handler.py`：spawn 前先 kill 同 port 的舊 client，或儲存 Popen handle 在 training 完成後主動 terminate |
| 中 | VM 加 swap | 避免 OOM 立即觸發 |
| 低 | Subscription 刪除時 unload ML model | `eventssubscription.go:268`：`remaining == 0` 時補呼叫 `mlClient.UnloadModel(modelId)` |
