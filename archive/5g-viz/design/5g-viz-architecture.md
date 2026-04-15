# 5g-viz 架構說明：資料流程

從 VM log 檔案到瀏覽器渲染的完整流程。

---

## 整體流程

```
5GC VM（free5gc / NWDAF）
  └─ log 檔案（free5gc.log、nwdaf.log）
       │  tail -F over SSH（asyncssh）
       ▼
collector.py  ──►  asyncio.Queue  ──►  parser.py  ──►  main.py (_broadcast)
                                                              │
                                                    WebSocket（FastAPI /ws）
                                                              │
                                                         瀏覽器
                                                    events.js ──► topology.js
```

---

## 1. Collector（`collector.py`）

### SSH 連線

啟動時，`_collect_5gc()` 用 `asyncssh.connect()` 建立與 5GC VM 的持久 SSH 連線。憑證（host、port、user、key 路徑）從 `.env` 透過 `config.py` 讀取。

### 找最新 log 目錄

free5gc 每次啟動都會在 `~/free5gc/log/` 下建立一個新的時間戳子目錄（例如 `20260312_083044/`）。Collector 執行：

```bash
ls -t ~/free5gc/log/ | head -1
```

取得最新的目錄。如果目錄是空的（NF 還沒啟動），每 5 秒輪詢一次直到有目錄出現。

### Tail log 檔案

在 VM 上同時啟動兩個 `tail -F` process：

| Process | 檔案 | Source tag |
|---|---|---|
| `tail -F ~/free5gc/log/<latest>/free5gc.log` | free5gc NF log（SMF、AMF 等） | `"free5gc"` |
| `tail -F ~/nwdaf.log` | NWDAF log（AnLF、MTLF、volume report） | `"nwdaf"` |

`tail -F` 以檔名追蹤，只要檔案存在就持續 stream，檔案被 rotate 或重建後也會自動重新 attach。

每收到一行 stdout，就以下列格式推入共用的 `asyncio.Queue`：
```python
{"source": "free5gc" | "nwdaf", "line": "<原始 log 行>"}
```

### Log 目錄切換偵測

第三個 coroutine `_watch_log_dir()` 每 30 秒檢查一次是否有新的 free5gc log 目錄（代表 free5gc 重啟了）。一旦偵測到新目錄，它 return，觸發 `asyncio.wait(FIRST_COMPLETED)`，取消其他兩個 coroutine 並以新路徑重新連線。

### 容錯機制

- SSH 斷線 / 錯誤 → 捕捉，5 秒後重試
- `tail -F` process 錯誤 → 捕捉，2 秒後重啟
- NF 還未啟動（log 目錄不存在）→ 每 5 秒輪詢等待

---

## 2. Queue（`main.py`）

`_queue` 是標準的 `asyncio.Queue`。Collector 推入原始 log item；`_process_queue()` 逐一消費：

```python
item = await _queue.get()
event = parser.parse_line(item["line"], source=item["source"])
if event:
    await _broadcast(event)
```

單一 consumer loop，事件按 log 順序串行處理。

---

## 3. Parser（`parser.py`）

### 第一步：Base regex

每行 log 先對 logrus 格式做 regex match：

```
time="..." level="..." msg="..." [CAT="..."] [NF="..."] [key="value" ...]
```

提取欄位：`time`、`level`、`msg`、`cat`、`nf`，以及其餘 key-value 對存入 `extra_kv`。

格式不符的行（空行、非 logrus 格式等）直接回傳 `None`。

### 第二步：Rule matching

`rules/` 下所有 rule 在 import 時由 `rules/__init__.py` 自動掃描彙整成 `ALL_RULES`。每條 rule 有一個 `match` dict 和一個 `build` lambda。

對每條 rule 依序檢查：
1. `nf` — 必須符合 `base["nf"]`（例如 `"SMF"`、`"NWDAF"`）
2. `cat` — 必須符合 `base["cat"]`（例如 `"MTLF"`、`"AnLF"`）
3. `source` — 必須符合 collector 的 source tag（例如 `"free5gc"`）
4. `msg` — regex 對 `base["msg"]` 做 search（找不到再 fallback 到整行）

第一條 match 的 rule 獲勝。呼叫 rule 的 `build` lambda（傳入 regex match 物件和 `base` dict），取得 event 欄位，再合入 `type` 和 `time` 組成最終 event。

### 範例

Log 行：
```
time="2026-03-12T08:46:25Z" level="info" msg="Accuracy-triggered retraining completed successfully" CAT="MTLF" NF="NWDAF"
```

Match 到 `rules/nwdaf.py` 中的 rule：
```python
{
    "match": {"nf": "NWDAF", "cat": "MTLF", "msg": re.compile(r"Accuracy-triggered retraining completed successfully")},
    "event": "retrain_done",
    "build": lambda m, base: {},
}
```

輸出 event：
```json
{"type": "retrain_done", "time": "2026-03-12T08:46:25Z"}
```

---

## 4. Broadcast（`main.py`）

`_broadcast(event)` 做兩件事：

1. **State 更新** — `state.apply_event(event)` 更新 server 端狀態（例如 NF 上線狀態、corr_id → SUPI mapping）。
2. **WebSocket broadcast** — 將 event 序列化成 JSON 發送給所有連線中的 client。送失敗的 client 從集合中移除。

---

## 5. WebSocket endpoint（`main.py`）

`GET /ws` 升級成 WebSocket。連線時：
1. Client 加入 `_clients` 集合。
2. 立即送出一個 **state snapshot**（`state.snapshot()`），讓剛連線的瀏覽器不用等新 event 就能渲染目前的 NF 狀態。

之後靠讀取（並丟棄）任何收到的訊息維持連線存活。斷線時從 `_clients` 移除。

---

## 6. 前端 — WebSocket client（`events.js`）

`connect()` 對 `ws://<host>/ws` 建立 WebSocket，斷線後 3 秒自動重連。

每收到一條訊息：
1. JSON parse。
2. `dispatch(event)` 依 `event.type` 路由到對應的 `Topology.*` handler。
3. `appendLog(event)` 在下方 event log 面板新增一行（上限 200 筆，自動捲動）。

---

## 7. 前端 — 拓樸圖渲染（`topology.js`）

拓樸圖使用 [Cytoscape.js](https://js.cytoscape.org/)，節點位置固定（preset layout）。節點：`Consumer`、`NWDAF`（compound parent）、`AnLF`、`MTLF`、`SMF`、`UPF-EES`、`UPF-EES2`。

### Edge flash（`flashEdge`）

大多數事件以「出現後消失」的暫時 edge 呈現：

1. `cy.add()` 新增 edge。
2. `setTimeout` 在 `duration` ms 後移除。
3. **Collapse 機制**：相同 `from→to:label` 的 edge 共用一個 bucket。同時存在超過 2 條時，個別 edge 改為單一 summary edge（`...(N more)`），避免畫面爆炸。

### Node pulse（`pulse`）

用 Cytoscape `animate()` 做邊框閃爍動畫。顏色語意：
- 綠色（`#00e676`）— 預設，一般活動
- 青色（`#00e5ff`）— accuracy check
- 橘色（`#ff7043`）— 警告 / 重訓

每次 pulse 前呼叫 `node.stop(true, true)` 清除已排隊的動畫，避免 burst 時動畫堆積狂閃。

### 持久狀態

部分事件改變節點的 CSS class 而非 flash：
- `nf_up` → 加 `up` class（綠色邊框）
- `retrain_trigger` → 加 `retraining` class（橘色邊框，持續到 `retrain_done`）

### Event → 視覺對應

| Event | 視覺效果 |
|---|---|
| `nf_up` | 節點邊框變綠 |
| `sbi_call` | 暫時 edge（顏色依 label 類型） |
| `upf_volume` | UPF → NWDAF edge "UPF Notify" + UPF pulse |
| `ml_inference` | AnLF pulse |
| `accuracy` | AnLF → MTLF edge（青色）+ AnLF pulse |
| `threshold_breach` | MTLF self-edge "degrade warn N/total"（橘色）+ MTLF pulse 橘色 |
| `retrain_trigger` | MTLF self-edge "retrain activated" + MTLF 持續橘色邊框 |
| `retrain_done` | MTLF → AnLF edge "provision" + MTLF pulse 綠色 + 移除橘色邊框 |
| `model_swap` | AnLF self-edge "swap new model" + AnLF pulse 綠色 |
