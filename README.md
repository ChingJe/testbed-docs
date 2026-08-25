# Testbed Documentation

本 repository 保存 workspace 內各代 5G testbed 與 `5g-viz` 的設計、操作、實驗及驗證紀錄。
不同分類代表不同系統與時間線，不能互相替代。

## 閱讀入口

- [新版 5G NWDAF Infrastructure](5g-nwdaf-infrastructure/README.md)：目前使用中的
  `5G_NWDAF_Infrastructure` plans、design、operations、experiments 與 records。
- [舊 5G Infrastructure](5g-infra/README.md)：舊 `5G_Infrastructure`、Daisy、replay experiments、
  site-specific settings 與歷史 reports。
- [5g-viz](5g-viz/README.md)：即時觀測、事件、metrics 與 replay visualization 文件。
- `archive/`：已被取代且不再作為目前依據的文件。
- `specs/`：舊 testbed 文件使用的規格節錄與 OpenAPI 參考。

## 文件時間線

### 新版 `5G_NWDAF_Infrastructure`

新版可重現 three-NWDAF testbed、後續 hierarchical FL adaptation 與新實驗工作只記錄在
`5g-nwdaf-infrastructure/`。目前進度入口：

- [Active plans](5g-nwdaf-infrastructure/plans/README.md)
- [HFL component baseline update](5g-nwdaf-infrastructure/plans/hierarchical-federated-learning/component-baseline-update.md)
- [Flat FL foundation history](5g-nwdaf-infrastructure/records/flat-federated-learning/foundation/plan.md)
- [Flat FL validation records](5g-nwdaf-infrastructure/records/flat-federated-learning/validation/README.md)

### 舊 `5G_Infrastructure`

`5g-infra/` 保留舊 VM、Daisy、UPF EES、NWDAF replay、`exp1–67`、testbed runs 與實驗室
network／migration evidence。這些內容可能仍有保存價值，但不代表新版 Infrastructure 的目前
架構、commands 或 component revisions。

### `5g-viz`

`5g-viz/` 依 guides、design、reference、plans 與 notes 分類，保存 visualization system 的
canonical 說明與歷史規劃。

## 結構

```text
testbed-docs/
├── 5g-nwdaf-infrastructure/   新版 NWDAF testbed
│   ├── plans/                 active／upcoming work
│   ├── design/                confirmed architecture
│   ├── operations/            lab-specific operational supplements
│   ├── experiments/           experiment design and acceptance
│   ├── records/               completed evidence, grouped by architecture generation
│   └── archive/               superseded documents
├── 5g-infra/                  舊 5G_Infrastructure 與歷史 experiments
├── 5g-viz/                    visualization system documentation
├── archive/                   repository-level archive
└── specs/                     historical testbed specification references
```

## 新文件放置原則

1. 先判斷文件屬於新版 Infrastructure、舊 testbed 或 `5g-viz`。
2. 未完成工作放入 `plans/`；已確認且仍有效的 architecture 放入 `design/`。
3. 實驗定義與實際 run evidence 分開；前者放 `experiments/`，後者放 `records/`。
4. Source repository 已維護的 code-bound commands 與 configuration reference 不在此複製。
5. 只有已被明確取代的文件才進 `archive/`；單純完成或日期較舊不構成歸檔理由。
