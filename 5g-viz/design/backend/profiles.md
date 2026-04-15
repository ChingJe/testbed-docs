# Profiles

本文描述 `5g-viz` 的 profile 結構，以及 `.env` 與 `topology.yaml` 如何共同影響 backend 行為。

## 1. Profile 的角色

`5g-viz` 用 `profiles/<profile>/` 來封裝一組部署與視覺化設定。backend 啟動時會依 `PROFILE` 選擇要載入哪一組 profile。

目前一個 profile 至少包含：

- `.env`
- `topology.yaml`

其中：

- `.env` 保存環境相依、可能含有憑證或 URL 的值
- `topology.yaml` 保存可版本化的拓樸、反應規則與 collector 來源結構

## 2. 載入流程

### `config.py`

模組匯入時，`config.py` 會先根據 `PROFILE` 決定：

```text
profiles/<PROFILE>/.env
```

若檔案不存在，會直接丟出錯誤並要求先跑 `./setup.sh -p <profile>`。

之後 `load_dotenv(...)` 會把該 profile 的 `.env` 載入目前程序。

### `main.py`

進入 lifespan 之後：

- live mode：`_load_live_topo_config()` 讀取 `profiles/<PROFILE>/topology.yaml`
- replay mode：`_load_replay_topo_config()` 讀取 session 目錄裡的 `topology.yaml`

這表示 replay 並不依賴目前 profile 下的最新 topology，而是依賴錄製時保存的那一份 session copy。

## 3. `.env` 目前提供哪些 backend 設定

### SSH 與 log 路徑

這組變數不由 backend 直接硬編碼，而是提供給 `topology.yaml` 的 `ssh_sources` 透過名稱引用：

- `SSH_5GC_HOST`
- `SSH_5GC_PORT`
- `SSH_5GC_USER`
- `SSH_5GC_KEY`
- `LOG_FREE5GC_DIR`
- `LOG_NWDAF`

collector 透過 `host_env` / `path_env` / `dir_env` 之類欄位在執行時查這些值。

### Grafana

`config.py` 目前直接匯出的 Grafana 相關變數有：

- `GRAFANA_BASE`
- `GRAFANA_ADMIN_USER`
- `GRAFANA_ADMIN_PASS`
- `GRAFANA_GROUPS`

其中：

- `GRAFANA_BASE` 會回傳給前端組 iframe URL
- `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASS` 由 `grafana_setup.py` 使用
- `GRAFANA_GROUPS` 會寫進 live session `meta.json`，也會參與 dashboard 建立

### UPF 映射

`.env` 另有兩組會直接影響 parser / metrics 語意的映射：

- `UPF_EES_API_IPS`
- `UPF_DATA_SUBNETS`

用途如下：

- `rules/smf.py`：把 `selected_upf_api_root` 中的 IP 映射成 `UPF-EES` / `UPF-EES2`
- `rules/nwdaf.py`：把 `10.x.x.x` 第二個 octet 映射成 UPF node 名稱

也就是說，這些值不只是部署細節，還會改變 event payload 中的 `to` 或 `upf` 欄位。

## 4. `topology.yaml` 的主要區塊

目前 `profiles/default/topology.yaml` 至少包含下列區塊：

| 區塊 | 主要用途 | 主要消費者 |
|---|---|---|
| `nodes` | 定義 Cytoscape node、label、position、compound 關係 | frontend、state |
| `nf_aliases` | parser / rule 名稱到 node ID 的映射 | frontend、state |
| `edge_styles` | 邊顏色、寬度、顯示名稱、動畫時間 | frontend |
| `layout` | 初始 zoom 與 pan offset | frontend |
| `panels` | topology / charts / event log 的版面比例 | frontend |
| `ssh_sources` | collector 的 SSH 與 log 來源結構 | backend collector |
| `event_reactions` | event type 到視覺 / 狀態 action 的對應 | frontend、state |

這也是為什麼 `topology.yaml` 不能簡單視為純前端配置檔。

## 5. `ssh_sources` 是 backend 契約

`ssh_sources` 目前長這樣：

```yaml
ssh_sources:
  - name: 5gc
    host_env: SSH_5GC_HOST
    port_env: SSH_5GC_PORT
    user_env: SSH_5GC_USER
    key_env: SSH_5GC_KEY
    logs:
      - dir_env: LOG_FREE5GC_DIR
        filename: free5gc.log
        latest_subdir: true
        source: free5gc
      - path_env: LOG_NWDAF
        source: nwdaf
```

這裡真正定義的是：

- collector 要連哪一台主機
- 需要 tail 哪些 log
- 每條 log 在 parser 端會帶什麼 `source` tag

因此 `ssh_sources` 同時影響：

- 能否成功建立 SSH 收集
- parser rule 能否用 `source` 做正確比對

## 6. `event_reactions` 是 cross-layer 契約

`event_reactions` 看起來像前端動畫設定，但其實同時被兩個地方使用：

- `frontend/topology.js`：解讀 `flash_edge`、`pulse`、`add_class`、`remove_class`
- `state.py`：只重用 `add_class` / `remove_class` 來維護 `state_snapshot`

這代表一筆 reaction 的影響範圍可能跨越：

- UI 動畫
- node 持久狀態
- 新 client 連線時看到的 snapshot

所以修改 `event_reactions` 時，不只是改動畫效果，也可能改變 backend 的 snapshot 語意。

## 7. live 與 replay 對 profile 的差異

### live

live mode 讀的是目前 `profiles/<PROFILE>/topology.yaml`，並在建立新 session 時把它複製到：

```text
sessions/<session_id>/topology.yaml
```

這份 copy 成為該次錄製的 canonical 設定快照。

### replay

replay mode 讀的是 session 目錄中的 `topology.yaml`，並額外把：

```python
cfg["_profile"] = meta.get("profile", "replay")
```

補到 config 中。

因此 replay 畫面與 state 重建會優先對齊錄製當下的拓樸，而不是現在 profile 最新內容。

## 8. `setup.sh` 與 `start.sh` 的關係

### `setup.sh`

`setup.sh -p <profile>` 會：

1. 建立 `profiles/<profile>/`
2. 若 `.env` 不存在，從 `.env.example` 複製一份
3. 若 `topology.yaml` 不存在，從 `profiles/default/topology.yaml` 複製一份

因此新 profile 的初始內容目前是以 default profile 為模板。

### `start.sh`

`start.sh` 會讀：

- `PROFILE`
- `SESSION_MODE`
- `SESSION_PATH`
- `FORCE_BACKFILL`

其中只有 `PROFILE` 直接對應到 `profiles/<profile>/`。另外三個屬於執行模式控制，不一定要放在 profile `.env` 裡。

## 9. 目前限制

- profile schema 目前沒有正式驗證器；`topology.yaml` 若欄位缺漏，多半會在 collector、frontend 或 state 執行時才出錯
- `.env` 與 `topology.yaml` 之間靠字串名稱關聯，例如 `host_env: SSH_5GC_HOST`；拼字錯誤無法在載入階段一次性驗證
- `nodes` ID、`nf_aliases`、`event_reactions` 彼此高度耦合；改 node ID 時通常必須同步改三處
- replay 模式會優先使用 session 內保存的 topology，因此修改目前 profile 並不會 retroactively 改變舊 session 的重播結果
