# State

> Historical note: the state model is still relevant, but this document may reference the older module layout and root-level app wiring.

本文描述 `state.py` 如何從 topology config 與事件流推導出目前的 `state_snapshot`。

## 1. 定位

`state.py` 的責任不是保存完整事件歷史，而是維持一份「目前拓樸狀態」的精簡投影。

這份狀態目前只包含兩種資料：

- `nf_status`：哪些 node 已被視為上線
- `node_classes`：哪些 node 目前帶有哪些持久 class

這份資料會被用在：

- live 模式新 WebSocket 連線建立時的初始同步
- `GET /api/state`
- replay 模式啟動後的初始畫面重建

## 2. 內部資料結構

`state.py` 目前維護的 module-level 狀態如下：

```python
_state = {
    "nf_status": {},    # node_id -> status
    "node_classes": {}, # node_id -> set[str]
}
```

另外還保留三份來自 topology config 的輔助資料：

- `_nf_aliases`
- `_event_reactions`
- `_managed_classes`

其中 `_managed_classes` 來自 `event_reactions` 中所有 `add_class` / `remove_class` action 的 class 名稱集合。

## 3. 初始化來源

`main.py` 在 live 與 replay 兩種模式下，都會先呼叫：

```python
state.init_from_topology(_topo_config)
```

`init_from_topology(cfg)` 會：

1. 從 `cfg["nodes"]` 收集所有合法 `node.id`
2. 把每個 node 初始化為：
   - `nf_status[node_id] = "unknown"`
   - `node_classes[node_id] = set()`
3. 讀入 `nf_aliases`
4. 讀入 `event_reactions`
5. 從 `event_reactions` 收集所有受管 class 名稱

因此 `state` 的資料空間完全由 topology config 定義，而不是由事件在執行中動態擴充。

## 4. `nf_aliases` 的用途

parser 事件中的 `nf`、`from`、`to` 等欄位使用的是 domain 名稱，例如：

- `SMF`
- `NWDAF`
- `UPF-EES`

但前端與 state 真正管理的是 topology 裡的 node ID，例如：

- `smf`
- `nwdaf`
- `upf_ees`

`state.py` 透過 `_nf_aliases` 進行這層對應，避免 parser event 直接綁死前端 node ID。

## 5. `apply_event()` 會做什麼

`apply_event(event)` 目前只處理兩類狀態更新。

### `nf_up`

若事件型別是 `nf_up`：

1. 取出 `event["nf"]`
2. 經過 `_nf_aliases` 轉成 node ID
3. 若該 node 存在於 `_state["nf_status"]`，就把狀態設為 `up`

這是目前唯一會改動 `nf_status` 的事件。

### `event_reactions` 中的 `add_class` / `remove_class`

對所有事件，`state.py` 都會去查：

```python
_event_reactions.get(event.get("type")) or []
```

若 action type 是：

- `add_class`
- `remove_class`

就會把它視為後端也需要持久保存的狀態變更。

其他 action，例如：

- `flash_edge`
- `pulse`

都只屬於前端瞬時視覺效果，`state.py` 會忽略。

## 6. 模板解析

`event_reactions` 裡的 `node` 欄位可以寫成模板，例如：

```yaml
nf_up:
  - add_class: { node: "{nf}", class: up }
```

`state.py` 會先用 `_resolve_template()` 把 `{nf}` 之類的占位符替換成 event 內的欄位值，再用 `_resolve_node()` 套用 `_nf_aliases`。

這讓同一份 `event_reactions` 可以同時服務：

- 前端的視覺反應
- 後端的 `state_snapshot`

而不需要為兩邊各寫一套規則。

## 7. Snapshot 形式

`snapshot()` 會回傳：

```json
{
  "type": "state_snapshot",
  "nf_status": {
    "smf": "up"
  },
  "node_classes": {
    "smf": ["up"],
    "nwdaf_mtlf": ["retraining"]
  }
}
```

其中：

- `node_classes` 在輸出前會把 `set[str]` 轉成排序後的 list
- `snapshot()` 不包含 event 歷史、邊動畫、時間軸資訊

## 8. Live 與 Replay 的使用方式

### live

live 模式下，每當 `_process_queue()` 產生一筆事件，`main.py` 會在 `_broadcast(event)` 內先呼叫：

```python
state.apply_event(event)
```

然後才把該事件送給所有 WebSocket client。

因此新 client 連上 `/ws` 時，收到的 `state_snapshot` 代表的是「截至目前最新事件後」的畫面狀態。

### replay

replay 模式下不會從 queue 漸進更新，而是在啟動時：

1. `state.init_from_topology(...)`
2. 逐筆對 `_events` 呼叫 `state.apply_event(event)`

也就是說，replay 的初始 snapshot 是由整份 `events.jsonl` 重播後算出來的結果。

## 9. 與前端的分工

前端在 render 畫面時仍會自己解讀 `event_reactions`，但 `state.py` 提供了兩個額外保證：

- 新連線不需要重播整份事件才能知道目前哪些 node 已經 `up`
- 持久 class 不必依賴前端自行從頭演算一次

這特別適合 live 模式下晚加入的瀏覽器 client。

## 10. 目前限制

- `state.py` 目前只保存 node 狀態，不保存 edge 狀態
- class 一旦加上，就會持續存在，直到出現對應的 `remove_class` 或整個 session 重新初始化
- `nf_status` 只有 `unknown` / `up` 兩種實際狀態，沒有 `down`、`degraded` 等更細緻模型
- `_managed_classes` 目前只作為 state 初始化時的受管集合，沒有進一步暴露為前端或 API 契約的一部分
