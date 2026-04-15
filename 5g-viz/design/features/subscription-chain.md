# Subscription Chain

本文從功能視角描述 `5g-viz` 目前如何把「Consumer 訂閱 NWDAF analytics，並一路串到 SMF、UPF，再回到 Consumer」這條鏈路轉成可觀察的事件與拓樸反應。

若要看各子系統細節，可再對照：

- [`../overview/event-schema.md`](../overview/event-schema.md)
- [`../backend/parser.md`](../backend/parser.md)
- [`../frontend/topology.md`](../frontend/topology.md)
- [`../reference/topology-yaml.md`](../reference/topology-yaml.md)
- [`../reference/env-config.md`](../reference/env-config.md)

## 1. 功能範圍

這條功能線關心的是 subscription control path 與其後續通知鏈，包含：

- Consumer 對 NWDAF 建立 analytics subscription
- NWDAF 對 SMF 建立事件訂閱
- SMF 對對應的 UPF 建立 EES subscription
- UPF 之後持續把事件通知回 NWDAF
- NWDAF 再把 analytics notification 送回 Consumer

這條 feature 主要影響：

- event log
- topology 動畫與 pulse
- session 錄製 / replay 時的拓樸重播

它目前不直接產生 Prometheus metrics，也不直接改變 Grafana 曲線。

## 2. 鏈路總覽

目前在 `5g-viz` 中，使用者可觀察到的主鏈路可整理成：

```text
Consumer
  -> NWDAF   (CreateSubscription)
  -> SMF     (EventExposure Subscribe)
  -> UPF     (EES subscription)
  -> NWDAF   (volume / event notify)
  -> Consumer (analytics notification)
```

其中前四段不完全來自同一個 event type：

- `Consumer -> NWDAF`、`NWDAF -> SMF`、`SMF -> UPF`、`NWDAF -> Consumer` 目前都以 `sbi_call` 表示
- `UPF -> NWDAF` 目前不是 `sbi_call`，而是 `upf_volume`

也就是說，畫面上看起來像一條完整 subscription chain，但在內部資料模型裡，其實是由多種事件共同組成。

## 3. 事件如何把這條鏈路拼起來

### Consumer -> NWDAF

`rules/nwdaf_sub.py` 會把 NWDAF 收到的：

```text
Handle CreateSubscription
```

轉成：

```json
{
  "type": "sbi_call",
  "from": "Consumer",
  "to": "NWDAF",
  "label": "Nnwdaf_EventsSubscription_Subscribe"
}
```

這是整條鏈在畫面上的起點。

### NWDAF -> SMF

當 NWDAF 決定向 SMF 建立 UE session event 訂閱時，`rules/nwdaf_sub.py` 會把 log 轉成：

```json
{
  "type": "sbi_call",
  "from": "NWDAF",
  "to": "SMF",
  "label": "Nsmf_EventExposure_Subscribe",
  "supi": "...",
  "corr_id": "..."
}
```

這一段除了驅動邊動畫，也會把 `supi` 與 `corr_id` 保留在 event 裡，供 event log 或後續人工追查使用。

### SMF -> UPF

`rules/smf.py` 會在偵測到：

```text
UPF subscription created
```

時產生另一筆 `sbi_call`：

```json
{
  "type": "sbi_call",
  "from": "SMF",
  "to": "UPF-EES",
  "label": "Nupf_EventExposure_Subscribe"
}
```

這裡的 `to` 並不是直接從 log 固定寫死，而是由 `selected_upf_api_root` 內的 IP 經過 `UPF_EES_API_IPS` 映射得出。

也就是說，SMF 這段是否會指向正確的 `UPF-EES` / `UPF-EES2`，取決於 profile `.env` 中的：

- `UPF_EES_API_IPS`

### SMF 訂閱確認

NWDAF 之後還會產生一筆：

```json
{
  "type": "smf_sub_confirmed",
  "supi": "...",
  "corr_id": "..."
}
```

這筆事件代表 NWDAF 端已確認 SMF subscription 建立成功，但它目前有一個重要特性：

- 會被錄進 event log 與 session
- 目前沒有 `event_reactions`
- 目前也沒有 metrics handler

因此它是這條功能線中的「語意事件」，但不是可見動畫的一部分。

### UPF -> NWDAF

subscription chain 建立後，UPF 持續回報的 volume/event notify 目前是由 `rules/nwdaf.py` 解析成：

```json
{
  "type": "upf_volume",
  "upf": "UPF-EES",
  "ip": "10.10.0.2",
  "ul_bytes": 123,
  "dl_bytes": 456
}
```

這裡的 `upf` 不是直接從 node ID 來，而是由資料面 IP 經過 `UPF_DATA_SUBNETS` 映射得出。

因此 profile `.env` 中的：

- `UPF_DATA_SUBNETS`

會決定 `UPF -> NWDAF` 這一段動畫最後落在哪個 UPF node 上。

### NWDAF -> Consumer

最後，當 NWDAF 把 analytics notification 成功送回 Consumer 時，`rules/nwdaf_sub.py` 會再產生一筆：

```json
{
  "type": "sbi_call",
  "from": "NWDAF",
  "to": "Consumer",
  "label": "Nnwdaf_EventsSubscription_Notify",
  "sub_id": "...",
  "n": 1
}
```

這讓整條鏈在拓樸上能回到 Consumer。

## 4. 前端為什麼能把它畫成一條鏈

這條 feature 在前端能成立，依賴兩層契約。

### `sbi_call` 的 generic reaction

`topology.yaml` 目前對 `sbi_call` 的反應是：

```yaml
sbi_call:
  - flash_edge: { from: "{from}", to: "{to}", label: "{label}" }
  - pulse: "{from}"
```

這代表只要 parser 產出的 event payload 長這樣：

- `from`
- `to`
- `label`

前端就能用同一套 generic reaction 來畫：

- Consumer -> NWDAF
- NWDAF -> SMF
- SMF -> UPF
- NWDAF -> Consumer

也就是說，subscription chain 在 UI 上不是靠四套硬編碼 handler，而是靠單一 `sbi_call` reaction 重用出來的。

### `upf_volume` 的專用 reaction

`UPF -> NWDAF` 這一段不是 `sbi_call`，所以它走的是另一條 reaction：

```yaml
upf_volume:
  - flash_edge: { from: "{upf}", to: nwdaf, label: Nupf_EventExposure_Notify }
  - pulse: "{upf}"
```

這讓控制鏈延伸到「訂閱建立後，UPF 如何持續 notify NWDAF」這一步。

## 5. 這條功能線的 cross-layer 契約

### `nf_aliases` 必須能解析 `from` / `to`

`sbi_call` 內的 `from`、`to` 在 frontend 會先經模板替換，再透過 `nf_aliases` 解析成真正的 node ID。

因此下列名稱必須在 topology config 中有對應：

- `Consumer`
- `NWDAF`
- `SMF`
- `UPF-EES`
- `UPF-EES2`

否則事件雖然存在，邊動畫會找不到正確節點。

### `edge_styles` 必須包含對應 label

subscription chain 目前依賴的 edge label 主要有：

- `Nnwdaf_EventsSubscription_Subscribe`
- `Nnwdaf_EventsSubscription_Notify`
- `Nsmf_EventExposure_Subscribe`
- `Nupf_EventExposure_Subscribe`
- `Nupf_EventExposure_Notify`

若 `edge_styles` 漏掉其中某條，前端仍會畫 edge，但只會回退成預設樣式。

### `.env` 內的 UPF 映射會改變鏈路終點

這條功能線不是完全由 topology config 決定，還有一部分受 profile `.env` 影響：

- `UPF_EES_API_IPS`：影響 `SMF -> UPF` 這段 `sbi_call.to`
- `UPF_DATA_SUBNETS`：影響 `UPF -> NWDAF` 這段 `upf_volume.upf`

也就是說，就算 `nodes` 與 `nf_aliases` 都沒改，只要 profile `.env` 映射改了，subscription chain 的終點 node 也可能改變。

## 6. 與 DVR / Replay 的關係

這條 feature 不直接進 Prometheus，但仍完整參與 DVR。

### 錄製

live mode 下，這些事件都會被寫進：

- `events.jsonl`

因此 session 內會保留：

- 各段 `sbi_call`
- `smf_sub_confirmed`
- `upf_volume`

### Replay Paused / Scrubbing

replay 時，前端會從 `_events` 緩衝重建拓樸靜態快照。

對 subscription chain 而言，這代表：

- 過去發生過的 `sbi_call` 可以在 target 時間附近以靜態 edge 呈現
- `upf_volume` 也會在對應時間窗內被重建
- `smf_sub_confirmed` 雖存在於事件集合，但因為沒有 reaction，所以不會出現在拓樸上

### Replay Playing

播放時，這批事件會再次經過 `Topology.react(event)`。

因此 subscription chain 在 replay 播放期間的表現，和 live 收到事件時幾乎相同：

- event log 會再次滾動
- topology 會再次 flash / pulse
- Grafana 不會因此新增任何曲線，因為這條 feature 沒有對應 metrics

## 7. 目前限制

- 整條鏈目前是由多筆獨立事件拼起來的，系統內沒有一個「完整 subscription transaction」物件把它們正式關聯在一起
- `smf_sub_confirmed` 目前只有語意記錄，沒有對應的視覺反應或狀態投影
- `rules/smf.py` 若無法從 `selected_upf_api_root` 對到 `UPF_EES_API_IPS`，會回退成 `"UPF-EES"`；這可能讓多 UPF 情境下的視覺終點失真
- `rules/nwdaf.py` 若無法從資料面 IP 對到 `UPF_DATA_SUBNETS`，`upf_volume.upf` 可能退回原始 IP 字串，導致拓樸找不到對應 node
- 這條功能線目前沒有 metrics，因此無法在 Grafana 端直接量化 subscription 建立成功率、延遲或每段 hop 的統計
