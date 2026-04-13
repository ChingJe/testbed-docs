# Config-driven topology 流程說明

整個 5g-viz 的核心思路：**所有「什麼 event 要讓哪個節點動、走哪條邊」的知識，全部集中在 `topology.yaml` 裡，後端和前端的程式碼只負責執行，不硬寫任何業務邏輯。**

---

## Cytoscape.js 是什麼

Cytoscape.js 是一套跑在瀏覽器裡的圖形視覺化函式庫，專門處理「節點 + 邊」這種結構（graph）。5g-viz 用它來畫 NF 拓樸圖。

使用方式：把節點和邊的資料（id、位置、顏色等）傳進去，Cytoscape 負責渲染、縮放、拖曳；程式碼只需要呼叫 `cy.add()`、`cy.$('#id').addClass()` 之類的 API 操作元素。

Cytoscape 內部用 **id** 定位所有節點和邊，**label** 只是顯示用的字串，兩者完全分開。

---

## 系統啟動流程

```
./start.sh
  │
  ├─ 啟動 Prometheus
  └─ uvicorn main:app
       │
       ├─ lifespan: _load_topo_config()   ← 讀 profiles/{PROFILE}/topology.yaml
       ├─ lifespan: grafana_setup.setup() ← 自動建 Grafana datasource/dashboard
       ├─ asyncio task: collector.start() ← SSH tail VM logs
       └─ asyncio task: _process_queue() ← parse log lines, broadcast events

瀏覽器開啟
  │
  └─ topology.js: fetch('/api/topology-config')
       │
       └─ _initFromConfig(cfg)
            ├─ 建 NODE_ID / EDGE_STYLE / EVENT_REACTIONS lookup tables
            ├─ 用 cfg.nodes 建立 Cytoscape 節點
            └─ 建 filter sidebar
```

---

## topology.yaml：唯一的設定來源

```yaml
# ── 1. 節點 ──────────────────────────────────────────────────
nodes:
  - id: consumer          # Cytoscape 內部 ID（程式碼用這個定位節點）
    label: Consumer       # 畫面上顯示的字串（可以和 id 不同）
    position: { x: 80, y: 180 }

  - id: nwdaf
    label: NWDAF
    compound: true        # compound node = 可以包含子節點的「框」

  - id: nwdaf_anlf
    label: AnLF
    parent: nwdaf         # 宣告父節點，Cytoscape 會把它畫在 nwdaf 框裡面

# ── 2. NF 名稱對應 ────────────────────────────────────────────
nf_aliases:
  Consumer: consumer      # 左邊：parser 在 event 裡寫的 NF 名稱
  NWDAF: nwdaf            # 右邊：對應的 Cytoscape node id（必須和 nodes[*].id 一致）
  AnLF: nwdaf_anlf        # 注意：nodes 裡 AnLF 的 label 也是 "AnLF"，但那是巧合
                          # label 純粹是畫面顯示字串，nf_aliases 的 key 看的是 parser 吐的字串
                          #
                          # 綁定關係：
                          #   rules/*.py build() 裡寫的 "from"/"to"/"nf" 字串
                          #     → 必須對應 nf_aliases 的 key（左邊）
                          #   nf_aliases 的 value（右邊）
                          #     → 必須對應 nodes[*].id

# ── 3. 邊的樣式 ───────────────────────────────────────────────
edge_styles:
  Nnwdaf_EventsSubscription_Subscribe:
    short: NnwdafSub      # canvas 縮寫（hover 才顯示全名）
    color: "#ce93d8"
    width: 2
    duration: 4000        # 這條邊幾毫秒後消失

# ── 4. 事件對應的視覺動作 ──────────────────────────────────────
event_reactions:
  sbi_call:
    - flash_edge: { from: "{from}", to: "{to}", label: "{label}" }
    #                       ↑ 大括號語法：執行時替換成 event payload 的欄位值
    - pulse: "{from}"

  retrain_trigger:
    - add_class: { node: nwdaf_mtlf, class: retraining }
    #              ↑ 這裡直接寫 node ID，不需要替換
    - flash_edge: { from: nwdaf_mtlf, to: nwdaf_mtlf, label: RetrainTrigger }
    - pulse: { node: nwdaf_mtlf, color: "#ff7043" }
```

---

## 後端：讀 YAML、提供 API、廣播 event

### `main.py` — 讀 YAML

```python
def _load_topo_config() -> dict:
    profile = os.environ.get("PROFILE", "default")
    # PROFILE 由 start.sh 透過 -p flag 設定，預設是 "default"

    path = Path(__file__).parent / "profiles" / profile / "topology.yaml"
    with open(path) as f:
        cfg = yaml.safe_load(f) or {}

    cfg["_profile"] = profile  # 前端用來判斷是否顯示 profile 標記
    return cfg
```

```python
@app.get("/api/topology-config")
async def topology_config():
    return _topo_config  # 直接把 YAML 解析成 dict 回傳，FastAPI 自動序列化成 JSON
```

**為什麼不讓前端直接讀 YAML？**
瀏覽器沒辦法直接 parse YAML，而且讓後端統一管理路徑和 profile 切換比較乾淨。後端只是傳話，不做額外的資料轉換。

---

### `rules/*.py` — log → event dict

`parser.py` 先用 `BASE` regex 把每行 log 拆成結構化欄位（`base` dict），再逐條比對 `ALL_RULES`，第一條匹配的 rule 決定產出什麼 event。

log 行的格式（free5gc / nwdaf 共用）：
```
time="2026-04-10T10:23:45Z" level="info" msg="Subscribing to SMF: ..." CAT="Consumer" NF="NWDAF" ...
```

`parser.py` 拆出的 `base` dict：
```python
{
  "time":     "2026-04-10T10:23:45Z",
  "level":    "info",
  "msg":      "Subscribing to SMF: ...",
  "cat":      "Consumer",   # CAT= 欄位
  "nf":       "NWDAF",      # NF= 欄位
  "source":   "nwdaf",      # collector 標的，"free5gc" 或 "nwdaf"
  "extra_kv": { ... },      # 其餘 key="value" 欄位，用 dict 存
}
```

---

每條 rule 是一個 dict，有三個欄位：

#### `match`：過濾條件（AND 關係，沒列的欄位不檢查）

| key | 比對對象 | 說明 |
|---|---|---|
| `nf` | `base["nf"]` | log 行的 `NF=` 欄位，例如 `"NWDAF"`、`"SMF"` |
| `cat` | `base["cat"]` | log 行的 `CAT=` 欄位，例如 `"Consumer"`、`"AnLF"`、`"MTLF"` |
| `source` | `base["source"]` | collector 來源，`"free5gc"` 或 `"nwdaf"` |
| `msg` | `base["msg"]`（或整行） | compiled regex，`search()` 比對，可用具名 group 抓取欄位值 |

`msg` 的 regex 用 `search()`（不是 `fullmatch`），所以只需要匹配 msg 字串的一部分。

#### `event`：輸出的 event type 字串

對應 `topology.yaml` 的 `event_reactions` key，也是 `METRIC_HANDLERS` 的 key。

#### `build`：產出 event payload

`lambda m, base -> dict`
- `m`：`match.msg` regex 的 match object，用 `m.group("name")` 取具名 group
- `base`：完整的 log 欄位 dict（如果需要 `extra_kv` 裡的欄位可以從這裡拿）
- 回傳的 dict 會和 `{"type": ..., "time": ...}` 合併成最終的 event dict

---

以 `nwdaf_sub.py` 的一條 rule 為例：

```python
{
    "match": {
        "nf": "NWDAF",      # 只處理 NF=NWDAF 的行
        "cat": "Consumer",  # 且 CAT=Consumer
        "msg": re.compile(
            r"Subscribing to SMF: endpoint=\S+ supi=(?P<supi>\S+) notifId=(?P<corr_id>\S+)"
            # (?P<supi>...) 是 Python named group，之後用 m.group("supi") 取值
        ),
    },
    "event": "sbi_call",
    "build": lambda m, base: {
        "from": "NWDAF",                        # 對應 nf_aliases 的 key
        "to": "SMF",
        "label": "Nsmf_EventExposure_Subscribe",
        "supi": m.group("supi").rstrip(","),    # 從 regex match 取值
    },
},
```

產出的 event dict：
```json
{
  "type": "sbi_call",
  "time": "2026-04-10T10:23:45Z",
  "from": "NWDAF",
  "to": "SMF",
  "label": "Nsmf_EventExposure_Subscribe",
  "supi": "imsi-208930000000001"
}
```

---

### `main.py` — metrics 更新 + WebSocket 廣播

```python
async def _process_queue():
    while True:
        item = await _queue.get()         # collector 把 log 行放進 queue
        event = parser.parse_line(...)    # 比對 RULES，產出 event dict（或 None）
        if event:
            _update_metrics(event)        # 更新 Prometheus gauges/counters
            await _broadcast(event)       # 序列化成 JSON，推給所有 WebSocket 連線

async def _broadcast(event: dict):
    msg = json.dumps(event)
    for ws in _clients:
        await ws.send_text(msg)
```

**WebSocket 的角色**：伺服器主動推送（push），瀏覽器不需要輪詢。每條 log 行解析完就立刻送到所有連線的瀏覽器，延遲通常在毫秒以內。

---

## 前端：初始化 + 收 event + 執行動作

### `topology.js` — 啟動時 fetch config

```js
// 頁面載入時立刻發這個 fetch
// TopologyReady 是一個 Promise，events.js 等它 resolve 後才建立 WebSocket
window.TopologyReady = fetch('/api/topology-config')
  .then(r => r.json())
  .then(cfg => _initFromConfig(cfg));

function _initFromConfig(cfg) {
  // 把三個 lookup table 存到 module-level 變數，之後收到 event 時查用

  NODE_ID = cfg.nf_aliases || {};
  // 例：{ "Consumer": "consumer", "NWDAF": "nwdaf", "AnLF": "nwdaf_anlf" }

  EDGE_STYLE = {};
  LABEL_SHORT = {};
  for (const [label, style] of Object.entries(cfg.edge_styles || {})) {
    EDGE_STYLE[label] = { color: style.color, width: style.width, duration: style.duration };
    if (style.short) LABEL_SHORT[label] = style.short;  // canvas 縮寫
  }

  EVENT_REACTIONS = cfg.event_reactions || {};
  // 例：{ "sbi_call": [{flash_edge: ...}, {pulse: ...}], "retrain_trigger": [...] }

  _initCytoscape(cfg.nodes, cfg.layout);
  _buildFilterPanel(cfg);
}
```

---

### `events.js` — 收到 WebSocket event 後分派

```js
function dispatch(event) {
  const t = event.type;

  // 特例：第一個 aggregated_slot 才掛載 Grafana iframe
  if (t === 'aggregated_slot') ensureChart();

  // 其他所有 topology 動作全部交給 Topology.react
  Topology.react(event);

  // 除了 state_snapshot（初始狀態快照）以外都記到 event log
  if (t !== 'state_snapshot') appendLog(event);
}
```

`events.js` 本身完全不知道「某種 event 要讓哪個節點動」，這些知識在 YAML 裡，由 `Topology.react` 負責查和執行。

---

### `topology.js` — react：查 YAML → 逐一執行 action

```js
window.Topology = {
  react(event) {
    const reactions = EVENT_REACTIONS[event.type];
    // 例：event.type = "sbi_call"
    // reactions = [
    //   { flash_edge: { from: "{from}", to: "{to}", label: "{label}" } },
    //   { pulse: "{from}" }
    // ]

    if (!reactions) return;  // 這種 event 沒有設定 reactions，忽略
    for (const action of reactions) _executeAction(action, event);
  }
};
```

---

### `_resolve` 和 `_resolveNode`：兩步解析

**`_resolve`**：把 `{field}` 替換成 event payload 的值

```js
function _resolve(val, event) {
  if (typeof val !== 'string') return val;
  return val.replace(/\{(\w+)\}/g, (_, k) => {
    return event[k] != null ? event[k] : `{${k}}`;
    // event[k] 存在就替換，不存在就原樣保留（方便 debug）
  });
}

// 例：
// val = "{from}",  event = { from: "NWDAF", to: "SMF", ... }
// 結果 = "NWDAF"
```

**`_resolveNode`**：在 `_resolve` 之後，再用 `nf_aliases` 把 NF 名稱轉成 Cytoscape node ID

```js
function _resolveNode(val, event) {
  const resolved = _resolve(val, event);  // 步驟一：替換 {field}
  return NODE_ID[resolved] || resolved;   // 步驟二：nf_aliases 查表
  // NODE_ID["NWDAF"] = "nwdaf"（YAML 裡的 Cytoscape node ID）
  // 如果查不到就原樣回傳（可能本來就是 node ID，例如 "nwdaf_mtlf"）
}

// 例：val = "{from}", event.from = "NWDAF"
// _resolve → "NWDAF"
// NODE_ID["NWDAF"] → "nwdaf"   ← Cytoscape 認識這個 ID
```

為什麼要兩步？Parser 產出的 NF 名稱（`"Consumer"`, `"NWDAF"`, `"AnLF"`）是 log 裡的原始字串，Cytoscape 用的 node ID 是我們自己定義的（`"consumer"`, `"nwdaf"`, `"nwdaf_anlf"`）。`nf_aliases` 就是這兩個世界之間的對照表。

`nf_aliases` 左右兩邊的對應關係：

```
rules/*.py build()        nf_aliases 左邊 key        nf_aliases 右邊 value        nodes
──────────────────        ───────────────────        ─────────────────────        ─────
"from": "AnLF"    ──→     AnLF（必須一致）     ──→    nwdaf_anlf（必須一致）  ──→  id: nwdaf_anlf
```

`nodes[*].label`（`"AnLF"`）和 `nf_aliases` key（`"AnLF"`）字串相同只是慣例，兩者沒有程式上的綁定。改 `label` 不影響任何邏輯；改 `id` 的話 `nf_aliases` 右邊也要跟著改。

---

### `_executeAction`：action type → 實際 Cytoscape 操作

```js
function _executeAction(action, event) {
  const [type, params] = Object.entries(action)[0];
  // action = { flash_edge: { from: "...", to: "...", label: "..." } }
  // type = "flash_edge",  params = { from: "...", to: "...", label: "..." }

  switch (type) {

    case 'flash_edge': {
      const from  = _resolveNode(params.from, event);   // NF名稱 → node ID
      const to    = _resolveNode(params.to, event);
      const label = _resolve(params.label, event);       // {label} → 實際介面名稱
      const base  = label.split(' ')[0];                 // "ThresholdBreach 2/3" → "ThresholdBreach"
      const style = EDGE_STYLE[base] || { color: '#78909c', width: 1, duration: 800 };
      // EDGE_STYLE 用完整介面名稱查，取得顏色和消失時間
      flashEdge(from, to, label, style, params.classes || '');
      break;
    }

    case 'pulse': {
      // pulse 支援兩種寫法：
      //   字串：  pulse: "{from}"
      //   object：pulse: { node: nwdaf_mtlf, color: "#ff7043", duration: 300 }
      if (typeof params === 'string') {
        pulse(_resolveNode(params, event));
      } else {
        pulse(_resolveNode(params.node, event), params.color, params.duration);
      }
      break;
    }

    case 'add_class':
      window._cy.$(`#${_resolveNode(params.node, event)}`).addClass(params.class);
      // 例：nwdaf_mtlf 加上 "retraining" class → Cytoscape CSS 讓邊框變橘紅色
      break;

    case 'remove_class':
      window._cy.$(`#${_resolveNode(params.node, event)}`).removeClass(params.class);
      break;
  }
}
```

---

## 完整範例：`retrain_trigger` event 從頭到尾

```
nwdaf.log 出現：
  [MTLF] Submitting training task to Daisy: model=anlf-v2 tid=tid-abc

    ↓ collector.py SSH tail → 放進 queue

    ↓ parser.py 比對 rules/nwdaf.py RULES

event dict：
  { "type": "retrain_trigger", "model": "anlf-v2", "tid": "tid-abc", "time": "..." }

    ↓ main.py _update_metrics()
  METRIC_HANDLERS["retrain_trigger"] → _retrain_total.inc()   （Prometheus counter +1）

    ↓ main.py _broadcast()
  WebSocket → 瀏覽器

    ↓ events.js dispatch() → Topology.react(event)

    ↓ EVENT_REACTIONS["retrain_trigger"]：
  [
    { add_class:  { node: "nwdaf_mtlf", class: "retraining" } },
    { flash_edge: { from: "nwdaf_mtlf", to: "nwdaf_mtlf", label: "RetrainTrigger" } },
    { pulse:      { node: "nwdaf_mtlf", color: "#ff7043" } }
  ]

    ↓ _executeAction 逐一執行：
  1. cy.$("#nwdaf_mtlf").addClass("retraining")
     → CSS: border-color: #ff7043, border-width: 3  （節點邊框變橘紅，持續到 retrain_done）

  2. flashEdge("nwdaf_mtlf", "nwdaf_mtlf", "RetrainTrigger", { color: "#ff7043", duration: 4000 })
     → Cytoscape 加一條 self-loop 邊，4 秒後自動移除

  3. pulse("nwdaf_mtlf", "#ff7043")
     → 節點邊框快速閃爍一次
```

後續 `retrain_done` event 到來時，`remove_class: { node: nwdaf_mtlf, class: retraining }` 把橘紅邊框還原。整個 retrain 的視覺週期（開始→結束）都在 YAML 裡定義，程式碼完全不知道 retrain 的存在。
