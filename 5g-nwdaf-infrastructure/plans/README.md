# Plans

本目錄只保存尚未完成或即將開始的新版 NWDAF testbed 工作。

每份計畫至少應列出：

- 狀態與日期；
- 目標、範圍與不包含項目；
- 受影響的 repositories、components、configs 與環境；
- acceptance criteria 與 verification matrix；
- 明確延後的工作；
- implementation／review／commit／testbed validation 進度。

目前 workstream：

- [Protocol-driven Hierarchical FL experiment](protocol-driven-hierarchical-fl-experiment/README.md)
  - [Multi-host Branch Replacement Experiment Plan](protocol-driven-hierarchical-fl-experiment/multi-host-branch-replacement-experiment-plan.md)
  - [正式 Branch Replacement 比較實驗計畫草案](protocol-driven-hierarchical-fl-experiment/formal-branch-replacement-comparison-plan.md)

前置 workstream（供 migration provenance，不表示目前 feature branch 必須維持舊 experiment compatibility）：

- [Flat／Hierarchical testbed scenario migration](hierarchical-federated-learning/flat-hierarchical-scenario-migration.md)
- [Phase 1 production Flat config migration](hierarchical-federated-learning/phase-1-production-flat-config-migration.md)
- [Phase 2 static scenario common foundation](hierarchical-federated-learning/phase-2-static-scenario-common-foundation.md)
- [Phase 3 static Flat flow](hierarchical-federated-learning/phase-3-static-flat-flow.md)
- [Phase 4 static Hierarchical flow](hierarchical-federated-learning/phase-4-static-hierarchical-flow.md)

已完成計畫不應長期和 active work 混在本目錄。完成後保留必要的最終設計，並把執行結果與
evidence 整理至 `records/`；已被取代且無目前效力的計畫移入 `archive/`。
