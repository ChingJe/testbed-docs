# topology.yaml

> Historical note: this document still includes parts of the older schema where `topology.yaml` also carried collector SSH/log source definitions. The current runtime keeps topology, event reactions, and frontend-facing behavior here, while collector sources now live in `config.yaml.collector.sources`.

本文定義 `5g-viz` 目前 `topology.yaml` 的 canonical schema 與 cross-layer 契約。

`topology.yaml` 不是純前端設定檔。它目前主要影響：

- frontend 如何建立 Cytoscape 拓樸、filter 與 event reaction
- backend state 如何從同一份 `event_reactions` 重建 `state_snapshot`
- live session 錄製時要複製哪一份 topology config 進 session 目錄

## 1. 載入位置與生命週期

live mode 讀取：

```text
profiles/<PROFILE>/topology.yaml
```

replay mode 讀取：

```text
sessions/<session_id>/topology.yaml
```

也就是說，replay 不會回頭讀目前 profile 的最新版本，而是使用錄製當下保存到 session 內的那一份。

`main.py` 在把 YAML 回傳給前端前，還會額外注入：

```python
cfg["_profile"] = profile
```

這個 `_profile` 欄位不是 YAML 作者要手動寫的 schema，而是後端在 API 回傳時附加的 metadata。

## 2. 頂層區塊

目前 `profiles/default/topology.yaml` 使用下列頂層區塊：

| 區塊 | 用途 | 主要消費者 |
|---|---|---|
| `nodes` | 定義拓樸節點、座標與 compound 關係 | frontend、state |
| `nf_aliases` | parser / event 欄位名稱到 node ID 的映射 | frontend、state |
| `edge_styles` | 邊的顏色、寬度、動畫時間與短標籤 | frontend |
| `layout` | 初始 zoom 與 pan offset | frontend |
| `panels` | topology / charts / event log 的版面比例 | frontend |
| `ssh_sources` | collector 的 SSH 與 log tail 設定 | backend collector |
| `event_reactions` | event type 對應的動作列表 | frontend、state |

目前沒有正式 schema validator。欄位拼錯或缺漏，通常要等到 collector、frontend 或 state 執行時才會暴露。

## 3. `nodes`

`nodes` 是陣列，每個元素至少要有：

| 欄位 | 型別 | 用途 |
|---|---|---|
| `id` | string | node 的 canonical ID，供 frontend、state、reaction、filter 共用 |
| `label` | string | 畫面顯示名稱 |
| `position` | object，可選 | Cytoscape preset layout 的座標，格式為 `{ x, y }` |
| `parent` | string，可選 | compound parent 的 node ID |
| `compound` | boolean，可選 | 目前只作語意標記，前端不直接讀這個欄位 |

範例：

```yaml
nodes:
  - id: nwdaf
    label: NWDAF
    compound: true

  - id: nwdaf_anlf
    label: AnLF
    parent: nwdaf
    position: { x: 300, y: 180 }
```

目前真正決定 compound 關係的是子節點的 `parent` 欄位，而不是 `compound: true`。也就是說：

- `parent` 必須指向另一個已存在的 `nodes[*].id`
- `compound` 少寫通常不會讓前端壞掉，但會失去文件語意上的明示

`id` 是這份檔案裡最重要的穩定識別子。下列地方都直接依賴它：

- `nf_aliases` 的 value
- `event_reactions` 中的 `node` / `from` / `to`
- frontend filter 的 node 可見性狀態
- `state_snapshot.nf_status` 與 `state_snapshot.node_classes`

因此改 node ID 時，通常要同步修改：

- `nodes`
- `nf_aliases`
- `event_reactions`

## 4. `nf_aliases`

`nf_aliases` 是字典，格式為：

```yaml
nf_aliases:
  NWDAF: nwdaf
  AnLF: nwdaf_anlf
  MTLF: nwdaf_mtlf
```

左側 key 對應的是 event payload 中出現的 NF 名稱，右側 value 對應的是 `nodes[*].id`。

這份映射目前同時被兩邊使用：

- frontend `topology.js` 的 `_resolveNode()`
- backend `state.py` 的 `_resolve_node()`

因此只要 reaction 或 event payload 裡出現：

- `nf`
- `from`
- `to`
- 其他會被拿去解 node 的模板欄位

最終都會先經過模板替換，再用 `nf_aliases` 映射到真正的 node ID。

## 5. `edge_styles`

`edge_styles` 是以 edge label 為 key 的字典：

```yaml
edge_styles:
  Nsmf_EventExposure_Subscribe:
    short: NsmfSub
    color: "#4fc3f7"
    width: 2
    duration: 4000
```

每個 entry 目前支援：

| 欄位 | 型別 | 用途 |
|---|---|---|
| `short` | string，可選 | 畫布與 filter sidebar 上顯示的短名稱 |
| `color` | string | edge 顏色 |
| `width` | number | edge 寬度 |
| `duration` | number | live 動畫邊保留多久，單位為毫秒 |

這裡的 key 不只用在顯示，也直接影響：

- `flash_edge` 找樣式時的 lookup key
- filter sidebar 的 edge checkbox 清單
- `_edgeFilter` 的可見性控制

對於動態 label，例如：

```yaml
label: "ThresholdBreach {n}/{total}"
```

frontend 目前會取空白前的第一段當 base edge type，也就是 `ThresholdBreach`。因此：

- `edge_styles` 應宣告 `ThresholdBreach`
- filter 也是以 `ThresholdBreach` 為開關單位

若某條 `flash_edge` 對不到 `edge_styles`，frontend 會回退到預設樣式：

```js
{ color: "#78909c", width: 1, duration: 800 }
```

## 6. `layout` 與 `panels`

### `layout`

`layout` 目前支援：

```yaml
layout:
  zoom: 1.7
  pan_offset: { x: 0, y: 0 }
```

用途：

- `zoom`：前端初始化 Cytoscape 後設定的初始 zoom
- `pan_offset`：在 `cy.center()` 後再額外平移的 offset

若缺漏，frontend 目前使用的預設值是：

- `zoom = 1.7`
- `pan_offset = { x: 0, y: 60 }`

### `panels`

`panels` 目前支援：

```yaml
panels:
  topology_flex: 1
  charts_flex: 1.1
  eventlog_height: 300
```

用途：

- `topology_flex`：上方 topology row 的 flex 值
- `charts_flex`：Grafana charts 區塊的 flex 值
- `eventlog_height`：底部 event log 高度，單位為像素

這些欄位只影響前端初始版面，不會寫回 profile 或 session。

## 7. `ssh_sources`

`ssh_sources` 是 backend collector 的主要契約。格式如下：

```yaml
ssh_sources:
  - name: 5gc
    host_env: SSH_5GC_HOST
    port_env: SSH_5GC_PORT
    user_env: SSH_5GC_USER
    key_env: SSH_5GC_KEY
    logs:
      - dir_env: LOG_FREE5GC_DIR
        filename: free5gc.log
        latest_subdir: true
        source: free5gc
      - path_env: LOG_NWDAF
        source: nwdaf
```

每個 source 目前支援：

| 欄位 | 型別 | 用途 |
|---|---|---|
| `name` | string | collector 的來源名稱，主要供 log 與識別使用 |
| `host_env` / `port_env` / `user_env` / `key_env` | string | 指向 profile `.env` 中實際 SSH 連線參數的變數名稱 |
| `logs` | array | 要 tail 的遠端 log 定義 |

每個 `logs[*]` 目前有兩種寫法：

| 寫法 | 欄位 | 行為 |
|---|---|---|
| 直接指定檔案 | `path_env` | 從 `.env` 取完整路徑，直接 `tail -F` |
| 先找最新子目錄 | `dir_env` + `filename` + `latest_subdir: true` | 先在目錄下找最新子目錄，再 tail 該子目錄中的檔案 |

此外每筆 log 定義都需要：

| 欄位 | 型別 | 用途 |
|---|---|---|
| `source` | string | collector 附加到 log line 的來源 tag，供 parser rule 以 `match.source` 比對 |

這代表 `ssh_sources` 同時決定：

- collector 能否成功建立 SSH 連線
- parser 能否用 `source` 正確區分 free5gc 與 nwdaf log

目前 collector 若找不到 `.env` 內對應值，會回退到一些內建預設，例如：

- `host = 127.0.0.1`
- `port = 22`
- `username = vagrant`

但對實際 testbed 而言，這通常不是正確值，因此問題多半會在執行期才暴露。

## 8. `event_reactions`

`event_reactions` 是由 event type 對應到 action list 的字典：

```yaml
event_reactions:
  retrain_trigger:
    - add_class: { node: nwdaf_mtlf, class: retraining }
    - flash_edge: { from: nwdaf_mtlf, to: nwdaf_mtlf, label: RetrainTrigger }
    - pulse: { node: nwdaf_mtlf, color: "#ff7043" }
```

目前 action 會依陣列順序執行。frontend 支援全部 action；backend `state.py` 只重用 `add_class` / `remove_class`。

### 模板語法

字串欄位可使用：

```text
{field_name}
```

例如：

```yaml
- flash_edge: { from: "{from}", to: "{to}", label: "{label}" }
- add_class: { node: "{nf}", class: up }
```

模板值來自 event payload。若欄位不存在，frontend 與 backend 目前都會保留原樣，例如 `{missing}` 不會自動報錯。

### 支援的 action 類型

#### `flash_edge`

格式：

```yaml
- flash_edge:
    from: "{from}"
    to: "{to}"
    label: "{label}"
    classes: arc-above
```

欄位：

| 欄位 | 必填 | 用途 |
|---|---|---|
| `from` | 是 | 起點 node，可為 node ID、NF alias 或模板 |
| `to` | 是 | 終點 node，可為 node ID、NF alias 或模板 |
| `label` | 是 | edge label，可為模板 |
| `classes` | 否 | 額外套用到 Cytoscape edge 的 class，例如 `arc-above`、`loop-left` |

`from` / `to` 的解析順序是：

1. 先做模板替換
2. 再套用 `nf_aliases`

#### `pulse`

支援兩種格式：

```yaml
- pulse: "{from}"
```

```yaml
- pulse:
    node: nwdaf_mtlf
    color: "#ff7043"
    duration: 300
```

欄位：

| 欄位 | 必填 | 用途 |
|---|---|---|
| `node` | object 寫法時必填 | 要 pulse 的 node |
| `color` | 否 | 邊框高亮顏色 |
| `duration` | 否 | live pulse 動畫時間，單位毫秒 |

`pulse` 只影響暫時視覺效果，不會被寫進 `state_snapshot`。

#### `add_class` / `remove_class`

格式：

```yaml
- add_class: { node: nwdaf_mtlf, class: retraining }
- remove_class: { node: nwdaf_mtlf, class: retraining }
```

欄位：

| 欄位 | 必填 | 用途 |
|---|---|---|
| `node` | 是 | 目標 node，可為 node ID、NF alias 或模板 |
| `class` | 是 | 要加上或移除的 class 名稱，也可用模板 |

這兩種 action 有兩個效果：

- frontend live 模式下立即對 Cytoscape node 加減 class
- backend `state.py` 用同一份設定維護 `state_snapshot.node_classes`

因此這是 `event_reactions` 中唯一具有持久語意的一組 action。

## 9. cross-layer 約束

目前最容易出錯的耦合有三類：

- `nodes[*].id`、`nf_aliases` value、`event_reactions` 的 node 目標必須彼此一致
- `event_reactions.flash_edge.label` 的 base label 應與 `edge_styles` key 對齊
- `ssh_sources.*_env` 與 `logs.*_env` 必須對應到 profile `.env` 內真實存在的變數名稱

另外還有兩個現況邊界：

- session 錄製時會複製 `topology.yaml`，因此修改目前 profile 不會 retroactively 改變舊 session 的 replay 結果
- 這份 schema 目前沒有集中驗證器，錯誤多半在 runtime 才會被發現
