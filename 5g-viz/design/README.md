# 5g-viz Design

本目錄收錄 `5g-viz` 目前系統行為的 canonical 設計文件。
這一層偏向 deep technical reference，不是第一次閱讀 `5g-viz` 的建議入口。

如果你是第一次接觸這套系統，先看：

- [`../README.md`](../README.md)
- [`../start-here/README.md`](../start-here/README.md)

歷史規劃與設計探索內容已移至 [`../plans/`](../plans/README.md)。

## 結構

- `overview/`：系統層架構、資料流、event schema
- `backend/`：collector、parser、state、API、metrics、profiles
- `frontend/`：拓樸渲染、event reactions、filter 行為
- `grafana/`：dashboard setup、embed、rendering 行為
- `dvr/`：canonical DVR 執行期行為
- `features/`：端對端功能流程說明
- `reference/`：topology 與設定 schema 參考
