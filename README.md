# docs

`~/testbed` 的 5G 測試床獨立文件庫。
刻意與程式碼儲存庫分離，讓文件可以獨立演進。

## 結構

```
docs/
├── 5g-infra/               5G 核心基礎設施（free5gc + NWDAF + Daisy FL）
│   ├── design/
│   │   ├── daisy/          Daisy 聯邦式學習框架設計
│   │   └── nwdaf/          NWDAF 與 ML 整合設計
│   ├── ops/                維運 runbook
│   │   ├── environment.md  IP 設定、MongoDB、網路配置、佈署流程
│   │   ├── setup-sh.md     `.agent/setup.sh` 內部機制與重建指南
│   │   ├── workflow.md     日常操作流程與常用指令
│   │   └── troubleshooting.md  已知問題與排查方式
│   └── reports/            事故與 bug 報告
│
├── 5g-viz/                 即時視覺化系統（FastAPI + WebSocket + Grafana）
│   ├── README.md           給人的入口頁，先講系統是什麼、怎麼讀這套文件
│   ├── start-here/         onboarding / first-read 文件
│   ├── design/
│   │   ├── overview/       canonical 系統層文件
│   │   ├── backend/        canonical 後端文件
│   │   ├── frontend/       canonical 拓樸與 UI 文件
│   │   ├── grafana/        canonical Grafana / Prometheus 文件
│   │   ├── dvr/            canonical DVR 行為文件
│   │   ├── features/       跨層功能流程說明
│   │   └── reference/      schema 與設定參考
│   ├── plans/              歷史實作規劃與設計探索
│   │   ├── architecture/   跨層規劃文件
│   │   ├── backend/        後端重構規劃
│   │   ├── docs/           文件架構與重構規劃
│   │   ├── frontend/       前端規劃文件
│   │   ├── grafana/        Grafana 整合規劃
│   │   └── dvr/            DVR 規劃與除錯筆記
│   ├── notes/
│   │   ├── meetings/       會議紀錄（YYYY-MM-DD-meeting.md）
│   │   ├── impl/           實作筆記與決策
│   │   └── internals/      系統內部閱讀筆記
│   └── tmp/                暫存測試資料（不作長期保存）
│
├── archive/                已淘汰或被取代的文件（例如 `archive/5g-viz/`）
└── specs/                  3GPP 規格與 OpenAPI YAML 檔
    ├── TS 23.288/          Analytics、ML 模型配置與監控
    ├── TS 23.502/          UPF event exposure
    ├── TS 29.520/          NWDAF service API
    ├── TS 29.575/          ADRF data management
    └── yaml/               OpenAPI 合約（Nnwdaf_*、Nadrf_*、Nsmf_* ...）
```
