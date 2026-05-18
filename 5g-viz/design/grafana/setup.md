# Setup

> Historical note: this document still describes the old `setup.sh + start.sh` startup split and disposable replay TSDB assumptions. The current runtime uses `config.yaml`, `run.py`, a managed Prometheus config, and a long-running Prometheus user service model.

本文描述 `5g-viz` 目前如何準備 Grafana 與 Prometheus，使前端能嵌入可查詢的 dashboard。

## 1. 這層的責任

Grafana setup 層目前由三部分共同構成：

- `setup.sh`：檢查本機環境是否具備 Prometheus 與 Grafana 的必要設定
- `start.sh`：啟動本地 Prometheus，並在 replay 模式下準備乾淨的 TSDB
- `grafana_setup.py`：在 app 啟動時建立或更新 Prometheus datasource 與 dashboard

其中前端 iframe 的 `src` 切換不在這一層，另見 [../frontend/grafana-embed.md](../frontend/grafana-embed.md)。

## 2. 環境前提

目前 Grafana 整合建立在幾個明確假設上：

- 瀏覽器能透過 `GRAFANA_BASE` 連到 Grafana
- `grafana_setup.py` 能用 `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASS` 呼叫 Grafana HTTP API
- Grafana 內的 Prometheus datasource 會連到 `http://localhost:9090`
- `start.sh` 啟動的 Prometheus 會開啟 remote write receiver

這代表目前預設部署假設是：

- Grafana 與 Prometheus 在同一台主機上
- `5g-viz` 也能直接存取這台本機 Prometheus 的 `9090`

若 Grafana 不是和 Prometheus 同機，`grafana_setup.py` 內寫死的 datasource URL 需要先調整。

## 3. `setup.sh` 做了什麼

`setup.sh` 的 Grafana / Prometheus 相關工作主要有四項：

這裡說的 setup 範圍是執行 `5g-viz` 的本機環境，例如：

- 本機是否找得到 `prometheus` binary
- 本機使用者家目錄下是否已有 `~/prometheus/prometheus.yml`
- 本機 `/etc/grafana/grafana.ini` 是否已開 `allow_embedding`

它不會去驗證遠端 5GC VM 的 SSH 可達性，也不會檢查 VM 上的 log 路徑是否存在；那些屬於 collector / profile 層的責任。

### 建立 profile 基本檔案

- 若 `profiles/<profile>/.env` 不存在，從 `.env.example` 複製
- 若 `profiles/<profile>/topology.yaml` 不存在，從 `profiles/default/topology.yaml` 複製

### 確認 Prometheus 可執行檔

允許兩種來源：

- 系統安裝的 `prometheus`
- `~/prometheus/prometheus`

若使用者家目錄下沒有 `prometheus.yml`，會從 repo 內的 `prometheus.yml` 複製一份過去。

### 檢查 Grafana `allow_embedding`

`setup.sh` 會確認 `/etc/grafana/grafana.ini` 中是否已有：

```ini
[security]
allow_embedding = true
```

若尚未設定且有 sudo 權限，會直接附加並重啟 Grafana；若沒有 sudo，則只印出手動操作提示。

### 檢查匿名唯讀存取

目前前端是以 kiosk iframe 嵌入一般 dashboard，而不是 public dashboard token，所以還需要：

```ini
[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

`setup.sh` 同樣會嘗試自動補上，否則提示使用者手動設定。

## 4. `start.sh` 對 Grafana 的影響

`start.sh` 本身不直接碰 Grafana API，但會決定 Grafana 背後看到的是哪一份 Prometheus 資料。

### Prometheus 啟動方式

每次啟動都會帶：

```text
--web.enable-admin-api
--web.enable-remote-write-receiver
```

其中 remote write receiver 是 replay backfill 與 pseudo-live 必要條件。

### 每次啟動的 nwdaf_* series 清理

無論 live 或 replay，`start.sh` 每次啟動 Prometheus 並等待 ready 之後，都會透過 admin API
刪除所有 `nwdaf_*` series：

```bash
curl -X POST --data-urlencode 'match[]={__name__=~"nwdaf_.+"}' \
    http://localhost:9090/api/v1/admin/tsdb/delete_series
curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

這樣做的目的，是確保前一次 session 的殘留 metrics 不會汙染新 session 的圖表，同時只精確清除
5g-viz 自身的 metrics，不影響 Prometheus 中其他 job 的資料。

## 5. `grafana_setup.py` 的初始化流程

FastAPI app 啟動時，`main.py` 會背景執行：

```python
_setup_grafana(_current_grafana_groups)
```

最終會呼叫：

```python
grafana_setup.setup(groups=groups)
```

這個 setup 流程分成兩步。

### Step 1. 解析或建立 Prometheus datasource

`_get_or_create_datasource_uid()` 會先列出 `/api/datasources`，尋找第一個 `type == "prometheus"` 的 datasource。

若找到，會檢查幾個欄位是否符合目前期待：

- `url = http://localhost:9090`
- `access = proxy`
- `isDefault = true`
- `jsonData.timeInterval = 5s`

不符合就會用 `PUT /api/datasources/uid/<uid>` 更新。

若找不到任何 Prometheus datasource，才會用 `POST /api/datasources` 新建一個。

### Step 2. 建立或覆蓋 dashboard

datasource UID 確定後，`setup()` 會組出一份固定 UID 的 dashboard：

- `uid = "nwdaf-traffic"`
- `title = "NWDAF Traffic Prediction"`

然後透過：

```python
POST /api/dashboards/db
```

以 `overwrite = True` 送進 Grafana。

這表示目前 dashboard 是「啟動即覆蓋」模式；若有人在 Grafana UI 手動改 panel，下一次啟動 `5g-viz` 可能會被程式重新寫回。

## 6. `GRAFANA_GROUPS` 如何影響 dashboard

`GRAFANA_GROUPS` 來自 profile `.env`，由 `config.py` 解析成字串陣列。

這份設定會同時影響：

- live session `meta.json` 內的 `grafana_groups`
- replay session 載入後的 `_current_grafana_groups`
- `grafana_setup.setup(groups=...)` 實際建立多少 traffic panels

目前 layout 規則是：

- 每個 group 一個 traffic panel
- 右側額外保留一個 deviation panel
- 總寬固定 24 grid columns，平均分配給所有 group panel 與 deviation panel

因此 `GRAFANA_GROUPS` 的數量會直接改變 dashboard 橫向切法。

## 7. Prometheus scrape 設定

repo 內的 `prometheus.yml` 很簡單，只定義兩個 scrape job：

- `prometheus`：抓 `localhost:9090`
- `5g-viz`：每 `5s` 抓 `localhost:8765`

這裡的 `5g-viz` scrape job 是 live 模式圖表資料的來源；replay 模式的歷史圖則另外靠 remote write 注入。

## 8. 目前限制

- `grafana_setup.py` 目前只會挑第一個 Prometheus datasource，不支援明確指定既有 datasource UID
- datasource URL 寫死為 `http://localhost:9090`，對分離式部署不夠彈性
- dashboard 採 `overwrite=True`，手動在 Grafana UI 做的修改不會被保留
- `setup.sh` 對 `grafana.ini` 的修改依賴 sudo；沒有權限時只能提示，不能保證實際環境已就緒
