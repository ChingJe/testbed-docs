# 5g-viz Plans

歷史實作規劃與設計探索文件放在這裡。
這些文件保留功能規劃或實作當下的設計脈絡與除錯背景，可作為背景材料，但不應視為目前程式碼的 canonical 描述。

如果需要維護中的系統文件，請優先閱讀 [`../design/`](../design/README.md)。
若需要 deep reference 的主題索引，請看 [`../reference/README.md`](../reference/README.md)。

## 結構

- `architecture/`：跨層架構與資料模型規劃
- `backend/`：後端重構與實作規劃
- `docs/`：文件架構、重構與寫作規則相關規劃
- `frontend/`：前端 UI / topology / interaction 規劃
- `grafana/`：Grafana 與 chart rendering 相關規劃
- `dvr/`：DVR、replay、pseudo-live 與驗證規劃

## 文件相關規劃

- [`docs/5g-viz-restructure-plan.md`](./docs/5g-viz-restructure-plan.md)：第二階段文件重構藍圖，包含資訊架構、phase、mapping、驗收與交接方式

## 近期規劃

- [`architecture/hierarchical-fl-observability-adaptation.md`](./architecture/hierarchical-fl-observability-adaptation.md)：新版 testbed HFL 實驗的 observability adaptation 評估，涵蓋 structured events、scrape／Remote Write 候選路徑、journal／Docker collector、topology、Grafana 與 portable replay
- [`architecture/nwdaf-accuracy-policy-migration.md`](./architecture/nwdaf-accuracy-policy-migration.md)：新版 NWDAF `Accuracy scope / Accuracy policy / Retrain trigger` 對齊方案，涵蓋 parser、metrics、topology、Grafana 與 replay
- [`backend/env-and-startup-config-alignment.md`](./backend/env-and-startup-config-alignment.md)：對齊 `.env.example`、`profiles/default/.env` 與 `start.sh`，清掉過時 env key 與註解，並讓 `WS_PORT` 真的影響啟動 port
- [`dvr/offline-log-to-session-import.md`](./dvr/offline-log-to-session-import.md)：2026-05-07 的後續新增實作規劃，離線將既有 `free5gc.log` / `nwdaf.log` 轉為可 replay session，重點在重用既有 parser、session artifact 與 replay/backfill 流程
- [`frontend/dvr-timeline-time-display.md`](./frontend/dvr-timeline-time-display.md)：2026-05-07 的 DVR 時間軸時間顯示改善規劃，將主顯示從絕對時鐘改成較易理解的相對播放時間
- [`grafana/same-origin-proxy.md`](./grafana/same-origin-proxy.md)：Grafana 經由 `5g-viz:8765` 同源代理的部署與程式修改規劃，用來解決外部無法直連 `:3000` 時的 iframe 載入失敗
