# 5g-viz Design

目前 `5g-viz` 系統的 canonical 設計文件放在這裡。
這些文件應描述「程式碼現在如何運作」；歷史規劃內容已移至 [`../plans/`](../plans/README.md)。

## 結構

- `overview/`：系統層架構、資料流、event schema
- `backend/`：collector、parser、state、API、metrics、profiles
- `frontend/`：拓樸渲染、event reactions、filter 行為
- `grafana/`：dashboard setup、embed、rendering 行為
- `dvr/`：canonical DVR 執行期行為
- `features/`：端對端功能流程說明
- `reference/`：topology 與設定 schema 參考

## 狀態

這個目錄是在舊規劃文件與維護中的設計文件拆分後重新建立的。
目前多數子目錄仍是骨架，後續撰寫時應以程式碼為準，而不是直接複製 plan 文件內容。
