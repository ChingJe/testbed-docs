# ⑤⑦ Dynamic Topology Config + Node/Edge Filter

Combined implementation plan for items ⑤ (topology filter) and ⑦ (dynamic topology config).

Core idea: build ⑦ config system first (YAML → API → frontend), then derive ⑤ filter
options from the config data — no hardcoded filter lists.

---

## Current state

Everything is hardcoded in `topology.js`:

| Constant | Purpose |
|---|---|
| `NODE_ID` | Parser NF name → node ID |
| `NODE_LABEL` | Node ID → display text |
| `LABEL_SHORT` | Full interface name → short label |
| `EDGE_STYLE` | Interface name → color/width/duration |
| `elements.nodes` | Node definitions (id, position, parent) |
| zoom / pan | Initial viewport |

---

## Design

### Config file: `topology.yaml`

YAML for human editing; backend reads and serves as JSON via API.

```yaml
nodes:
  - id: consumer
    label: Consumer
    position: { x: 80, y: 180 }
    category: Consumer

  - id: nwdaf
    label: NWDAF
    compound: true          # compound parent node
    category: NWDAF

  - id: nwdaf_anlf
    label: AnLF
    parent: nwdaf
    position: { x: 300, y: 180 }
    category: NWDAF

  - id: nwdaf_mtlf
    label: MTLF
    parent: nwdaf
    position: { x: 420, y: 180 }
    category: NWDAF

  - id: smf
    label: SMF
    position: { x: 360, y: 320 }
    category: Core

  - id: adrf
    label: ADRF
    position: { x: 530, y: 320 }
    category: Storage

  - id: upf_ees
    label: UPF-EES1
    position: { x: 620, y: 140 }
    category: UPF

  - id: upf_ees2
    label: UPF-EES2
    position: { x: 620, y: 240 }
    category: UPF

# Parser NF name -> node ID
nf_aliases:
  Consumer: consumer
  NWDAF: nwdaf
  AnLF: nwdaf_anlf
  MTLF: nwdaf_mtlf
  SMF: smf
  UPF-EES: upf_ees
  UPF-EES2: upf_ees2
  ADRF: adrf

# Edge styles keyed by label used in flashEdge / onSbiCall
edge_styles:
  # --- SBI calls (looked up by onSbiCall) ---
  Nnwdaf_EventsSubscription_Subscribe:
    short: NnwdafSub
    color: "#ce93d8"
    width: 2
    duration: 4000
  Nnwdaf_EventsSubscription_Notify:
    short: NnwdafNotify
    color: "#80cbc4"
    width: 2
    duration: 2000
  Nsmf_EventExposure_Subscribe:
    short: NsmfSub
    color: "#4fc3f7"
    width: 2
    duration: 4000
  Nupf_EventExposure_Subscribe:
    short: NupfSub
    color: "#4fc3f7"
    width: 2
    duration: 4000
  Nupf_EventExposure_Notify:
    short: NupfNotify
    color: "#4db6ac"
    width: 1
    duration: 2000
  # --- Internal edges (used by specific event handlers) ---
  AccuracyReport:
    color: "#00e5ff"
    width: 2
    duration: 4000
  ModelProvision:
    color: "#a5d6a7"
    width: 2
    duration: 4000
  ModelDeploy:
    color: "#a5d6a7"
    width: 2
    duration: 4000
  ThresholdBreach:
    color: "#ff7043"
    width: 2
    duration: 4000
  RetrainTrigger:
    color: "#ff7043"
    width: 2
    duration: 4000
  Nadrf_DataManagement_StorageRequest:
    short: NadrfStore
    color: "#78909c"
    width: 1
    duration: 2000
  Nadrf_DataManagement_RetrievalSubscribe:
    short: NadrfRetSub
    color: "#ff7043"
    width: 2
    duration: 4000
  Nadrf_DataManagement_RetrievalNotify:
    short: NadrfRetNotify
    color: "#ffb74d"
    width: 1
    duration: 4000
  Nadrf_DataManagement_RetrievalRequest:
    short: NadrfRetReq
    color: "#ce93d8"
    width: 1
    duration: 1000

layout:
  zoom: 1.7
  pan_offset: { x: 0, y: 60 }
```

### What stays in JS (not configurable)

- Cytoscape CSS style rules (selectors, animations)
- Event handler logic (`onAccuracy`, `onRetrainTrigger`, etc.)
- Special edge classes (`arc-above`, `arc-above-rev`, `loop-left`)
- `pulse()` / `flashEdge()` animation functions

### API endpoint

```
GET /api/topology-config  →  JSON (parsed from topology.yaml)
```

Backend reads YAML once at startup and caches in memory.

### Loading sequence

```
topology.js:  fetch('/api/topology-config')
              → build cytoscape nodes from config
              → build NODE_ID / LABEL_SHORT / EDGE_STYLE lookups
              → build filter sidebar checkboxes
              → resolve window.TopologyReady

events.js:    TopologyReady.then(() => connect())
```

Global `window.TopologyReady` promise — no ES module conversion needed.

### Filter sidebar layout

```
┌─────────────────────────────────────┐
│ header                    [3 min ↺] │
├──┬──────────────────────────────────┤
│☰ │                                  │
│  │       topology canvas            │
│  │                                  │
├──┴──────────────────────────────────┤
│ charts (Grafana iframe)             │
└─────────────────────────────────────┘

☰ expanded:
┌────────────┬────────────────────────┐
│ Filter     │                        │
│ ──────     │                        │
│ Nodes      │   topology canvas      │
│  ☑ NWDAF   │                        │
│  ☑ Core    │                        │
│  ☑ UPF     │                        │
│  ☑ Storage │                        │
│  ☑ Consumer│                        │
│ ──────     │                        │
│ Edges      │                        │
│  ☑ NnwdafS │                        │
│  ☑ NnwdafN │                        │
│  ☑ NsmfSub │                        │
│  ...       │                        │
│ ──────     │                        │
│ [All] [None│                        │
├────────────┴────────────────────────┤
│ charts                              │
└─────────────────────────────────────┘
```

- Default: collapsed, only ☰ button visible (~24px wide)
- Expanded: ~160px sidebar, dark background matching theme
- Node filter: one checkbox per `category` (toggles all nodes in that category)
- Edge filter: one checkbox per `edge_styles` key (displays `short` label, hover shows full name)
- Show All / Hide All buttons at bottom

### Filter mechanics

- **Node hide**: `cy.$('#nodeId').hide()` — cytoscape automatically hides edges
  where both endpoints are hidden
- **Edge hide by label**: tag each dynamic edge with a `data.edgeType` field,
  then `cy.edges('[edgeType = "X"]').hide()`
- **Compound node**: hiding NWDAF category hides parent + AnLF + MTLF together
  (all share `category: NWDAF`)

---

## File change list

| File | Action | Description |
|---|---|---|
| `topology.yaml` | **new** | YAML config: nodes, nf_aliases, edge_styles, layout |
| `pyproject.toml` | modify | Add `pyyaml` dependency |
| `main.py` | modify | Add `GET /api/topology-config` endpoint (read YAML, cache, serve JSON) |
| `frontend/topology.js` | **rewrite** | Fetch config → build cytoscape → build filter UI → expose `TopologyReady` |
| `frontend/events.js` | modify | Wait for `TopologyReady` before `connect()` |
| `frontend/index.html` | modify | Add filter sidebar container + CSS |

### Detailed changes per file

#### `topology.yaml` (new)

Full content as shown in the schema section above.

#### `pyproject.toml`

Add `"pyyaml>=6.0"` to `dependencies`.

#### `main.py`

```python
import yaml
from pathlib import Path

_topo_config: dict = {}

# In lifespan(), before yield:
_topo_config_path = Path(__file__).parent / "topology.yaml"
with open(_topo_config_path) as f:
    global _topo_config
    _topo_config = yaml.safe_load(f)

@app.get("/api/topology-config")
async def topology_config():
    return _topo_config
```

#### `frontend/topology.js`

Remove all hardcoded constants (`NODE_ID`, `NODE_LABEL`, `LABEL_SHORT`, `EDGE_STYLE`,
inline node elements). Replace with:

1. `fetch('/api/topology-config')` at top
2. Build `NODE_ID` from `config.nf_aliases`
3. Build `EDGE_STYLE` from `config.edge_styles` (each value = `{color, width, duration}`)
4. Build `LABEL_SHORT` from `config.edge_styles` (entries that have a `short` field)
5. Build cytoscape `elements.nodes` array from `config.nodes`
6. Apply `config.layout.zoom` and `config.layout.pan_offset`
7. Build filter sidebar from `config.nodes` categories + `config.edge_styles` keys
8. Resolve `window.TopologyReady`

All existing functions (`pulse`, `flashEdge`, `_syncSummary`, `_deployAnlf`,
`Topology.*` handlers, tooltip handlers) remain unchanged — they reference the
lookup objects which are now built from config instead of hardcoded.

#### `frontend/events.js`

```diff
-connect();
+TopologyReady.then(() => connect());
```

Also guard `ensureChart()` — it already checks `_dashboardUid`, no change needed.

#### `frontend/index.html`

- Add `#filter-toggle` button (absolute-positioned over topology top-left)
- Add `#filter-panel` container (absolute or flex sidebar, initially hidden)
- CSS: dark theme consistent with existing styles, `position: absolute` overlay
  so it doesn't shift the topology canvas layout

---

## Implementation order

1. `topology.yaml` + `pyproject.toml` (add pyyaml) + `uv sync`
2. `main.py` — add endpoint, verify with `curl`
3. `topology.js` — rewrite to fetch config, verify topology renders identically
4. `events.js` — add `TopologyReady` gate
5. `index.html` + `topology.js` filter UI — add sidebar, wire checkboxes

Steps 1-4 should produce zero visual change (same topology, same behavior).
Step 5 adds the filter feature.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Config fetch fails → blank topology | Show error message in topology area; log to console |
| YAML parse error at startup | Log warning, return 500 from endpoint with error detail |
| Compound node hide | Filter by category ensures parent + children toggle together |
| Edge type not in config | `flashEdge` already falls back: `EDGE_STYLE[label] \|\| {color:'#78909c', ...}` |
| Loading order race | `TopologyReady` promise gates WebSocket connection |
