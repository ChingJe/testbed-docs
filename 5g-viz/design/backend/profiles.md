# Profiles

本文描述 `5g-viz` 目前的 profile 模型，以及 `config.yaml` 與 `topology.yaml` 如何共同影響 backend 行為。

## 1. Profile 的角色

`5g-viz` 用 `profiles/<profile>/` 封裝一組部署與視覺化設定。

目前一個 profile 的核心檔案是：

- `config.yaml`
- `topology.yaml`

其中：

- `config.yaml` 保存 server、Grafana、Prometheus、mapping、collector、replay policy 等 runtime 設定
- `topology.yaml` 保存 topology、event reactions、edge styles、layout 等視覺與 state 相關設定

profile 由 CLI 決定：

```bash
uv run run.py live --profile default
uv run run.py replay sessions/<id> --profile lab2
```

## 2. 載入流程

### `run.py`

`run.py` 先解析 `--profile`，再呼叫：

```python
load_profile_config(profile)
```

這會驗證：

- `profiles/<profile>/config.yaml`
- `profiles/<profile>/topology.yaml`

是否都存在且符合基本 schema。

### app runtime

進入 FastAPI lifespan 之後：

- live：讀取目前 profile 的 `topology.yaml`
- replay：讀取 session 目錄中的 `topology.yaml`

因此 replay 優先對齊錄製當下的 session topology，而不是現在 profile 最新內容。

## 3. `config.yaml` 的主要區塊

目前 `config.yaml` 至少包含下列區塊：

| 區塊 | 主要用途 |
|---|---|
| `server` | `ws_port` 等 server-level 設定 |
| `grafana` | iframe base、API base、admin credentials、groups、query/render defaults |
| `prometheus` | URL、query step、prediction offset、retention、`out_of_order_time_window` |
| `mappings` | UPF API IP 與 data subnet 映射 |
| `collector` | SSH source 與 log source 定義 |
| `replay` | `backfill_policy_default` |

### `collector.sources`

collector 來源現在已從舊的 `topology.yaml ssh_sources` 移到：

```yaml
collector:
  sources:
    - name: 5gc
      host: ...
      port: ...
      user: ...
      key_path: ...
      logs:
        - path: ...
          source: nwdaf
        - dir: ...
          filename: free5gc.log
          latest_subdir: true
          source: free5gc
```

這讓 collector runtime 設定和 topology 視覺設定分開管理。

### `mappings`

`mappings` 會直接影響 parser / metrics 語意，例如：

- `upf_ees_api_ips`
- `upf_data_subnets`

它們不是純部署細節，會影響 event payload 中的 node / UPF 映射結果。

## 4. `topology.yaml` 的角色

`topology.yaml` 目前主要負責：

- `nodes`
- `nf_aliases`
- `edge_styles`
- `layout`
- `panels`
- `event_reactions`

它同時影響：

- frontend topology 畫法
- `state_snapshot` 如何從 `event_reactions` 重建持久狀態
- live session 錄製時複製進 session 目錄的拓樸快照

換句話說，`topology.yaml` 不是純前端樣式檔，而是 frontend / state 共享契約。

## 5. live 與 replay 對 profile 的差異

### live

live 讀的是：

```text
profiles/<profile>/config.yaml
profiles/<profile>/topology.yaml
```

live 開始錄製後，會把 `topology.yaml` 複製到：

```text
sessions/<session_id>/topology.yaml
```

### replay

replay 仍使用目前 profile 的：

- `config.yaml`
- Prometheus / Grafana / collector / mapping policy

但 topology 與 event history 來自 session：

- `sessions/<id>/events.jsonl`
- `sessions/<id>/meta.json`
- `sessions/<id>/topology.yaml`

因此 replay 不會 retroactively 套用現在 profile 的 topology 改動。

## 6. `setup.sh` 與 `run.py` 的分工

### `setup.sh`

`setup.sh -p <profile>` 目前負責：

- 若 `config.yaml` 不存在，從 `config.example.yaml` 建立
- 若 `topology.yaml` 不存在，從 `profiles/default/topology.yaml` 複製
- 安裝 Python dependencies
- 檢查 Prometheus binary
- 準備 base `~/prometheus/prometheus.yml`
- 檢查 Grafana embedding / anonymous viewer 設定

### `run.py`

`run.py` 負責：

- 載入並驗證 profile
- 同步 managed `prometheus.yml`
- 檢查 Prometheus 是否已在跑
- 設定 runtime context
- 啟動 live / replay

也就是：

- `setup.sh` 偏環境準備
- `run.py` 偏正式 runtime control plane

## 7. 目前限制

- `topology.yaml` 仍沒有完整正式 schema 文件化；部分欄位是 runtime 約定
- `config.yaml` 的 validation 主要在 loader 層，不是獨立 schema 檔
- replay 仍會使用 session 內的 topology copy，因此 profile 更新不會自動修正舊 session 的重播結果
