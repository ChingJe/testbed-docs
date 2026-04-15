# docs

Standalone documentation repository for the 5G testbed workspace (`~/testbed`).
Intentionally separate from code repositories so docs can evolve independently.

## Structure

```
docs/
├── 5g-infra/               5G core infrastructure (free5gc + NWDAF + Daisy FL)
│   ├── design/
│   │   ├── daisy/          Daisy federated learning framework design
│   │   └── nwdaf/          NWDAF + ML integration design
│   ├── ops/                Operational runbooks
│   │   ├── environment.md  IP config, MongoDB, network setup, provision flow
│   │   ├── setup-sh.md     .agent/setup.sh internals & rebuild guide
│   │   ├── workflow.md     Day-to-day operations, common commands
│   │   └── troubleshooting.md  Known issues and fixes
│   └── reports/            Incident & bug reports
│
├── 5g-viz/                 Real-time visualisation system (FastAPI + WebSocket + Grafana)
│   ├── design/
│   │   ├── overview/       Canonical system-level docs
│   │   ├── backend/        Canonical server-side docs
│   │   ├── frontend/       Canonical topology and UI docs
│   │   ├── grafana/        Canonical Grafana/Prometheus docs
│   │   ├── dvr/            Canonical DVR behavior docs
│   │   ├── features/       Cross-layer feature walkthroughs
│   │   └── reference/      Schemas and config reference
│   ├── plans/              Historical implementation plans and design explorations
│   │   ├── architecture/   Cross-layer planning docs
│   │   ├── backend/        Backend refactor plans
│   │   ├── frontend/       Frontend planning docs
│   │   ├── grafana/        Grafana integration plans
│   │   └── dvr/            DVR planning and debug notes
│   ├── notes/
│   │   ├── meetings/       Meeting records (YYYY-MM-DD-meeting.md)
│   │   ├── impl/           Implementation notes and decisions
│   │   └── internals/      System internals reading notes
│   └── tmp/                Throwaway test data (not for long-term storage)
│
├── archive/                Retired or superseded documents (for example `archive/5g-viz/`)
└── specs/                  3GPP specifications and OpenAPI YAML files
    ├── TS 23.288/          Analytics, ML model provisioning & monitoring
    ├── TS 23.502/          UPF event exposure
    ├── TS 29.520/          NWDAF service APIs
    ├── TS 29.575/          ADRF data management
    └── yaml/               OpenAPI contracts (Nnwdaf_*, Nadrf_*, Nsmf_*, ...)
```
