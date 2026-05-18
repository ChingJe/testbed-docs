# Subscription Chain

本文從功能角度描述 `5g-viz` 如何把「Consumer 訂閱 NWDAF analytics，並一路串到 SMF、UPF，再回到 Consumer」這條鏈路轉成可觀察的事件與拓樸反應。

若要看子系統細節，可再對照：

- [`../overview/event-schema.md`](../overview/event-schema.md)
- [`../backend/parser.md`](../backend/parser.md)
- [`../frontend/topology.md`](../frontend/topology.md)
- [`../backend/profiles.md`](../backend/profiles.md)
- [`../reference/topology-yaml.md`](../reference/topology-yaml.md)

## 1. 功能範圍

這條功能線關心的是 subscription control path 與其後續通知鏈，包含：

- Consumer 對 NWDAF 建立 analytics subscription
- NWDAF 對 SMF 建立事件訂閱
- SMF 對對應的 UPF 建立 EES subscription
- UPF 持續把資料或事件 notify 回 NWDAF
- NWDAF 再把 analytics notification 送回 Consumer

它主要影響：

- event log
- topology 動畫與 pulse
- session recording / replay

它目前不直接產生 Prometheus metrics，也不直接改變 Grafana 主圖。

## 2. 鏈路總覽

使用者目前可觀察到的主鏈路可整理成：

```text
Consumer
  -> NWDAF   (CreateSubscription)
  -> SMF     (EventExposure Subscribe)
  -> UPF     (EES subscription)
  -> NWDAF   (volume / event notify)
  -> Consumer (analytics notification)
```

其中前四段不完全來自同一個 event type：

- `Consumer -> NWDAF`
- `NWDAF -> SMF`
- `SMF -> UPF`
- `NWDAF -> Consumer`

目前都以 `sbi_call` 表示。

而：

- `UPF -> NWDAF`

目前則主要由 `upf_volume` 表示。

## 3. 事件如何把這條鏈路拼起來

### Consumer -> NWDAF

`rules/nwdaf_sub.py` 會把 NWDAF 收到的 CreateSubscription log 轉成：

```json
{
  "type": "sbi_call",
  "from": "Consumer",
  "to": "NWDAF",
  "label": "Nnwdaf_EventsSubscription_Subscribe"
}
```

這是整條鏈在 UI 上的起點。

### NWDAF -> SMF

當 NWDAF 決定向 SMF 建立 UE session event 訂閱時，會產生：

```json
{
  "type": "sbi_call",
  "from": "NWDAF",
  "to": "SMF",
  "label": "Nsmf_EventExposure_Subscribe"
}
```

### SMF -> UPF

`rules/smf.py` 在偵測到 UPF subscription 建立成功時，會產生另一筆 `sbi_call`：

```json
{
  "type": "sbi_call",
  "from": "SMF",
  "to": "UPF-EES",
  "label": "Nupf_EventExposure_Subscribe"
}
```

這裡的 `to` 並不是固定寫死，而是由 `selected_upf_api_root` 經 profile `config.yaml` 內的 `mappings.upf_ees_api_ips` 映射得出。

### SMF 訂閱確認

NWDAF 之後還會產生一筆：

```json
{
  "type": "smf_sub_confirmed"
}
```

這筆事件會被錄進 event log 與 session，但目前：

- 沒有 topology reaction
- 沒有 metrics handler

因此它是語意事件，不是可見動畫的一部分。

### UPF -> NWDAF

subscription chain 建立後，UPF 持續回報的 notify 目前主要會被解析成：

```json
{
  "type": "upf_volume",
  "upf": "UPF-EES",
  "ip": "10.10.0.2",
  "ul_bytes": 123,
  "dl_bytes": 456
}
```

其中 `upf` 不是直接從 node id 取得，而是由資料面 IP 經過 profile `config.yaml` 內的 `mappings.upf_data_subnets` 映射得出。

### NWDAF -> Consumer

當 NWDAF 把 analytics notification 送回 Consumer 時，會再產生一筆：

```json
{
  "type": "sbi_call",
  "from": "NWDAF",
  "to": "Consumer",
  "label": "Nnwdaf_EventsSubscription_Notify"
}
```

這讓整條鏈在 topology 上能回到 Consumer。

## 4. 前端為什麼能把它畫成一條鏈

### `sbi_call` 的 generic reaction

目前 topology 對 `sbi_call` 的 reaction 是 generic 的：

- flash edge
- pulse source node

因此只要 event payload 有：

- `from`
- `to`
- `label`

前端就能用同一套 reaction 畫出：

- Consumer -> NWDAF
- NWDAF -> SMF
- SMF -> UPF
- NWDAF -> Consumer

### `upf_volume` 的專用 reaction

`UPF -> NWDAF` 這段不是 `sbi_call`，所以它走 `upf_volume` 的專用 reaction。

這讓使用者能看見「訂閱建立完成後，UPF 持續把資料 notify 回 NWDAF」的後續資料路徑。

## 5. 這條功能線的 cross-layer 契約

### `nf_aliases` 必須能解析 `from` / `to`

`sbi_call` 內的 `from`、`to` 最後會透過 topology `nf_aliases` 解析成真正的 node id。

因此下列名稱必須可被解析：

- `Consumer`
- `NWDAF`
- `SMF`
- `UPF-EES`
- `UPF-EES2`

### `edge_styles` 必須包含對應 label

subscription chain 目前依賴的 label 主要有：

- `Nnwdaf_EventsSubscription_Subscribe`
- `Nnwdaf_EventsSubscription_Notify`
- `Nsmf_EventExposure_Subscribe`
- `Nupf_EventExposure_Subscribe`
- `Nupf_EventExposure_Notify`

若 `edge_styles` 漏掉其中某條，前端仍可能畫出 edge，但只會回退成預設樣式。

### profile mapping 會改變鏈路終點

這條功能線不完全由 topology 決定，還會受 profile `config.yaml` 內的 mapping 影響：

- `mappings.upf_ees_api_ips`
  - 影響 `SMF -> UPF` 這段 `sbi_call.to`
- `mappings.upf_data_subnets`
  - 影響 `UPF -> NWDAF` 這段 `upf_volume.upf`

也就是說，就算 nodes 與 aliases 沒改，只要 mapping 改了，拓樸上的 UPF 終點也可能改變。

## 6. 與 DVR / Replay 的關係

這條功能線不直接進 Prometheus，但仍完整參與 DVR。

### recording

live mode 下，這些事件都會被寫進 session `events.jsonl`。

因此錄製中會保留：

- 各段 `sbi_call`
- `smf_sub_confirmed`
- `upf_volume`

### replay paused / scrubbed

replay 時，前端會從 `_events` 重建拓樸靜態快照。

對 subscription chain 而言，這代表：

- 過去發生過的 `sbi_call` 可以在對應時間點附近以靜態 edge 呈現
- `upf_volume` 也會依 active range 被重建
- `smf_sub_confirmed` 仍存在於事件集合，但因沒有 reaction，所以不會出現在 topology 上

### replay playing

播放時，這批事件會再次經過 `Topology.react(event)`。

因此 subscription chain 在 replay `PLAYING` 的表現和 live 幾乎相同：

- event log 會再次滾動
- topology 會再次 flash / pulse
- Grafana 不會因此新增曲線，因為這條 feature 沒有對應 metrics

## 7. 目前限制

- 整條鏈目前由多筆獨立事件拼起來，系統內沒有一個完整 subscription transaction 物件
- `smf_sub_confirmed` 目前只有語意記錄，沒有對應的視覺投影
- `rules/smf.py` 若無法從 `selected_upf_api_root` 對到 `mappings.upf_ees_api_ips`，可能回退到預設 UPF 名稱
- `rules/nwdaf.py` 若無法從資料面 IP 對到 `mappings.upf_data_subnets`，`upf_volume.upf` 可能退回原始 IP 字串
- 這條功能線目前沒有 metrics，因此無法在 Grafana 端直接量化 subscription 建立成功率、延遲或 hop 統計

若看到本文提到 profile `.env` 映射，應理解為舊版本配置模型；目前對應設定已收斂到 `config.yaml`。
