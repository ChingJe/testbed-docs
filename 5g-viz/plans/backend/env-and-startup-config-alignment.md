# Env And Startup Config Alignment

## Background

`5g-viz` 目前有三個相關但未完全對齊的設定面：

- root `/.env.example` 作為 profile `.env` 模板
- `profiles/default/.env` 作為實際預設 profile 設定
- `start.sh` 作為實際啟動入口

目前存在幾個落差：

- `.env.example` 仍保留已不再讀取的 `PROMETHEUS_BASE`
- `.env.example` 遺漏目前程式已支援的 Grafana metric / threshold 設定
- `WS_PORT` 的註解把它描述成 WebSocket port，但實際上是整個 HTTP + WebSocket server port
- `start.sh` 啟動 `uvicorn` 時仍寫死 `8765`，沒有使用 profile `.env` 中的 `WS_PORT`
- `profiles/default/.env` 的 `GRAFANA_BASE` 註解仍停留在較早期的 host-IP 說法

## Goal

將 env 模板、default profile 與啟動腳本對齊到目前實作，讓：

- 範例檔只保留仍被程式讀取的 key
- default profile 與範例檔的說明語意一致
- `WS_PORT` 真的會影響 `start.sh` 啟動的 server port
- Prometheus URL 的覆寫方式在模板與實作中一致

## Scope

本次只調整：

- `5g-viz/.env.example`
- `5g-viz/profiles/default/.env`
- `5g-viz/start.sh`

不包含：

- 變更 Grafana / Prometheus 功能行為
- 新增新的 profile 結構
- 修改 `README.md` 的大段部署說明

## Planned Changes

1. 將 `.env.example` 的 `PROMETHEUS_BASE` 改為 `PROMETHEUS_URL`
2. 補上目前程式已支援的 metric / threshold env keys
3. 移除不再讀取的 deviation panel 舊設定註解
4. 統一 `WS_PORT` 的說明文字為 server port
5. 在 `start.sh` 載入 profile `.env`，並以 `WS_PORT` 啟動 `uvicorn`
6. 修正 `profiles/default/.env` 的 `GRAFANA_BASE` 註解，使其反映 browser-facing URL/path 語意

## Acceptance

- `start.sh` 不再硬編碼 `8765`
- `.env.example` 與 `profiles/default/.env` 都不再出現 `PROMETHEUS_BASE`
- `.env.example` 與 `config.py` / `main.py` 的實際讀取 key 一致
- `profiles/default/.env` 仍可直接作為 default profile 啟動
