# 5g-viz Design

Canonical design documents for the current `5g-viz` system live here.
These files should describe how the code works today. Historical planning
material has been moved to [`../plans/`](../plans/README.md).

## Structure

- `overview/`: system-level architecture, data flow, event schema
- `backend/`: collector, parser, state, API, metrics, profiles
- `frontend/`: topology rendering, event reactions, filter behavior
- `grafana/`: dashboard setup, embedding, rendering behavior
- `dvr/`: canonical DVR runtime behavior
- `features/`: end-to-end feature walkthroughs
- `reference/`: topology/config schema reference

## Status

This tree has been recreated after separating old planning docs from maintained
design docs. Most subdirectories currently contain stubs and should be filled
from code, not by copying plan text verbatim.
