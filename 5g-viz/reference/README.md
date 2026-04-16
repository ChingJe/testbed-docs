# Reference

本目錄收錄 `5g-viz` 的 deep reference 文件入口。

這裡主要整理的是：

- 系統層資料路徑
- 前端與後端的實際行為
- replay / pseudo-live / metrics 的 runtime 細節
- schema、config 與 session artifact 參考

若目前的需求是：

- 理解主畫面怎麼讀
- 理解操作流程
- 理解 live / replay 的使用差異
- 理解常見現象與最常見解釋

較適合先讀：

- [`../guides/start-here/`](../guides/start-here/README.md)
- [`../guides/ui-workflows/`](../guides/ui-workflows/README.md)
- [`../guides/mental-model/`](../guides/mental-model/README.md)
- [`../guides/troubleshooting/common-scenarios.md`](../guides/troubleshooting/common-scenarios.md)

## 什麼問題適合到這裡查

常見情況包括：

- 需要查實際 API、payload、session artifact 或 config schema
- 需要確認某個 runtime 行為在 code path 上怎麼發生
- 需要對照 `main.py`、`events.js`、`state.py`、`metric_player.py` 的設計邊界
- 需要確認某個現象是否已知存在 replay / live 不對稱

## 依問題找文件

### 想看系統層與資料路徑

- [`../design/overview/`](../design/overview/README.md)

適合的問題：

- live / replay 的端到端資料流怎麼走
- event schema 長什麼樣
- 系統啟動時有哪些主要元件

### 想看前端行為

- [`../design/frontend/`](../design/frontend/README.md)

適合的問題：

- Topology 如何套用 reaction 與 snapshot
- event buffer、timeline、scrub、playback 如何運作
- Grafana iframe URL、session 與時間窗如何同步

### 想看後端行為

- [`../design/backend/`](../design/backend/README.md)

適合的問題：

- collector / parser / state / API / metrics 的實際責任邊界
- `state_snapshot` 如何生成
- live metrics、replay backfill、pseudo-live remote write 如何分工

### 想看 DVR / replay

- [`../design/dvr/`](../design/dvr/README.md)

適合的問題：

- session artifact 是什麼
- replay runtime 怎麼啟動
- pseudo-live、pre-seed、一致性邊界是什麼

### 想看 Grafana / Prometheus 整合

- [`../design/grafana/`](../design/grafana/README.md)

適合的問題：

- dashboard 怎麼建
- panel query 與 `session` variable 怎麼切換
- rendering 語意與 live / replay / pseudo-live 有什麼差異

### 想看單一功能流

- [`../design/features/`](../design/features/README.md)

適合的問題：

- 某條 feature flow 如何跨 backend、frontend、Grafana、DVR
- traffic chart、subscription chain、NWDAF ML cycle 的完整路徑

### 想看 schema 與設定

- [`../design/reference/`](../design/reference/README.md)

適合的問題：

- `topology.yaml` 的區塊與 cross-layer 契約
- profile `.env` 和 runtime env 的實際消費位置

## 與 `plans/`、`notes/` 的邊界

- [`../design/`](../design/README.md)：目前系統行為的 canonical reference
- [`../plans/`](../plans/README.md)：歷史規劃、方案探索、當時的決策背景
- [`../notes/`](../notes/README.md)：實作筆記、會議紀錄、內部閱讀材料
