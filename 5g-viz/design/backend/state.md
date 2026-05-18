# State

本文描述目前 `runtime/state.py` 如何從 topology config 與事件流推導出 `state_snapshot`。

## 1. 定位

`state.py` 的責任不是保存完整事件歷史，而是維持一份「目前拓樸狀態」的精簡投影。

這份狀態目前主要包含兩類資料：

- `nf_status`
- `node_classes`

它會被用在：

- live 模式新 WebSocket 連線的初始同步
- `GET /api/state`
- live `Go Live` 回跳
- replay 啟動後的初始 state rebuild

## 2. 狀態來源

state 的資料空間由 session / profile 所使用的 `topology.yaml` 決定。

初始化時會讀入：

- `nodes`
- `nf_aliases`
- `event_reactions`

並建立：

- `nf_status[node_id] = "unknown"`
- `node_classes[node_id] = set()`

因此 state 管理的是 topology 定義範圍內的 node，不會在事件流中無限制動態擴充。

## 3. `nf_aliases` 的用途

parser 事件中的 `nf`、`from`、`to` 使用的是 domain 名稱，例如：

- `SMF`
- `NWDAF`
- `UPF-EES`

state 真正管理的則是 topology node id，例如：

- `smf`
- `nwdaf`
- `upf_ees`

`state.py` 透過 `nf_aliases` 做這層對應，避免 parser 事件直接綁死前端 node id。

## 4. `apply_event()` 會做什麼

目前 `apply_event(event)` 只處理會留下持久狀態的部分。

### `nf_up`

若事件型別是 `nf_up`：

1. 取出 `event["nf"]`
2. 經 `nf_aliases` 轉成 node id
3. 將該 node 的 `nf_status` 設為 `up`

這是目前唯一會直接改動 `nf_status` 的事件。

### `add_class` / `remove_class`

對所有事件，state 都會查對應的 `event_reactions`。

若 action 是：

- `add_class`
- `remove_class`

就把它視為後端也需要持久保存的狀態變更。

其他 reaction，例如：

- `flash_edge`
- `pulse`

都屬於前端瞬時視覺效果，state 不會保存。

## 5. 模板解析

`event_reactions` 裡的 `node` 欄位可以是模板，例如：

```yaml
nf_up:
  - add_class: { node: "{nf}", class: up }
```

state 會先做模板替換，再套用 `nf_aliases` 解析成真正 node id。

這讓同一份 `event_reactions` 能同時服務：

- 前端視覺反應
- 後端 `state_snapshot`

## 6. Snapshot 形式

目前 snapshot 會回傳：

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

- `node_classes` 會在輸出前把 set 轉成排序後的 list
- snapshot 不包含 event 歷史、邊動畫或 timeline 資訊

## 7. Live 與 Replay 的使用方式

### live

live 模式下，每當 backend 產生一筆事件，就會先呼叫 `state.apply_event(event)`，再把該事件送給 WebSocket clients。

因此新 client 連上 `/ws` 時收到的 `state_snapshot`，代表的是「截至目前事件流位置」的畫面狀態。

### replay

replay 模式下不會從 queue 漸進更新，而是在啟動時：

1. `state.init_from_topology(...)`
2. 逐筆對 `_events` 呼叫 `state.apply_event(event)`

也就是說，replay 的初始 snapshot 是由整份 `events.jsonl` 重播後算出的結果。

## 8. 與前端的分工

前端仍會自己解讀 `event_reactions`，但 state 提供兩個額外保證：

- 新連線不需重播整份事件，就能知道目前哪些 node 已 `up`
- 持久 class 不需依賴前端自行從頭演算一次

這特別適合 live 模式下晚加入的瀏覽器 client，以及 Go Live 回跳。

## 9. 目前限制

- 目前只保存 node 狀態，不保存 edge 狀態
- class 一旦加上，就會持續存在，直到出現對應 `remove_class` 或整個 session 重新初始化
- `nf_status` 目前沒有 `down`、`degraded` 等更細緻模型
- state 是權威 snapshot，不是 event history cache；若需要時間軸與動畫語意，仍需回到 `_events`
