# 5g-viz Design

本目錄收錄 `5g-viz` 目前系統行為的 canonical 設計文件。
這一層偏向 deep technical reference，不是第一次閱讀 `5g-viz` 的建議入口。

第一次接觸這套系統時，先看：

- [`../README.md`](../README.md)
- [`../guides/start-here/README.md`](../guides/start-here/README.md)

歷史規劃與設計探索內容已移至 [`../plans/`](../plans/README.md)。

若需要 deep reference 的主題索引，先看：

- [`../reference/README.md`](../reference/README.md)

## 什麼時候應該讀 `design/*`

適合的情況包括：

- 需要確認某個行為在目前程式碼裡的 canonical 描述
- 需要查 subsystem 邊界、API、schema、runtime 契約
- 需要把 human-facing docs 往下連到更細的實作文件

這一層不處理的事情是：

- onboarding
- 畫面導覽
- first-pass mental model
- 常見現象的 first-line explanation

## 結構

- `overview/`：系統層架構、資料流、event schema
- `backend/`：collector、parser、state、API、metrics、profiles
- `frontend/`：拓樸渲染、event reactions、filter 行為
- `grafana/`：dashboard setup、embed、rendering 行為
- `dvr/`：canonical DVR 執行期行為
- `features/`：端對端功能流程說明
- `reference/`：topology 與設定 schema 參考
