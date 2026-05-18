# env-config

> Note: the filename is historical. The current runtime no longer uses profile `.env` files. Read this page as a bridge from the old env model to the current `config.yaml` model.

本文的重點不是再描述 `.env` 契約，而是說明：

1. 舊 `.env` 心智現在對應到哪裡
2. 目前真正的設定來源是什麼
3. 哪些設定屬於 `config.yaml`、哪些屬於 `topology.yaml`

若要看現況 canonical profile reference，請優先讀：

- [`../backend/profiles.md`](../backend/profiles.md)

## 1. 現況結論

目前 `5g-viz` 已正式移除：

- `profiles/<profile>/.env`
- root `.env.example`
- 以 env 作為 runtime config source 的模式

現在的設定來源是：

```text
profiles/<profile>/config.yaml
profiles/<profile>/topology.yaml
```

而執行模式由 CLI 顯式指定：

```bash
uv run run.py live --profile <profile>
uv run run.py replay sessions/<id> --profile <profile>
```

## 2. 舊 `.env` 變數現在對應到哪裡

### SSH / collector 設定

舊模型裡：

- `SSH_5GC_HOST`
- `SSH_5GC_PORT`
- `SSH_5GC_USER`
- `SSH_5GC_KEY`
- `LOG_*`

現在都收斂到：

- `config.yaml -> collector.sources[*]`

### Grafana 設定

舊模型裡：

- `GRAFANA_BASE`
- `GRAFANA_API_BASE`
- `GRAFANA_ADMIN_USER`
- `GRAFANA_ADMIN_PASS`
- `GRAFANA_GROUPS`

現在對應到：

- `config.yaml -> grafana.*`

### Prometheus 設定

舊模型裡主要靠：

- `PROMETHEUS_URL`

現在對應到：

- `config.yaml -> prometheus.*`

除了 URL 外，還包括：

- `retention_time`
- `retention_size`
- `out_of_order_time_window`

### UPF 映射

舊模型裡：

- `UPF_EES_API_IPS`
- `UPF_DATA_SUBNETS`

現在對應到：

- `config.yaml -> mappings.upf_ees_api_ips`
- `config.yaml -> mappings.upf_data_subnets`

這兩組 mapping 仍會影響 parser 產出的 event payload，例如：

- `sbi_call.to`
- `upf_volume.upf`

## 3. `config.yaml` 與 `topology.yaml` 的邊界

目前建議心智如下：

### `config.yaml`

承載：

- server port
- Grafana base / API base / groups / admin credentials
- Prometheus URL / retention / window
- replay policy default
- collector sources
- UPF / subnet mappings

### `topology.yaml`

承載：

- nodes
- aliases
- edge styles
- event reactions
- visual defaults

也就是：

- `config.yaml` 偏 runtime / environment / integration
- `topology.yaml` 偏 UI / topology semantics / reactions

## 4. 舊 runtime env 現在對應到哪裡

舊模型裡常見：

- `PROFILE`
- `SESSION_MODE`
- `SESSION_PATH`
- `FORCE_BACKFILL`

這些現在都不再作為主要 env surface，而是改成 CLI 參數：

| 舊概念 | 現在做法 |
|---|---|
| `PROFILE` | `--profile <profile>` |
| `SESSION_MODE=live` | `run.py live` |
| `SESSION_MODE=replay` | `run.py replay` |
| `SESSION_PATH` | `run.py replay <session_dir>` |
| `FORCE_BACKFILL` | `--backfill=overwrite` |

## 5. 仍值得記住的遷移點

雖然 `.env` 已移除，但舊文件裡常見的欄位語意仍有延續性：

- `GRAFANA_GROUPS` 仍對應到 panel groups
- `UPF_EES_API_IPS` / `UPF_DATA_SUBNETS` 仍決定 subscription chain 與 UPF notify 的終點
- `PROMETHEUS_URL` 的概念仍存在，只是現在寫在 `config.yaml`

因此看到舊文件提到這些 env name 時，應把它理解成「今天的 `config.yaml` 欄位」。

## 6. 現在不該再做的事

以下都已不是現況：

- 建立或複製 profile `.env`
- 透過 shell env 覆蓋一般 profile 設定
- 以 `start.sh -p ...` 選 profile
- 把 collector source 定義寫進 `topology.yaml` 的 `ssh_sources`

## 7. 目前文件中的定位

因為檔名 `env-config.md` 已是歷史名稱，這份文件現在應視為：

- 一份 migration-oriented reference
- 幫助讀者把舊 `.env` 心智映射到新 `config.yaml`

若需要實際現況 schema 與欄位來源，請回到：

- [`../backend/profiles.md`](../backend/profiles.md)
