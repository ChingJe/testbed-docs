# env-config

本文整理 `5g-viz` 目前與 profile `.env` 相關的環境變數契約，以及少數不屬於 profile 檔、但會一起影響啟動模式的 runtime env。

> 重要：目前若要覆寫 Prometheus 位址，應設定的是 `PROMETHEUS_URL`。`.env.example` 內的 `PROMETHEUS_BASE` 目前只是範本殘留，程式不會讀這個名稱。

## 1. 載入方式

`config.py` 在模組匯入時，會根據：

```text
PROFILE
```

選擇要載入哪一份：

```text
profiles/<PROFILE>/.env
```

若檔案不存在，程式會直接丟出錯誤，並提示先執行：

```bash
./setup.sh -p <profile>
```

`.env` 目前透過 `python-dotenv` 載入。這表示：

- profile `.env` 是主要設定來源
- 若程序在啟動前已經有同名 shell env，`load_dotenv()` 預設不會覆蓋它

## 2. profile `.env` 的主要變數

### SSH 與 log 路徑

這組變數不直接被 `config.py` 匯出，而是由 `topology.yaml` 的 `ssh_sources` 以名稱引用。

| 變數 | 格式 | 主要消費者 | 用途 |
|---|---|---|---|
| `SSH_5GC_HOST` | host / IP | `collector.py` | SSH 連線主機 |
| `SSH_5GC_PORT` | integer | `collector.py` | SSH port |
| `SSH_5GC_USER` | string | `collector.py` | SSH username |
| `SSH_5GC_KEY` | path | `collector.py` | SSH private key 路徑 |
| `LOG_FREE5GC_DIR` | path | `collector.py` | `latest_subdir` 模式下的 log 根目錄 |
| `LOG_NWDAF` | path | `collector.py` | 直接 tail 的 NWDAF log 路徑 |

這些值透過 `topology.yaml` 內的下列欄位被間接使用：

- `host_env`
- `port_env`
- `user_env`
- `key_env`
- `dir_env`
- `path_env`

也就是說，`.env` 和 `topology.yaml` 之間是靠字串名稱關聯，而不是靠正式 schema 綁定。

### Grafana

這組變數由 `config.py` 直接解析，並供 backend 與 frontend 間接使用。

| 變數 | 格式 | 主要消費者 | 用途 |
|---|---|---|---|
| `GRAFANA_BASE` | URL | `config.py`、`main.py`、前端 iframe | 瀏覽器可達的 Grafana base URL |
| `GRAFANA_ADMIN_USER` | string | `grafana_setup.py` | 建 datasource / dashboard |
| `GRAFANA_ADMIN_PASS` | string | `grafana_setup.py` | 同上 |
| `GRAFANA_GROUPS` | `a,b,c` | `config.py`、`grafana_setup.py`、`main.py` | dashboard panel 群組與 session metadata |
| `GRAFANA_DEVIATION_LABEL` | string | `grafana_setup.py` | deviation panel 標題（選填，預設 `"Deviation (sMAPE)"`） |

這裡有兩個重要限制：

- `GRAFANA_BASE` 必須是瀏覽器看得到的位址，不應盲目設成 `http://localhost:3000`
- `GRAFANA_GROUPS` 會被解析成逗號分隔字串陣列，空白會被 `strip()`

live session 建立時，`main.py` 也會把 `GRAFANA_GROUPS` 寫進 `meta.json`。replay 時若 `meta.json` 內有保存值，會優先使用 session 內保存的 groups。

### Grafana 遺留變數

`profiles/default/.env` 目前還留著：

| 變數 | 現況 |
|---|---|
| `GRAFANA_PUBLIC_TOKEN` | 目前程式碼不讀取，也不參與 iframe 嵌入；屬於舊設計殘留 |

### Parser / metrics 用的 UPF 映射

| 變數 | 格式 | 主要消費者 | 用途 |
|---|---|---|---|
| `UPF_EES_API_IPS` | `ip=name,ip=name` | `rules/smf.py` | 把 `selected_upf_api_root` 中的 API IP 映射成 node 名稱 |
| `UPF_DATA_SUBNETS` | `octet=name,octet=name` | `rules/nwdaf.py` | 把 `10.x.x.x` 第二個 octet 映射成 UPF 名稱 |

這兩個變數都經過 `_parse_kv()` 解析成字典，所以格式必須是：

```text
key=value,key=value
```

它們不只是部署細節，也會改變 event payload：

- `sbi_call.to`
- `upf_volume.upf`

目前 fallback 行為如下：

- `UPF_EES_API_IPS` 若找不到對應 IP，`rules/smf.py` 會回退成 `"UPF-EES"`
- `UPF_DATA_SUBNETS` 若找不到對應 subnet，`rules/nwdaf.py` 會保留原始 IP 字串

### 其他 profile 變數

| 變數 | 格式 | 現況 |
|---|---|---|
| `WS_PORT` | integer | `config.py` 會讀取，但 `start.sh` 目前仍硬編碼用 `uvicorn --port 8765` 啟動，因此改它不會改變實際 listen port |

## 3. Prometheus 相關變數

目前程式實際使用的是：

| 變數 | 格式 | 主要消費者 | 用途 |
|---|---|---|---|
| `PROMETHEUS_URL` | URL | `main.py` | replay backfill / query / admin API 使用的 Prometheus base URL |

但目前 `.env.example` 寫的是：

| 變數 | 現況 |
|---|---|
| `PROMETHEUS_BASE` | 範本檔存在，但目前程式不讀這個名稱 |

因此目前若要覆寫預設的 `http://localhost:9090`，應設定的是 `PROMETHEUS_URL`，不是 `PROMETHEUS_BASE`。

## 4. 不屬於 profile `.env` 的 runtime env

這幾個變數通常由 `start.sh` 匯出，不一定放在 `profiles/<profile>/.env` 裡：

| 變數 | 來源 | 用途 |
|---|---|---|
| `PROFILE` | `start.sh -p` 或 shell env | 決定要載入哪個 profile 目錄 |
| `SESSION_MODE` | `start.sh` | `live` 或 `replay` |
| `SESSION_PATH` | `start.sh --replay` | replay 模式下要讀的 session 路徑 |
| `FORCE_BACKFILL` | `start.sh --force-backfill` | 強制 replay metric backfill |

這組變數比較像執行期模式控制，而不是 profile 的持久設定。

## 5. `setup.sh` 建立哪些預設值

`setup.sh -p <profile>` 目前會：

1. 建立 `profiles/<profile>/`
2. 若 `.env` 不存在，從 `.env.example` 複製
3. 若 `topology.yaml` 不存在，從 `profiles/default/topology.yaml` 複製

`.env.example` 目前提供的預設鍵主要包含：

- SSH / log 路徑變數
- `WS_PORT`
- `UPF_EES_API_IPS`
- `UPF_DATA_SUBNETS`
- `GRAFANA_BASE`
- `GRAFANA_ADMIN_USER`
- `GRAFANA_ADMIN_PASS`
- `GRAFANA_GROUPS`
- `GRAFANA_DEVIATION_LABEL`（選填，已以 `#` 註解，預設 `"Deviation (sMAPE)"`）
- `PROMETHEUS_BASE`

其中 `setup.sh` 在第一次建立 profile 後，還會特別提示使用者檢查：

- `SSH_5GC_KEY`
- `GRAFANA_BASE`
- `GRAFANA_ADMIN_USER`
- `GRAFANA_ADMIN_PASS`
- `GRAFANA_GROUPS`

## 6. 目前限制與已知差異

- `.env` 與 `topology.yaml` 之間只有字串名稱關聯，例如 `host_env: SSH_5GC_HOST`；拼字錯誤通常要到 runtime 才會發現
- `WS_PORT` 雖然存在於 `config.py`，但 `start.sh` 目前沒有用它來決定 uvicorn port
- `.env.example` 目前列出 `PROMETHEUS_BASE`，但 `main.py` 實際讀的是 `PROMETHEUS_URL`
- replay mode 不啟動 collector，因此 `SSH_5GC_*`、`LOG_*` 這類 collector 相關變數在 replay runtime 中通常不會被實際使用
- `GRAFANA_PUBLIC_TOKEN` 目前仍可能出現在既有 `.env`，但程式碼不會讀它
- replay 時主要依賴 session 內保存的 `meta.json` 與 `topology.yaml`；修改目前 profile `.env` 不會 retroactively 改變舊 session 的 topology 或 grafana groups metadata
