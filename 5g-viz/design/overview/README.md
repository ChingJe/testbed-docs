# Overview

本目錄收錄 `5g-viz` 系統層的 canonical 文件。

適合的情況包括：

- 需要從 subsystem 角度確認 live / replay 資料路徑
- 需要查 event schema、架構圖與系統層責任分工

若目前需求是先理解畫面或操作，先讀：

- [`../../start-here/`](../../start-here/README.md)
- [`../../mental-model/`](../../mental-model/README.md)

目前文件：

- [system.md](system.md)：系統組件、執行模式、啟動流程與對外介面
- [data-flow.md](data-flow.md)：live / replay 兩條資料路徑的端對端流程
- [event-schema.md](event-schema.md)：事件欄位、Prometheus metric 映射與 `state_snapshot` 結構
- [architecture.md](architecture.md)：系統整體組件圖與 live / replay 資料流（Mermaid）
