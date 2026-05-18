# Setup

本文描述 `5g-viz` 目前如何準備 Grafana 與 Prometheus，使前端能嵌入可查詢的 dashboard。

## 1. 目前的責任分工

Grafana / Prometheus 相關啟動責任目前分成三部分：

- `setup.sh`：環境準備與基本驗證
- `run.py`：runtime 啟動、Prometheus config sync、Prometheus availability check
- `services/grafana_setup.py`：建立或更新 datasource 與 dashboard

目前已沒有：

- `start.sh`
- replay 專用暫時 TSDB
- 啟動時全域清空所有 `nwdaf_*` series 的模型

## 2. Prometheus 前提

### binary 與 data dir

Prometheus binary 可以來自：

- 系統安裝的 `prometheus`
- `~/prometheus/prometheus`

資料目錄固定為：

```text
~/prometheus/data
```

managed config 固定為：

```text
~/prometheus/prometheus.yml
```

### 正式建議模式

目前正式建議是把 Prometheus 作為長駐 user service：

```bash
uv run run.py prom install-user-service --profile default
systemctl --user daemon-reload
systemctl --user enable --now 5g-viz-prometheus.service
```

`run.py live/replay` 只會：

- 同步 managed `prometheus.yml`
- 檢查 Prometheus ready
- 觸發 config reload

若 Prometheus 沒開，`run.py` 會停止並提示使用者先啟動 service。

## 3. `setup.sh` 做什麼

`setup.sh` 現在不是正式啟動入口，而是 setup / validation helper。

主要工作：

### 建立 profile 基本檔案

- 若 `profiles/<profile>/config.yaml` 不存在，從 `config.example.yaml` 建立
- 若 `profiles/<profile>/topology.yaml` 不存在，從 `profiles/default/topology.yaml` 複製

### 安裝 Python 依賴

- 執行 `uv sync`

### 檢查 Prometheus

- 確認系統 binary 或 `~/prometheus/prometheus` 是否存在
- 若 `~/prometheus/prometheus.yml` 不存在，從 repo 內 `prometheus.yml` 建立 base file

### 檢查 Grafana embedding / anonymous viewer

若有 sudo 權限，`setup.sh` 會嘗試確保：

```ini
[security]
allow_embedding = true

[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

若沒有 sudo，則印出手動操作步驟。

## 4. `run.py` 對 Prometheus 的責任

在 live / replay 啟動前，`run.py` 會：

1. 載入 profile `prometheus` 設定
2. 生成 managed `~/prometheus/prometheus.yml`
3. 確認 TSDB path 與 log path 存在
4. 檢查 `cfg.prometheus.url` 是否 ready
5. 呼叫 `/-/reload`

目前 `run.py` 不再：

- 自己 spawn 一個 fallback Prometheus process
- 在 replay 時切成獨立暫時 TSDB
- 啟動時刪除所有既有 `nwdaf_*` series

## 5. `grafana_setup.py` 做什麼

app 啟動時，backend 會呼叫：

```python
grafana_setup.setup(groups=...)
```

流程分兩步：

### Step 1. 解析或建立 Prometheus datasource

- 列出 Grafana datasources
- 尋找第一個 `type == prometheus`
- 若找到但設定不符，更新為期望值
- 若找不到，建立新的 Prometheus datasource

目前 datasource 期望值包括：

- `url = http://localhost:9090`
- `access = proxy`
- `isDefault = true`
- `jsonData.timeInterval = PROMETHEUS_QUERY_TIME_INTERVAL`

### Step 2. 建立或覆蓋 dashboard

dashboard 由 Python 動態生成，固定：

- `uid = "nwdaf-traffic"`
- `overwrite = true`

內容依 `grafana.groups` 建立：

- 每個 group 一張 traffic panel
- degradation / chronic / deviation / annotation 相關 panel

因此手動在 Grafana UI 修改這個 dashboard，下次 app 啟動仍可能被覆蓋。

## 6. Grafana profile 設定如何影響 dashboard

`config.yaml.grafana` 目前直接影響：

- `base`
- `api_base`
- `admin_user`
- `admin_pass`
- `groups`
- `refresh`
- `default_from`
- panel query / threshold 相關常數

其中最直接的使用路徑是：

- frontend iframe：`base`
- Grafana admin API：`api_base` + admin credentials
- dashboard panel layout / query target：`groups`

## 7. 目前限制

- datasource URL 目前仍預設為 `http://localhost:9090`
- dashboard 由 Python 生成，Grafana UI 的手動客製化不易保留
- `setup.sh` 只處理 embedding / anonymous viewer，不會自動幫你修好 Grafana `[server] root_url` 與 `/grafana` subpath
