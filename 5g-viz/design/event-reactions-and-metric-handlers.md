# Config-Driven Refactor: Profiles + Event Reactions + Metric Handlers

Three changes that together make 5g-viz fully config-driven. Adding a new NF
or switching experiment environments requires no changes to shared code.

---

## Current state — problems

1. **Adding a new NF** requires touching 5 files across 3 layers (main.py,
   events.js, topology.js, plus the new rule file and topology.yaml).
2. **Switching experiment environments** requires manually swapping root-level
   `topology.yaml` and `.env`.
3. **Event handlers** (`Topology.onXxx`) hardcode node IDs like `nwdaf_anlf`.
4. **Prometheus metrics** are defined and dispatched in `main.py`, not with
   the parser rules that produce the events.

---

## Design overview

| Change | Scope | Goal |
|---|---|---|
| **A. Profiles** | config loading | All per-environment config under `profiles/`; switch via `-p` flag |
| **B. Event reactions** | frontend | YAML-driven event → topology animation mapping |
| **C. Metric handlers** | backend | Per-rule-file metric registration and dispatch |

---

## A. Profiles

### Directory structure

All per-environment config lives under `profiles/`. No config files at root.

```
5g-viz/
├── profiles/
│   ├── default/              ← used when no -p flag given
│   │   ├── topology.yaml
│   │   └── .env
│   ├── lab-a/
│   │   ├── topology.yaml
│   │   └── .env
│   └── lab-b/
│       ├── topology.yaml
│       └── .env
├── .env.example              ← template (stays at root for discoverability)
├── rules/                    ← shared across all profiles
├── frontend/                 ← shared
├── main.py
├── config.py
├── start.sh
└── setup.sh
```

### Profile selection

Single mechanism: `PROFILE` env var. Defaults to `"default"`.

```bash
./start.sh                      # PROFILE=default
./start.sh -p lab-a             # PROFILE=lab-a

PROFILE=lab-a uv run uvicorn main:app --host 0.0.0.0 --port 8765
```

### Resolution rules

| File | Path |
|---|---|
| topology.yaml | `profiles/{PROFILE}/topology.yaml` (required — error if missing) |
| .env | `profiles/{PROFILE}/.env` (required — error if missing) |

No fallback chain. Each profile is self-contained.

### config.py changes

```python
_profile = os.environ.get("PROFILE", "default")
_base = Path(__file__).parent
_env_path = _base / "profiles" / _profile / ".env"
if not _env_path.exists():
    raise FileNotFoundError(
        f"Profile '{_profile}' .env not found: {_env_path}\n"
        f"Run: ./setup.sh -p {_profile}"
    )
load_dotenv(_env_path)
```

### main.py — `_load_topo_config()`

```python
def _load_topo_config() -> dict:
    profile = os.environ.get("PROFILE", "default")
    path = Path(__file__).parent / "profiles" / profile / "topology.yaml"
    if not path.exists():
        raise FileNotFoundError(
            f"Profile '{profile}' topology.yaml not found: {path}"
        )
    with open(path) as f:
        cfg = yaml.safe_load(f) or {}
    cfg["_profile"] = profile
    return cfg
```

### start.sh

```bash
PROFILE="${PROFILE:-default}"
while getopts "p:" opt; do
  case $opt in p) PROFILE="$OPTARG" ;; esac
done
shift $((OPTIND - 1))
export PROFILE

PROFILE_DIR="$SCRIPT_DIR/profiles/$PROFILE"
if [ ! -d "$PROFILE_DIR" ]; then
    echo "[start] ERROR: Profile not found: $PROFILE_DIR"
    echo "        Run: ./setup.sh -p $PROFILE"
    exit 1
fi
echo "[start] Profile: $PROFILE"

# ... rest unchanged ...
```

### setup.sh — profile scaffolding

```bash
PROFILE="${PROFILE:-default}"
while getopts "p:" opt; do
  case $opt in p) PROFILE="$OPTARG" ;; esac
done

PROFILE_DIR="$SCRIPT_DIR/profiles/$PROFILE"
mkdir -p "$PROFILE_DIR"

if [ ! -f "$PROFILE_DIR/.env" ]; then
    cp "$SCRIPT_DIR/.env.example" "$PROFILE_DIR/.env"
    echo "[setup] Created profiles/$PROFILE/.env — edit and re-run"
    exit 0
fi

if [ ! -f "$PROFILE_DIR/topology.yaml" ]; then
    # Copy from default if it exists, otherwise from .env.example equivalent
    if [ -f "$SCRIPT_DIR/profiles/default/topology.yaml" ]; then
        cp "$SCRIPT_DIR/profiles/default/topology.yaml" "$PROFILE_DIR/topology.yaml"
    fi
fi
```

### Frontend header badge

Show profile name when not "default":

```js
if (cfg._profile && cfg._profile !== 'default') {
  const badge = document.createElement('span');
  badge.textContent = cfg._profile;
  badge.style.cssText = 'font-size:0.7rem;color:#78909c;border:1px solid #333;'
    + 'border-radius:3px;padding:1px 6px;';
  document.getElementById('header').insertBefore(
    badge, document.getElementById('live-controls')
  );
}
```

### .gitignore

```gitignore
profiles/*/.env
```

`topology.yaml` in profiles is committed (no secrets).

### Migration

Move existing root files into `profiles/default/`:
- `topology.yaml` → `profiles/default/topology.yaml`
- `.env` → `profiles/default/.env` (gitignored, manual copy needed)

### What varies per profile vs. global

| Per profile | Global (shared) |
|---|---|
| `topology.yaml` (nodes, layout, event_reactions) | `rules/*.py` (parser rules) |
| `.env` (SSH, IPs, Grafana groups) | `prometheus.yml` |
| | Grafana system config |
| | Python dependencies |

Parser rules stay global — rules for absent NFs simply never match.

---

## B. Event reactions in topology.yaml

Move event → topology animation mapping from hardcoded JS to YAML config.

### Atomic actions

| Action | Description |
|---|---|
| `flash_edge` | Animate a temporary edge (from, to, label, classes) |
| `pulse` | Flash a node border (node, color, duration) |
| `add_class` | Add a CSS class to a node (e.g. `retraining`) |
| `remove_class` | Remove a CSS class from a node |

### Full event_reactions definition

```yaml
event_reactions:
  # --- Generic (template variables from event payload) ---
  nf_up:
    - add_class: { node: "{nf}", class: up }

  sbi_call:
    - flash_edge: { from: "{from}", to: "{to}", label: "{label}" }
    - pulse: "{from}"

  upf_volume:
    - flash_edge: { from: "{upf}", to: nwdaf, label: Nupf_EventExposure_Notify }
    - pulse: "{upf}"

  # --- NWDAF ---
  ml_inference:
    - pulse: nwdaf_anlf

  accuracy:
    - flash_edge: { from: nwdaf_anlf, to: nwdaf_mtlf, label: AccuracyReport, classes: arc-above }
    - pulse: nwdaf_anlf

  threshold_breach:
    - flash_edge: { from: nwdaf_mtlf, to: nwdaf_mtlf, label: "ThresholdBreach {n}/{total}" }
    - pulse: { node: nwdaf_mtlf, color: "#ff7043" }

  retrain_trigger:
    - add_class: { node: nwdaf_mtlf, class: retraining }
    - flash_edge: { from: nwdaf_mtlf, to: nwdaf_mtlf, label: RetrainTrigger }
    - pulse: { node: nwdaf_mtlf, color: "#ff7043" }

  retrain_done:
    - remove_class: { node: nwdaf_mtlf, class: retraining }
    - flash_edge: { from: nwdaf_mtlf, to: nwdaf_anlf, label: ModelProvision, classes: arc-above-rev }
    - pulse: { node: nwdaf_mtlf, color: "#69f0ae" }

  model_swap:
    - flash_edge: { from: nwdaf_anlf, to: nwdaf_anlf, label: ModelDeploy, classes: loop-left }
    - pulse: { node: nwdaf_anlf, color: "#69f0ae" }

  # --- ADRF ---
  adrf_stored:
    - flash_edge: { from: nwdaf, to: adrf, label: Nadrf_DataManagement_StorageRequest }
    - pulse: { node: adrf, color: "#78909c" }

  adrf_retrieval_start:
    - flash_edge: { from: nwdaf_mtlf, to: adrf, label: Nadrf_DataManagement_RetrievalSubscribe }
    - pulse: { node: nwdaf_mtlf, color: "#ff7043" }

  adrf_retrieval_notify:
    - flash_edge: { from: adrf, to: nwdaf_mtlf, label: Nadrf_DataManagement_RetrievalNotify }
    - pulse: { node: adrf, color: "#ffb74d" }

  adrf_fetch:
    - flash_edge: { from: nwdaf_mtlf, to: adrf, label: Nadrf_DataManagement_RetrievalRequest }
    - pulse: { node: nwdaf_mtlf, color: "#ce93d8" }
```

### Template syntax

`{field}` is replaced with `event[field]` at runtime.

Node references go through `nf_aliases` lookup, falling back to raw value:
- `"{from}"` → event.from=`"SMF"` → nf_aliases → `"smf"`
- `"nwdaf_anlf"` → nf_aliases miss → `"nwdaf_anlf"` as-is

### Pulse shorthand

```yaml
- pulse: nwdaf_anlf                               # default color/duration
- pulse: "{from}"                                  # template
- pulse: { node: nwdaf_mtlf, color: "#ff7043" }   # custom color
```

### Edge style resolution

`flash_edge` looks up the label (base type, split on space) in `edge_styles`
for color/width/duration. Fallback: `{ color: "#78909c", width: 1, duration: 800 }`.

### What stays in events.js

- `ensureChart()` — Grafana iframe mount on `aggregated_slot` (not a topology concern)
- `appendLog()` — event log
- WebSocket connection

### JS interpreter (topology.js)

```js
let EVENT_REACTIONS = {};  // from config

function _resolve(val, event) {
  if (typeof val !== 'string') return val;
  return val.replace(/\{(\w+)\}/g, (_, k) => event[k] ?? k);
}

function _resolveNode(val, event) {
  const resolved = _resolve(val, event);
  return NODE_ID[resolved] || resolved;
}

function _executeAction(action, event) {
  const [type, params] = Object.entries(action)[0];
  switch (type) {
    case 'flash_edge': {
      const from  = _resolveNode(params.from, event);
      const to    = _resolveNode(params.to, event);
      const label = _resolve(params.label, event);
      const base  = label.split(' ')[0];
      const style = EDGE_STYLE[base] || { color: '#78909c', width: 1, duration: 800 };
      flashEdge(from, to, label, style, params.classes || '');
      break;
    }
    case 'pulse': {
      if (typeof params === 'string')
        pulse(_resolveNode(params, event));
      else
        pulse(_resolveNode(params.node, event), params.color, params.duration);
      break;
    }
    case 'add_class': {
      window._cy.$(`#${_resolveNode(params.node, event)}`).addClass(params.class);
      break;
    }
    case 'remove_class': {
      window._cy.$(`#${_resolveNode(params.node, event)}`).removeClass(params.class);
      break;
    }
  }
}

window.Topology.react = function(event) {
  const reactions = EVENT_REACTIONS[event.type];
  if (!reactions) return;
  for (const action of reactions) _executeAction(action, event);
};
```

### Simplified events.js dispatch

```js
function dispatch(event) {
  if (event.type === 'aggregated_slot') ensureChart();
  Topology.react(event);
  if (event.type !== 'state_snapshot') appendLog(event);
}
```

The 14-line if/else chain is replaced by one line.

---

## C. Metric handlers in rules/*.py

Move Prometheus metrics from `main.py` into the rule file for each NF.

### Convention

Each `rules/<nf>.py` can optionally export:

```python
RULES = [...]                # parser rules (existing)
METRIC_HANDLERS = {           # event_type -> handler (new, optional)
    "event_type": handler_fn,
}
```

### Auto-discovery: rules/__init__.py

```python
ALL_RULES: list[dict] = []
ALL_METRIC_HANDLERS: dict[str, list[Callable]] = {}

for mod_info in pkgutil.iter_modules([str(Path(__file__).parent)]):
    mod = importlib.import_module(f"rules.{mod_info.name}")
    if hasattr(mod, "RULES"):
        ALL_RULES.extend(mod.RULES)
    if hasattr(mod, "METRIC_HANDLERS"):
        for event_type, handler in mod.METRIC_HANDLERS.items():
            ALL_METRIC_HANDLERS.setdefault(event_type, []).append(handler)
```

### Simplified main.py

```python
from rules import ALL_METRIC_HANDLERS

def _update_metrics(event: dict):
    for handler in ALL_METRIC_HANDLERS.get(event.get("type"), []):
        handler(event)
```

All Gauge/Counter definitions move to `rules/nwdaf.py` (see appendix).

---

## After refactor: adding a new NF

| Step | File | What |
|---|---|---|
| 1 | `profiles/*/topology.yaml` | Add node, nf_alias, edge_styles, event_reactions |
| 2 | `rules/<nf>.py` | Add RULES + METRIC_HANDLERS |

**No other files change.**

For a simple NF using only standard SBI calls, step 1 only needs
node + alias (the generic `sbi_call` reaction handles the rest).

## After refactor: adding a new experiment environment

```bash
./setup.sh -p my-experiment   # scaffolds profiles/my-experiment/
vim profiles/my-experiment/.env
vim profiles/my-experiment/topology.yaml
./start.sh -p my-experiment
```

---

## File change list

| File | Action | Description |
|---|---|---|
| `profiles/default/topology.yaml` | **new** (moved from root) | Default topology + event_reactions + panels |
| `profiles/default/.env` | **new** (moved from root) | Default env (gitignored) |
| `topology.yaml` (root) | **delete** | Moved to profiles/default/ |
| `.gitignore` | modify | Add `profiles/*/.env` |
| `config.py` | modify | Read PROFILE env var, load .env from profile dir |
| `main.py` | modify | Read topology.yaml from profile dir; remove metric defs; simplify `_update_metrics()` |
| `start.sh` | modify | Accept `-p` flag, export PROFILE, validate profile dir |
| `setup.sh` | modify | Accept `-p` flag, scaffold profile dir |
| `rules/__init__.py` | modify | Collect ALL_METRIC_HANDLERS |
| `rules/nwdaf.py` | modify | Add metric definitions + METRIC_HANDLERS |
| `frontend/topology.js` | modify | Add reaction interpreter + `Topology.react()`; remove all `Topology.onXxx` + `_deployAnlf()` |
| `frontend/events.js` | modify | Replace dispatch if/else with `Topology.react()` |

### What gets deleted

- `topology.yaml` (root) — moved to `profiles/default/`
- `main.py`: 6 metric variables, `_known_models`, `_update_metrics()` body
- `topology.js`: all `Topology.onXxx` handlers (~80 lines), `_deployAnlf()`
- `events.js`: 14-line if/else dispatch chain

---

## Implementation order

### Phase 1 — Profiles (config restructure, no logic change)

1. Create `profiles/default/`, move `topology.yaml` and `.env` into it
2. Update `config.py` to read from `profiles/{PROFILE}/`
3. Update `main.py` `_load_topo_config()` to read from profile dir
4. Update `start.sh` to accept `-p`, export PROFILE, validate
5. Update `setup.sh` to accept `-p`, scaffold profile dir
6. Update `.gitignore`
7. Verify: `./start.sh` works identically (uses default profile)

### Phase 2 — Metric handlers (backend refactor)

8. Add metric definitions + METRIC_HANDLERS to `rules/nwdaf.py`
9. Update `rules/__init__.py` to collect ALL_METRIC_HANDLERS
10. Simplify `main.py` `_update_metrics()`
11. Verify: `curl /metrics` returns same metrics

### Phase 3 — Event reactions (frontend refactor)

12. Add `event_reactions` section to `profiles/default/topology.yaml`
13. Add `_applyPanelSizes` call from `_initFromConfig` (already done)
14. Add interpreter (`_resolve`, `_resolveNode`, `_executeAction`,
    `Topology.react`) to `topology.js`; load EVENT_REACTIONS from config
15. Replace `events.js` dispatch chain with `Topology.react(event)`
16. Remove all `Topology.onXxx` handlers and `_deployAnlf()`
17. Verify: all event types render identically

Each phase can be committed independently. Phase 1 is a prerequisite for
the others only in that file paths change.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Profile dir missing | `start.sh` + `config.py` + `main.py` all check existence and show actionable error with `setup.sh -p` hint |
| Unresolved `{field}` template | Keeps literal text → visible in edge label, easy to spot |
| Event type not in event_reactions | Silent no-op (same as current unknown events) |
| Node alias not found | Falls back to raw value → works for both alias names and node IDs |
| Metric import order | Rule modules imported at startup; `generate_latest()` collects all |
| Multiple metric handlers for same event | List-based dispatch, all fire |
| `_known_models` cross-event state | Module-level set in rules/nwdaf.py, same lifetime as before |
| Existing users with root .env | `setup.sh` (no -p) creates default profile; README documents migration |

---

## Appendix: rules/nwdaf.py metric additions

```python
from prometheus_client import Counter, Gauge

_ground_truth_ul = Gauge("nwdaf_ground_truth_ul_bytes",
    "Ground truth UL volume per subscription slot", ["sub_id", "target"])
_ground_truth_dl = Gauge("nwdaf_ground_truth_dl_bytes",
    "Ground truth DL volume per subscription slot", ["sub_id", "target"])
_predicted_ul = Gauge("nwdaf_predicted_ul_bytes",
    "NWDAF AnLF predicted UL volume", ["sub_id", "target"])
_predicted_dl = Gauge("nwdaf_predicted_dl_bytes",
    "NWDAF AnLF predicted DL volume", ["sub_id", "target"])
_retrain_total = Counter("nwdaf_retrain_total",
    "Number of MTLF retraining events triggered")
_deviation = Gauge("nwdaf_deviation",
    "NWDAF AnLF model prediction deviation", ["model"])
_known_models: set[str] = set()


def _on_aggregated_slot(event):
    labels = {"sub_id": event["sub_id"], "target": event["target"]}
    _ground_truth_ul.labels(**labels).set(event["ul_vol"])
    _ground_truth_dl.labels(**labels).set(event["dl_vol"])

def _on_ml_inference(event):
    labels = {"sub_id": event["sub_id"], "target": event["target"]}
    _predicted_ul.labels(**labels).set(event["ul_vol"])
    _predicted_dl.labels(**labels).set(event["dl_vol"])

def _on_accuracy(event):
    _known_models.add(event["model"])
    _deviation.labels(model=event["model"]).set(event["deviation"])

def _on_retrain_trigger(event):
    _retrain_total.inc()

def _on_model_swap(event):
    for model in _known_models:
        _deviation.remove(model)
    _known_models.clear()


METRIC_HANDLERS = {
    "aggregated_slot": _on_aggregated_slot,
    "ml_inference":    _on_ml_inference,
    "accuracy":        _on_accuracy,
    "retrain_trigger": _on_retrain_trigger,
    "model_swap":      _on_model_swap,
}
```
