# Replay Runtime Simplification and Persistent Prometheus Plan

本文件規劃 `5g-viz` 下一輪較大面積的架構重構，核心目標有三條：

1. 正式移除 replay pseudo-live 路徑，收斂成單一 replay / Grafana 模型
2. 把 Prometheus 從「每次啟動都清空的暫時 cache」改成「可長期保存的 session store」
3. 為 topology transient effects 加入獨立於 playback 的 runtime controls
4. 將目前偏 `.env` 導向的 profile 設定，收斂成較一般的 `config.yaml` 模型

這三件事彼此有關：一旦 replay 不再依賴 pseudo-live，Prometheus 的角色就從「播放時的臨時模擬器」
更明確地回到「持久保存 replay metrics 的查詢層」；而前端也可把原本混在 DVR speed 裡的 transient
effect duration 抽出成獨立控制面板。另一方面，當 Python control plane 取代 `start.sh` 成為主要入口後，
設定模型也不應再只停留在 shell-friendly `.env`。

> 狀態：`config.yaml + run.py + shared loader` groundwork 與 Prometheus lifecycle groundwork 已落地；後續 phases 尚未開始。

---

## 1. 背景與問題陳述

目前 `5g-viz` 的 replay 路徑是歷史演進的混合結果：

- replay `paused / scrubbed` 看的是原始 session backfill
- replay `playing` 原本看的是 pseudo-live session
- 啟動腳本會在每次啟動 Prometheus 後刪除所有 `nwdaf_*`
- replay mode 還會使用暫時 TSDB 目錄，process 結束後直接刪掉
- topology transient effect 的顯示時間，一部分寫死在 `topology.yaml`，一部分又受 playback speed 影響

這些設計各自都有當時的合理性，但放在一起後，帶來了三個長期問題：

### 1.1 replay 模型過於分裂

目前 backend 還保留：

- `MetricPlayer`（[5g-viz/metric_player.py](/home/chingje/testbed/5g-viz/metric_player.py)）
- `/api/replay/play|pause|speed`（[5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:1116)）
- pseudo session cleanup（[5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:616)）

但 frontend 的實驗 branch 已經證明，單靠 historical relative Grafana window 就能在 `1x` 播放下提供接近
live 的平滑體驗，不再需要 pseudo-live remap。

### 1.2 Prometheus 目前更像 disposable cache，而不是 long-lived session store

啟動腳本現況：

- live mode 使用 `~/prometheus/data`
- replay mode 使用 `/tmp/prometheus-replay-*`
- 不論 live / replay，都在啟動後先：
  - `delete_series match[]={__name__=~"nwdaf_.+"}`
  - `clean_tombstones`

對應實作現況見 [5g-viz/run.py](/home/chingje/testbed/5g-viz/run.py)。

這代表：

- session 雖然有 `session` label 隔離
- 但啟動流程實際上沒有把 Prometheus 當成可持久保留的 session repository
- replay 是否可直接播放，也無法只靠既有 TSDB 狀態穩定判斷

### 1.3 transient visual effects 缺乏獨立的 runtime control

目前前端 topology 效果主要由兩層控制：

- config-driven durations：`topology.yaml` 中的 `edge_styles.*.duration` 與 `pulse.duration`
- playback-speed scaling：`topology.js` 的 `_effectiveDuration()` 會把 base duration 除以 `_playbackSpeed`

對應實作見：

- [5g-viz/profiles/default/topology.yaml](/home/chingje/testbed/5g-viz/profiles/default/topology.yaml)
- [5g-viz/frontend/topology.js](/home/chingje/testbed/5g-viz/frontend/topology.js:8)

這導致：

- 想調整「邊閃多久、pulse 持續多久」時，必須改 profile config 或改 playback speed
- playback speed 同時影響 replay timeline 與 transient duration，語意耦合
- 使用者無法在 runtime 直接試 `x2`、`x0.5`、reset 等互動式調整

---

## 2. 目前專案現況總結

以下是與本次重構最直接相關的現況盤點。

### 2.1 Replay 路徑現況

backend：

- replay session 載入：`_init_replay_session()`  
  [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:879)
- replay backfill：`_run_replay_backfill()`  
  [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:663)
- pseudo-live controller：`MetricPlayer`  
  [5g-viz/metric_player.py](/home/chingje/testbed/5g-viz/metric_player.py:201)
- replay playback API：
  [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:1116)

frontend：

- DVR / replay state machine：`frontend/events.js`
- 實驗 branch 現況已存在 `RELATIVE_PLAY`
  [5g-viz/frontend/events.js](/home/chingje/testbed/5g-viz/frontend/events.js:24)
- 但 backend pseudo-live 路徑仍完整保留，尚未真正移除

結論：

- **架構上**仍是 pseudo-live system
- **行為上**已經有相對時間窗播放的實驗成功案例
- 下一步應把成功的行為正式收斂成 canonical runtime，而不是讓 frontend / backend 長期分叉

### 2.2 Session artifact 現況

session 仍是 replay 的 canonical artifact：

- `meta.json`
- `events.jsonl`
- `topology.yaml`

現有 `meta.json` 已有：

- `session_id`
- `profile`
- `grafana_groups`
- `start_time`
- `end_time`
- `event_count`

相關邏輯見：

- [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:816)
- [5g-viz/session_import.py](/home/chingje/testbed/5g-viz/session_import.py:120)
- [5g-viz/session_io.py](/home/chingje/testbed/5g-viz/session_io.py)

此外，`/api/sessions` 已經會回傳 `event_count`、`start_time`、`end_time` 等 metadata：

- [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py:1160)

結論：

- 本地 session metadata 基本上已足以支撐 replay session chooser
- 缺的是「Prometheus 端是否已存在、是否需要覆蓋、若存在有哪些 TSDB-side 資訊」

### 2.3 Prometheus / Grafana 啟動現況

在本計畫開始時，Prometheus 一直被視為由 `5g-viz` 直接控制的本地 companion service：

- 啟動時 `pkill` 舊 Prometheus
- 重新起一個本地 Prometheus
- replay 模式使用 isolated temp TSDB
- 啟動後清除所有 `nwdaf_*`

Grafana query 語意已 session-aware，但底層 TSDB lifecycle 仍不 session-aware。

結論：

- 現有 Prometheus datasource / dashboard 已足夠
- 真正要改的是 TSDB lifecycle policy，而不是 Grafana query 本身

### 2.4 Topology filter / controls 現況

目前頁面已有可收合左側 filter sidebar：

- node filter
- edge filter
- all / none

對應實作：

- UI 結構：[frontend/index.html](/home/chingje/testbed/5g-viz/frontend/index.html:96)
- 邏輯：[frontend/topology.js](/home/chingje/testbed/5g-viz/frontend/topology.js:284)

這代表新的 transient effect controls 最自然的落點，不是再塞進 DVR bar，而是採用相似的
collapsible sidebar / panel pattern。

### 2.5 Config / profile 現況

目前 `5g-viz` 的 profile 設定主要由：

- `profiles/<profile>/.env`
- `profiles/<profile>/topology.yaml`

組成，其中 `config.py` 會直接 `load_dotenv()` 載入 profile `.env`，而 `start.sh` 與 `collector.py`
也都直接依賴環境變數。

這個模型目前能運作，但有兩個問題：

1. `.env` 內多數內容其實不是 secrets，而是一般操作設定
   - `WS_PORT`
   - `GRAFANA_BASE`
   - `PROMETHEUS_URL`
   - `UPF_EES_API_IPS`
   - `UPF_DATA_SUBNETS`
2. 設定語意被切碎在 shell、Python process env、`topology.yaml` 三層，不利於未來 Python control plane 收斂

此外，`topology.yaml` 目前不只承載 topology / visual config，也包含 `ssh_sources` 這種 collector 操作設定。
這讓「畫面如何呈現」與「後端從哪裡收 log」混在同一份檔案裡，語意邊界不夠清楚。

目前真正比較接近敏感資訊的只有：

- Grafana admin credentials
- SSH private key path / SSH 連線資訊

但即便如此，現況也多半只是本機實驗環境路徑與預設帳密，而不是高價值 application secret。

結論：

- 現況比較像是「因 shell 啟動方便而採用 `.env`」，不是「因 secrets 管理需求而必須用 `.env`」
- 這代表把主要設定模型收斂成較普通的 YAML config 是合理方向

---

## 3. 重構目標

### 3.1 Replay runtime

1. replay 只保留一條 Grafana data path：原始 session
2. replay `Play` 改以 historical relative window 播放
3. 正式移除 pseudo-live backend / API / cleanup
4. replay speed 從主功能移除，回放語意收斂成 `1x + pause + scrub + resume`

### 3.2 Prometheus persistence

1. Prometheus 不再在每次啟動後刪除所有 `nwdaf_*`
2. replay session metrics 可跨重啟保留
3. replay 啟動時能偵測：
   - session 是否存在於本地
   - session 是否已存在於 Prometheus
   - 若已存在，是否要保留或覆蓋
4. 顯示足夠 metadata 供使用者判斷覆蓋風險
5. 採單一 persistent TSDB 作為主方案，不引入雙 TSDB 架構

### 3.3 Visual effect controls

1. 將 transient effect duration 與 playback speed 解耦
2. 新增 runtime controls：
   - edge flash duration multiplier
   - node pulse duration multiplier
   - scrub/static edge window duration
   - presets：`0.5x / 1x / 2x`
   - reset
3. 以可收合 UI 呈現，和現有 filter sidebar 同風格

### 3.4 Config model simplification

1. 將 profile 主設定從 `.env` 收斂到 `config.yaml`
2. 移除 `.env` / process env 作為正常設定來源，改由 CLI `--profile` 選擇 profile
3. 讓 `setup.sh`、`start.sh` wrapper、Python control plane、backend 共用同一份設定模型
4. 保留 `topology.yaml` 作為 topology / visual defaults 專屬檔案，不把所有設定混進單一大 YAML

---

## 4. 非目標

1. 不把 session artifact 改成只依賴 Prometheus TSDB
2. 不在本 phase 重新設計 Grafana dashboard query 模型
3. 不在本 phase 引入多使用者 replay ownership
4. 不把 topology effect config 全部搬離 `topology.yaml`
5. 不保留 replay speed 與 pseudo-live 的雙軌相容
6. 不在本 phase 建立複雜 secrets manager；先假設現有設定多數是非敏感操作參數

---

## 5. 核心設計決定

## 5.1 Replay 的 canonical model：單一 session、單一時間軸、單一 Grafana source

正式設計決定：

- replay metrics 一律查 `session=<orig_session>`
- replay `Pause / Scrub` 用 absolute trailing window
- replay `Play` 用 historical relative window
- 不再建立 `_live_...` pseudo session

結果：

- `MetricPlayer` 可刪
- `/api/replay/play|pause|speed` 可刪
- pseudo session cleanup 可刪
- replay chart window 變更不再需要「重啟 pseudo-live stream」

### 為什麼接受拿掉 replay speed

這是一個 deliberate simplification，而不是 feature regression accident：

- relative window 天然綁定 wall clock
- 要保留非 `1x`，就又會回到 remap 或頻繁改 query 的複雜路徑
- 目前使用者需求已明確偏向「pause/scrub 觀察」而非「播放倍率」

因此本次重構把 replay speed 視為可移除能力，而不是必須保留的需求。

## 5.2 Session artifact 是 source of truth，Prometheus 是 persistent query cache

正式分工：

- **session dir**：canonical replay artifact
- **單一 Prometheus TSDB**：persistent metric cache / query store

這表示：

- 若 Prometheus 中已有 session，可直接重用既有 metrics，省略 backfill
- 若 Prometheus 中沒有，但本地 session dir 存在，可選擇 backfill
- 若兩邊都沒有，就不能 replay

需要明確補一句：

- **真正的 replay 仍依賴本地 session artifact**
- Prometheus 中既有 session 只能省略 metrics backfill，不取代 `events.jsonl` / `topology.yaml`

這個分工與目前 `README` 中「session 目錄才是可攜 replay 核心」的方向一致，但比現在更完整地把
Prometheus 的角色明確化。

## 5.3 Prometheus lifecycle 必須改成 session-aware，而不是 global wipe

未來 Prometheus startup policy：

- 不再在 app 啟動後執行 `delete_series {__name__=~"nwdaf_.+"}`
- replay mode 不再強制使用 temp TSDB
- TSDB path 一律使用單一持久目錄
- retention 改為顯式設定，而非依賴預設值
- `out_of_order_time_window` 採超大窗口策略，而不是雙 TSDB 隔離

建議新增的主設定欄位：

- `prometheus.retention_time`
- `prometheus.retention_size`
- `prometheus.out_of_order_time_window`

最終由 Python control plane 從 `config.yaml` 解析後，再轉成 Prometheus 啟動 flags。

### 為什麼本計畫不採雙 TSDB

本輪結論明確採用**單一 persistent TSDB** 作為主方案，不再把「live TSDB / replay TSDB 分開」當作
預設方向。原因如下：

1. 雙 TSDB 只是風險隔離，不是根因消除
2. 會增加操作與心智負擔，讓 session status / overwrite decision 變得更分裂
3. 產品上更直覺的語意是「所有 replay-able metrics 都在同一個 Prometheus 裡」

因此本計畫選擇正面面對單一 TSDB 的 ingest / retention 規則，並把 replay workflow 設計成可與其共存。

### TSDB 風險控制基線

需要先明講：

- Prometheus 沒有一個「無條件接受任意古老樣本」的絕對保證開關
- 但在我們可控制本機 Prometheus 啟動參數的前提下，可以把風險壓到很低

本計畫採用的 TSDB 風險控制基線如下：

1. `retention.time = 0`
   - 關閉 time-based retention，避免資料寫入後因時間過舊被自動刪除
2. `retention.size = 0`
   - 關閉 size-based retention，避免資料因容量策略被提早淘汰
3. `out_of_order_time_window = 99y` 或同等級的大窗口
   - 讓非常舊的 replay session 仍有機會被 ingest
4. overwrite 採 `delete session series -> backfill`
   - 不做直接重寫
5. backfill payload 保持 per-series timestamp ordering
   - 避免 receiver 端因亂序拒收

### 關於 `out_of_order_time_window`

這裡的設計結論是：

- 不找「關閉 too-old 檢查」的不存在選項
- 直接採用超大時間窗口策略，例如 `99y`

這不等於 Prometheus 絕對保證任何年代資料都會被接受，但在單機、可控啟動參數、且 replay session
timestamp 合法的前提下，這是最接近「讓它照單全收」的實務配置。

## 5.4 Replay session overwrite 是 explicit user decision，不是隱式 `--force-backfill`

目前 replay backfill 只分：

- 有資料就 skip
- `--force-backfill` 就重寫

未來改成更明確的 session-aware 決策：

1. 先查本地 session metadata
2. 再查 Prometheus session presence
3. 若 Prometheus 已存在，顯示：
   - session id
   - start / end time
   - event_count（本地）
   - Prometheus presence / sample availability
4. 由使用者選擇：
   - reuse existing Prometheus copy
   - overwrite and backfill again

本輪正式採 CLI-only，不規劃後續補前端 overwrite / reuse UI。

### overwrite 的正式定義

本輪設計將 overwrite 明確定義為：

1. 先刪除該 `session=<orig_session>` 的 `nwdaf_*` series
2. 確認刪除完成
3. 再重新 backfill

不採用「直接重寫同 session label 的樣本」做法，因為先刪再灌的風險更低，語意也更清楚。

## 5.5 Python control plane 取代 `start.sh` 成為主要入口

本輪結論也明確採用：

- `start.sh` 不再作為主要控制平面
- 新的 Python CLI / control plane 成為 Prometheus 與 replay 行為的主入口

原因：

1. 啟動流程已經包含大量決策，不再只是 shell 層級的 process launch
2. replay session reuse / overwrite / status 檢查不適合長期放在 shell script
3. Prometheus query / delete / backfill / status 等管理操作，更適合用 Python 實作

建議最終形態：

```text
uv run run.py live --profile <profile>
uv run run.py session-status <session_ref> --profile <profile>
uv run run.py replay <session_dir> --profile <profile> --backfill=auto
uv run run.py replay <session_dir> --profile <profile> --backfill=overwrite
uv run run.py replay <session_dir> --profile <profile> --backfill=skip
uv run run.py prom delete-session <session_id> --profile <profile>
uv run run.py prom backfill-session <session_dir> --profile <profile> --overwrite
```

`start.sh` 可保留為薄 wrapper，但不再承擔主要流程決策。

### CLI 與 FastAPI 不應各自實作一套 session 決策邏輯

這裡再補一條明確約束：

- `run.py` 不應自己重做一套 replay status / overwrite / backfill 判斷
- FastAPI 也不應自己重做另一套

較合理的分層應是：

```text
shared service layer
  ├─ config resolution
  ├─ session status
  ├─ overwrite planning
  ├─ Prometheus delete/backfill
  └─ replay startup decision

CLI entry script -> thin adapter
FastAPI routes   -> thin adapter
```

建議新增一層共用 service，例如：

- `ReplaySessionService`
- `PrometheusSessionService`

CLI 與必要的 read-only HTTP API 都只呼叫這層，不各自維護獨立商業邏輯。

## 5.6 Runtime visual controls 是前端 local UI state，不是 session artifact

新的 transient effect controls 不應寫回 session，也不應改動 topology config 檔案。

它應是：

- per-browser local state
- 可存 `localStorage`
- 對 live / replay 共用
- 只影響前端視覺表現，不影響事件本體與 metrics

因此建議將它建模為：

```text
VisualFxRuntimeConfig
  edge_duration_scale
  pulse_duration_scale
  static_effect_window_ms
```

而不是直接覆蓋 `topology.yaml`。

## 5.7 Config model：`config.yaml` 為主，移除 env-based profile config

本輪結論新增一條明確設計決定：

- profile 設定的 canonical source 不再是 `.env`
- 改為 `profiles/<profile>/config.yaml`
- `profiles/<profile>/topology.yaml` 仍專責 topology / edge style / visual defaults
- profile 的選擇改由 CLI `--profile` 決定
- 臨時測試變更不走 env override，而是複製新的 profile

### 為什麼不再以 `.env` 當主設定來源

根據目前專案現況，`.env` 內承載的大部分不是 secrets，而是一般操作設定：

- port
- base URL
- query interval
- group IDs
- IP mapping

這類設定更適合以結構化 config 檔表達，原因是：

1. 可讀性更高
2. 更適合 schema 驗證
3. 更適合被 Python control plane、backend、setup tooling 共用
4. 避免 shell script 與 Python 各自解析同一組 env key

### 建議的 profile 設定分工

```text
profiles/<profile>/
  config.yaml
  topology.yaml
```

其中：

- `config.yaml`
  - server / Grafana / Prometheus / collector / replay policy
- `topology.yaml`
  - nodes / aliases / edge styles / visual defaults

### 補充：collector source 定義應搬離 `topology.yaml`

為了避免設定邊界再次混亂，本計畫在此明確定義：

- `topology.yaml` 不再承載 `ssh_sources`
- SSH 連線資訊與 log source 定義改由 `config.yaml.collector` 管理

未來的責任邊界應是：

- `config.yaml`
  - SSH hosts
  - SSH key path
  - remote log paths / source definitions
  - Prometheus / Grafana / replay policy
- `topology.yaml`
  - node layout
  - alias mapping
  - edge styles
  - visual defaults

### 建議的 `config.yaml` 範圍

```yaml
server:
  ws_port: 8765

grafana:
  browser_base: /grafana
  api_base: http://localhost:3000/grafana
  admin_user: admin
  admin_pass: admin
  groups:
    - group-test-001
    - group-test-002
  refresh: 10s
  default_from: now-10m

prometheus:
  url: http://localhost:9090
  retention_time: 0
  retention_size: 0
  out_of_order_time_window: 99y
  query:
    time_interval: 10s
    panel_interval: 10s
    annotation_step: 10s
    prediction_offset: 10s

collector:
  ssh:
    5gc:
      host: 127.0.0.1
      port: 2222
      user: vagrant
      key_path: /path/to/private_key
  sources:
    - name: 5gc
      logs:
        - type: latest_subdir_file
          dir: ~/free5gc/log
          filename: free5gc.log
          source: free5gc
        - type: file
          path: ~/nwdaf.log
          source: nwdaf

mapping:
  upf_ees_api_ips:
    192.168.107.10: UPF-EES
    192.168.107.11: UPF-EES2
  upf_data_subnets:
    "10": UPF-EES
    "100": UPF-EES2

replay:
  backfill_default: auto
```

這不是要求一次把所有 key 命名完全定死，而是先確立：未來應以結構化 config 表達這些設定，且
collector source 不應再依附於 `topology.yaml` 的 env indirection。

### environment 的未來角色

本 phase 的設計結論是：

- 正常操作不依賴 env
- profile 選擇由 CLI `--profile` 處理
- 臨時實驗差異透過複製新的 profile 處理
- 若未來真的出現敏感資訊，再另外設計注入方式

也就是說：

- `config.yaml` 必須能完整描述一個可啟動 profile
- 不再新增 `.env`-only canonical settings

---

## 6. 目標架構

```text
session dir (meta.json + events.jsonl + topology.yaml)
        │
        ├─ replay load
        │
        ├─ if TSDB missing session -> optional backfill
        │
        └─ if TSDB has session    -> direct reuse

Prometheus TSDB (single persistent store)
        │
        ├─ live scrape
        ├─ replay backfill
        └─ long-lived session store

Profile config
        │
        ├─ config.yaml
        └─ topology.yaml

Frontend replay
        │
        ├─ topology/log: local _events playback
        ├─ paused/scrub: absolute trailing Grafana window
        └─ playing: historical relative Grafana window

Frontend visual fx controls
        │
        ├─ edge flash scale
        ├─ pulse scale
        ├─ static snapshot window
        └─ presets/reset

Python control plane
        │
        ├─ replay status
        ├─ overwrite decision
        ├─ config loading / validation
        ├─ session delete/backfill
        └─ Prometheus launch config
```

---

## 7. 模組級改造規劃

### 7.0 本輪模組化目標結構

在不改核心邏輯的前提下，Python 程式目錄收斂成：

```text
5g-viz/
├─ run.py                      # 主要 CLI 入口
├─ import_logs.py              # log import CLI shim
├─ config.py                   # 舊 constants 介面相容 shim
├─ backend/
│  ├─ app.py                   # FastAPI app 與 route / lifespan wiring
│  └─ grafana_proxy.py         # Grafana HTTP / WS proxy helpers
├─ runtime/
│  ├─ profile_config.py        # profile schema / loader
│  ├─ runtime_context.py       # process runtime context
│  └─ state.py                 # topology runtime state
├─ services/
│  ├─ collector.py             # SSH log collector
│  ├─ grafana_setup.py         # dashboard provisioning
│  ├─ prometheus_service.py    # Prometheus admin / status helpers
│  └─ replay_session_service.py
├─ replay/
│  ├─ parser.py                # line -> event parser
│  ├─ session_io.py            # session/meta I/O helpers
│  ├─ session_import.py        # log -> session importer
│  └─ metrics_backfill.py      # replay metric backfill encoding / remote write
├─ rules/
├─ frontend/
├─ profiles/
└─ sessions/
```

收斂原則：

- root 只保留真正的入口與相容 shim
- route / UI server、Prometheus / Grafana / SSH 外部整合、replay/session utilities 分層放置
- 不在本輪同時重寫核心行為；以搬移、抽出 helper、收斂 import 為主
- `rules/` 先保留原位置，避免 parser 規則系統在同一輪再做更深遷移

## 7.1 `start.sh`

現況問題：

- replay 使用 temp TSDB
- 啟動時清空所有 `nwdaf_*`
- retention 未顯式設定
- shell script 承擔了過多流程決策責任

重構方向：

1. 移除 replay temp TSDB 特例
2. 移除全域 `delete_series`
3. 顯式設定 retention flags
4. 顯式設定 `out_of_order_time_window`
5. 保留 admin API，但只給 session-scoped delete / overwrite 使用
6. 降格為 wrapper 或相容入口

本輪決策：

- 預設直接採永久 retention（`time=0`, `size=0`）
- 若使用者真的要 clean all，提供獨立 maintenance command，而不是啟動時自動做
- 主要入口完全轉向 Python CLI；`start.sh` 只保留薄 wrapper

## 7.2 `config.py` / profile config loader

現況問題：

- `config.py` 直接 `load_dotenv()`，把 `.env` 視為 canonical source
- 設定 schema 不明確，型別轉換散在程式碼中
- `start.sh`、`collector.py`、`main.py` 也各自直接讀 env，導致設定面分裂

重構方向：

1. 新增 `profiles/<profile>/config.yaml`
2. 讓 `config.py` 改成：
   - 讀 `config.yaml`
   - 回傳結構化 config object
3. 把目前零散的：
   - `_int_env()`
   - `_float_env()`
   - `_parse_kv()`
   收斂到統一的 config parsing / validation 層
4. 讓 backend 與 Python control plane 共用同一份 loader

建議輸出形態：

- `AppConfig`
- `GrafanaConfig`
- `PrometheusConfig`
- `CollectorConfig`
- `ReplayConfig`

補充要求：

- loader 需要一併覆蓋原本 `collector.py` 的 env-based config 來源
- `ssh_sources` 的解析責任應搬到 `config.yaml.collector.sources`
- 不應保留「collector 還是從 topology + env 讀、其他模組改讀 YAML」這種半套狀態

## 7.3 Python control plane / CLI（新）

本 phase 建議新增一個新的 Python 控制入口，承擔原本散落在 `start.sh`、`main.py` 與手動操作中的流程。

建議責任：

1. 啟動 Prometheus 並注入正確 TSDB flags
2. 查 session status（local + Prometheus）
3. 執行 session-scoped delete
4. 執行 backfill / overwrite
5. 決定 replay 啟動模式與參數

這一層應該是本輪重構的 orchestration 中心。

此外，它也應成為 config lifecycle 的唯一入口：

- parse `--profile`
- load `config.yaml`
- 驗證 config
- 再啟動 Prometheus / app

實作限制：

- CLI 不直接實作 Prometheus / replay 決策邏輯
- CLI 只呼叫 shared service layer
- 若保留 HTTP API，僅限前端真正需要的 read-only API，且也只呼叫 shared service layer

## 7.4 `main.py`

現況問題：

- replay session presence check 太粗，只看 ground-truth instant vector
- replay overwrite decision 沒有完整 metadata model
- pseudo-live 路徑與 replay 主流程仍糾纏在同一檔案

重構方向：

1. 新增 session presence / availability query helpers
2. 改用 range-safe session existence query，例如：
   - `count_over_time(nwdaf_session_anchor{session="..."}[...])`
3. 新增 session overwrite / delete helpers
4. 將 replay status / overwrite / backfill 決策抽到 shared service
5. 刪除：
   - `MetricPlayer` 生命週期接線
   - `/api/replay/play|pause|speed`
   - pseudo session cleanup
6. `main.py` 對外只保留 route / lifecycle wiring
7. 讓 replay lifespan 只剩：
   - load session
   - inspect Prometheus
   - optional backfill / overwrite
   - serve replay APIs

建議保留的 HTTP API：

- `GET /api/replay/session-status?session=<id>`

其中 `session-status` 應整合：

- local metadata
- TSDB presence
- 是否建議 backfill
- 是否存在 overwrite 風險

但這些欄位的組裝邏輯應來自 shared service，而不是 route handler 自己拼接。

本輪不把以下 mutation 能力設計成 browser-facing HTTP workflow：

- backfill
- delete-session-series
- overwrite decision

這些能力由 CLI 直接呼叫 shared service layer 完成。

## 7.5 `metric_player.py`

目標：

- 完整移除

前提：

- replay `playing` 正式改成 historical relative window
- speed 功能移除
- 所有 pseudo-live 相關 docs / API / cleanup 均已收掉

## 7.6 `frontend/events.js`

現況問題：

- 邏輯曾同時支援 backfill 與 pseudo-live mode switch
- chart window、speed、Grafana mode 相互牽扯

重構方向：

1. 保留單一 replay Grafana model：
   - `BACKFILL`
   - `RELATIVE_PLAY`
2. 刪除 pseudo-live 專用狀態與 API 呼叫
3. speed control 從 replay UI 移除
4. 將 historical relative range 計算整理成正式 helper，而不是實驗分支殘留邏輯
5. 不新增 replay overwrite decision UI，維持 CLI-only 策略

## 7.7 `frontend/topology.js`

現況問題：

- `_effectiveDuration()` 綁在 `_playbackSpeed`
- effect duration 只能從 config 與 playback speed 間接控制

重構方向：

1. 將 runtime effect config 抽出：
   - `edgeDurationScale`
   - `pulseDurationScale`
   - `staticWindowMs`
2. `_effectiveDuration()` 改依 action type 套用不同 scale
3. `renderStaticSnapshot()` 的 `edgeWindowMs` 改接 runtime config
4. `Topology.setPlaybackSpeed()` 可保留給 live DVR 或徹底改名，避免和 effect config 混淆

## 7.8 `frontend/index.html`

重構方向：

1. 保留現有 DVR bar
2. 在 filter sidebar 旁新增或整合一個 collapsible controls panel
3. controls 提供：
   - preset buttons：`0.5x / 1x / 2x`
   - edge duration input
   - pulse duration input
   - static edge window input
   - reset

UI 原則：

- 不與 replay playback speed 混名
- 明確標為 `Visual FX` 或 `Effects`
- 和 node/edge filter 同樣是「視覺觀察工具」，不是 runtime semantics

## 7.9 `topology.yaml`

重構方向：

- 保留作為 default visual config
- 不再承擔 runtime tuning 角色
- 不再承擔 collector / SSH source 定義

可加但非必要：

- `defaults.visual_fx.edge_scale`
- `defaults.visual_fx.pulse_scale`
- `defaults.visual_fx.static_window_ms`

若加入，僅作初始值來源，不代表 session 固定綁定這些值。

## 7.10 `setup.sh`

現況問題：

- 目前主要工作之一是複製 `.env.example -> profiles/<profile>/.env`
- 這使 setup 語意與 `.env` 綁得過深

重構方向：

1. 改成建立 `profiles/<profile>/config.yaml`
2. 若不存在，從新版範本產生初始檔
3. topology 仍維持複製 `topology.yaml`
4. setup phase 增加 config validation

本輪決策：

- 改為 `config.example.yaml`
- 不再保留 `.env.example` 作為正常設定入口

---

## 8. Replay session status / overwrite UX 規劃

## 8.1 CLI only

本輪正式採 CLI-only，不規劃前端 modal 或 overwrite chooser。

可能形式：

```bash
uv run run.py replay sessions/20260513T033836881 --profile default --backfill=auto
uv run run.py replay sessions/20260513T033836881 --profile default --backfill=overwrite
uv run run.py replay sessions/20260513T033836881 --profile default --backfill=skip
```

### `auto`

- TSDB 沒有 session -> backfill
- TSDB 已有 session -> reuse

### `overwrite`

- 先刪除該 session 的 `nwdaf_*` series
- 再重新 backfill

### `skip`

- 即使 TSDB 沒有 session，也不自動 backfill
- 讓使用者只測 topology / event replay

## 8.2 Session status payload

建議回傳：

```json
{
  "session_id": "20260513T033836881",
  "local": {
    "exists": true,
    "start_time": "...",
    "end_time": "...",
    "event_count": 3425,
    "profile": "default",
    "corrupted": false
  },
  "prometheus": {
    "exists": true,
    "anchor_present": true,
    "traffic_series_present": true,
    "sample_hint": 2
  },
  "recommended_action": "reuse"
}
```

這份 payload 主要供 CLI diagnostics 與必要的只讀 HTTP 檢視使用，不作為未來 overwrite UI 的前置設計。

---

## 9. Visual effect controls 規劃

## 9.1 使用者需求拆解

你現在的需求其實包含三類不同語意：

1. **倍率式調整**
   - `x2`
   - `x0.5`
2. **絕對時間調整**
   - edge flash 顯示多久
   - pulse 顯示多久
3. **reset**
   - 回到 config / default

因此不應只提供單一 slider，而應區分：

- `Preset scale`
- `Advanced values`
- `Reset`

## 9.2 建議控制項

### 基礎模式

- `0.5x`
- `1x`
- `2x`
- `Reset`

### 進階模式

- `Edge flash`
- `Pulse`
- `Static replay window`

其中：

- `Edge flash` 影響 `flashEdge()`
- `Pulse` 影響 `pulse()`
- `Static replay window` 影響 `renderStaticSnapshot()` 的 edge / pulse reconstruction window

## 9.3 Persistence

建議保存在 `localStorage`：

- 不影響 session artifact
- 符合「個別觀察者偏好」
- 重整頁面後保留

## 9.4 與 playback 的關係

正式決策：

- visual fx controls 與 playback speed 解耦
- 若 replay speed 被移除，effect controls 仍然存在
- live mode 也可使用同一套 visual tuning

---

## 10. 文件與知識面更新

本次重構完成後，文件需要同步大幅收斂。

### 需要更新

- `design/dvr/replay.md`
- `design/grafana/rendering.md`
- `design/grafana/setup.md`
- `guides/start-here/live-vs-replay.md`
- `guides/start-here/terminology.md`
- `guides/ui-workflows/grafana.md`
- `guides/ui-workflows/dvr-controls.md`
- `guides/troubleshooting/common-scenarios.md`
- `guides/mental-model/live-vs-replay-data-paths.md`

### 需要移除或改寫的觀念

- replay `playing` = pseudo-live session
- pre-seed
- `MetricPlayer`
- replay speed 會影響 pseudo-live chart
- replay chart window change 需要重啟 pseudo-live

### 新 canonical model

- replay `playing` = original session + historical relative window
- Prometheus = long-lived session store
- overwrite / reuse = explicit replay decision
- transient effects = independent visual controls

---

## 11. 實作順序建議

### 11.1 簡易進度表

| Phase | 主題 | 主要產出 | 狀態 |
|---|---|---|---|
| 1 | Config loader + CLI skeleton | `config.yaml` schema、shared loader、`run.py` 入口骨架 | Done |
| 2 | Prometheus lifecycle | persistent TSDB、retention / `out_of_order_time_window`、delete helper | Done |
| 3 | Replay status / overwrite | session status、CLI backfill policy、overwrite flow | Done |
| 4 | Remove pseudo-live | 刪除 pseudo-live API / runtime / cleanup | Done |
| 5 | Visual FX controls | runtime effect config、panel、presets / reset | Done |
| 6 | Modular structure refactor | backend / runtime / services / replay 分層與 root shim 收斂 | Done |
| 7 | Docs convergence | 移除舊 mental model、更新操作文件 | Planned |

狀態欄建議只用：

- `Planned`
- `In Progress`
- `Done`

### 11.2 實作節奏原則

整體順序應遵循這三條原則：

1. 先收斂設定與控制入口，再改 runtime 行為
2. 先建立可觀測的 session status / overwrite 能力，再移除 pseudo-live
3. 每次 commit 只跨一個明確主題，避免同一個 commit 同時混入 config、Prometheus、frontend UI 三種變更

因此本計畫預設採：

- **一個 phase 一個 commit**
- phase 內可以分段驗證、反覆測試，但不必為了驗證節點而切成多個小 commit

只有在以下情況才建議拆成多個 commit：

- 同一個 phase 內同時包含高風險遷移與獨立重構
- 某一段需要單獨回退
- 某個中間狀態本身就值得保留成清楚的歷史節點

### 11.3 Phase 1. Config loader + Python control plane groundwork

建議順序：

1. 定義 `config.yaml` schema 與預設 profile 範本
2. 實作 shared config loader
3. 讓 backend 先能讀 YAML config
4. 建 `run.py` skeleton
5. 最後才把 `start.sh` 降格成 wrapper

實作前確認的現況落差：

- [5g-viz/config.py](/home/chingje/testbed/5g-viz/config.py) 目前在 import time 直接載入 `profiles/<profile>/.env`，並以 module-level constants 形式提供設定；[5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py)、[5g-viz/grafana_setup.py](/home/chingje/testbed/5g-viz/grafana_setup.py)、[5g-viz/metric_player.py](/home/chingje/testbed/5g-viz/metric_player.py)、[5g-viz/rules/*.py](/home/chingje/testbed/5g-viz/rules) 都已直接依賴這組介面。
- 原本由 `start.sh` 承擔的啟動責任已收斂進 [5g-viz/run.py](/home/chingje/testbed/5g-viz/run.py)；Phase 1 完成時仍保留 wrapper，相依的 Prometheus policy 則留到 Phase 2 再處理。
- [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py) 目前仍直接讀 `PROFILE`、`SESSION_MODE`、`SESSION_PATH`、`FORCE_BACKFILL`、`PROMETHEUS_URL` 等 env；Phase 1 應先讓 `run.py` 成為唯一正常入口，再決定 app process 內部是改吃 structured settings，還是暫時由 `run.py` 建立最小相容層。
- [5g-viz/collector.py](/home/chingje/testbed/5g-viz/collector.py) 目前只理解 `topology.yaml` 中的 `ssh_sources + host_env/path_env` indirection；因此 `config.yaml.collector.sources` 上線時，第一刀應先由 loader 轉成 collector 可消費的結構，再決定是否連 collector 內部格式一起清掉。
- [5g-viz/import_logs.py](/home/chingje/testbed/5g-viz/import_logs.py) 目前透過 `os.environ[\"PROFILE\"]` 後設載入 [5g-viz/config.py](/home/chingje/testbed/5g-viz/config.py)；[5g-viz/session_import.py](/home/chingje/testbed/5g-viz/session_import.py) 則直接複製 profile `topology.yaml` 並從呼叫端取得 `grafana_groups`。Phase 1 需同步讓 importer 改成顯式載入 profile config，而不是繼續依賴 env side effect。
- [5g-viz/setup.sh](/home/chingje/testbed/5g-viz/setup.sh) 與 [5g-viz/README.md](/home/chingje/testbed/5g-viz/README.md) 目前都以 `.env` 為主流程；若 Phase 1 要一個 commit 收完，文件與 setup 入口必須一起切到 `config.yaml`，否則 repo 會在同一個 phase 內出現兩套初始化心智模型。

Phase 1 的相容策略建議：

- 不保留 runtime fallback 到 `.env`，但允許提供一次性的 profile migration helper，將舊 `profiles/<profile>/.env` 轉成 `config.yaml`。
- `config.py` 第一刀可以保留為 thin compatibility module，但其資料來源必須改成 shared YAML loader，而不是再直接 `load_dotenv`。
- `run.py` 應成為 `live` / `replay` / `prom ...` 的唯一正式入口。
- `collector`、`import_logs.py`、`main.py` 在 Phase 1 內至少都要切到 shared config loader，避免新舊設定來源並存。

建議 commit 策略：

- 預設整個 Phase 1 一個 commit
- commit 內容可同時包含 `config.yaml` 範本、shared loader、`run.py` skeleton、入口收斂

每次 commit 前至少驗證：

- `python3 -m py_compile` 可通過所有新改的 Python 檔
- 舊 profile 轉換後可啟動，或轉換失敗時有明確錯誤訊息
- `./setup.sh` / 新 CLI 不會產生壞掉的 profile 結構

Phase 完成判準：

- 同一份 profile 設定可被 backend 與 CLI 共用
- `run.py` 已成為唯一正式入口
- `collector` 不再依賴 `topology.yaml + env indirection` 取得 source 定義

目前已完成的實作進度：

- `profiles/<profile>/config.yaml` 與 root `config.example.yaml` 已建立
- shared YAML loader 已落地，並由 `config.py` 作為相容薄層提供既有 constants
- `uv run run.py live|replay|prom status --profile ...` 已可實際執行
- `setup.sh` 與 `README` 已改成 `config.yaml` workflow
- `main.py`、`collector.py`、`import_logs.py` 已接到 shared config flow
- smoke validation 已完成：
  - `python3 -m py_compile`
  - `bash -n setup.sh`
  - `uv run run.py prom status --profile default`
  - `uv run run.py live --profile default`
  - `uv run run.py replay sessions/20260513T033836881 --profile default`

### 11.4 Phase 2. Prometheus lifecycle groundwork

建議順序：

1. 收斂 Prometheus launch flags
2. 去掉 temp TSDB
3. 去掉 global wipe
4. 加 session-scoped delete helper
5. 補 retention / window diagnostics

建議 commit 策略：

- 預設整個 Phase 2 一個 commit
- phase 內可先逐步驗證 TSDB 路徑、global wipe 移除、delete helper，再一次提交

每次 commit 前至少驗證：

- 啟動後 Prometheus 可正常 listen
- 關掉程式再重開後，舊 session metrics 仍可查到
- session-scoped delete 不會誤刪其他 session
- backfill 舊 session 時沒有立即出現明顯 too-old / retention 問題

Phase 完成判準：

- Prometheus 不再把自己當 disposable cache
- 單一 persistent TSDB 可跨重啟保留 replay session

目前已完成的實作進度：

- `run.py` 啟動 Prometheus 時已固定使用 persistent `~/prometheus/data`，不再為 replay 建立 `/tmp/prometheus-replay-*`
- Prometheus 啟動時不再全域刪除 `nwdaf_*`
- retention 仍由啟動 flags 顯式設定，`out_of_order_time_window` 則由 `run.py` 寫入 managed `~/prometheus/prometheus.yml`
- session-scoped delete helper 已落地為 shared service，並提供 `uv run run.py prom delete-session <session_id> --profile ...`
- `main.py` 的 replay backfill presence check 已改成 TSDB series existence，而不是對歷史 session 不可靠的 instant query
- `start.sh` 已移除，`run.py` 成為唯一正式入口
- `run.py` 會在啟動前同步 managed `prometheus.yml`，因此當前本機 Prometheus `3.5.1` 也能透過 YAML config 接受較舊 replay session 的 backfill
- `run.py` 支援產生 `systemctl --user` 用的 `5g-viz-prometheus.service`，並已收斂成「Prometheus 需先常駐啟動；5g-viz 只做 reuse + config sync + reload，不再提供本地 fallback spawn」

### 11.5 Phase 3. Replay status / overwrite model

建議順序：

1. backend session status query helpers
2. CLI `session-status`
3. CLI `replay --backfill=auto|overwrite|skip`
4. overwrite = delete then backfill

建議 commit 策略：

- 預設整個 Phase 3 一個 commit
- phase 內先把 session status、CLI、overwrite flow 串好並驗證，再一次提交

每次 commit 前至少驗證：

- 本地有 session、Prometheus 無 session 時，`auto` 會 backfill
- 本地有 session、Prometheus 已有 session 時，`auto` 會 reuse
- `overwrite` 確實是先刪再灌
- status 輸出能正確顯示 `event_count`、time range、Prometheus presence

Phase 完成判準：

- 使用者可不靠手動 Prometheus API 操作就完成 replay reuse / overwrite 決策
- 已實作 `uv run run.py session-status <session_ref> --profile ...`
- 已實作 `uv run run.py replay <session_dir> --profile ... --backfill=auto|overwrite|skip`

### 11.6 Phase 4. Remove pseudo-live runtime

建議順序：

1. 先確認 frontend 已完全只走 original session + relative play
2. backend 刪除 `/api/replay/play|pause|speed`
3. 刪除 `MetricPlayer`
4. 刪除 pseudo session cleanup 與殘留 docs / config hooks

建議 commit 策略：

- 預設整個 Phase 4 一個 commit
- phase 內先確認 frontend runtime 穩定，再一次移除 backend pseudo-live 殘留並提交

每次 commit 前至少驗證：

- replay `Play` 時 Grafana 仍平滑更新，不再依賴 pseudo-live
- `Pause / Scrub / Resume` 不壞
- browser network 中不再出現 `/api/replay/play|pause|speed`
- backend 啟動與關閉流程不再引用 pseudo-live 物件

Phase 完成判準：

- pseudo-live 只存在於 git history，不再存在於 runtime 與主要文件中

目前進度：

- frontend replay 已只走 original session + historical relative Grafana window
- backend 已刪除 `/api/replay/play|pause|speed`
- `MetricPlayer` 與 pseudo-session cleanup 已從 runtime 移除
- README 已改成以 original session replay model 說明

### 11.7 Phase 5. Visual FX controls

建議順序：

1. 抽 runtime effect config
2. 補 presets / reset
3. 補 advanced inputs
4. 接 `localStorage`

目前 scope 收斂：

- 將現有 `Speed` 控制正式改造成 `Visual FX` 控制，不再承載 replay / live timeline speed 語意
- 視覺效果調校需與 playback semantics 完全解耦；replay 仍固定 `1x + pause + scrub + resume`
- 第一輪以 frontend runtime 設定為主，不先擴張 profile schema 或 backend config surface
- presets 至少提供 `0.5x / 1x / 2x`
- advanced inputs 應以細項 duration 為主，而不是全域 window：
  - per-edge flash duration
  - pulse default duration
  - pulse explicit override duration
- pulse duration 的使用者語意應是「總可見時間」，而不是動畫單段時間
- 實作上可將 pulse animation 拆成兩段，但使用者輸入值與 topology default 都應代表 total duration
- paused / scrubbed static snapshot 不應再使用全域 `static window`
- static snapshot 顯示判斷應改成 per-event active range：
  - edge event 在 `eventMs <= targetMs <= eventMs + edgeDuration` 時可見
  - pulse event 在 `eventMs <= targetMs <= eventMs + pulseDuration` 時可見
- `pause -> play` 恢復時，若當前時間點已有 active edge / pulse，播放不應直接清空畫面
- resumed playback 必須延續當前 active effects 的剩餘時間：
  - edge / pulse 需能計算 `remainingMs`
  - resumed playback 先播完 residual effects，再接續未來事件
- static snapshot、paused view、resumed playback 三者必須共享同一套 active-range 定義
- 需提供 `Reset` 回到預設值
- 設定需在 live 與 replay 都即時生效
- 設定需寫入 `localStorage`，重整後保留使用者偏好
- timeline 需支援鍵盤左右方向鍵 step scrub
- step scrub 秒數需可在頁面上調整，不限定為 5 秒

目前實作收斂重點：

- `frontend/index.html` 已將舊的 `Speed` dropdown 收斂為 `FX` controls 與 `Step` 控制
- `frontend/events.js` 已移除 replay speed 對 timeline 的影響，改由 per-edge / per-pulse duration 與 keyboard step 控制處理前端互動
- `frontend/topology.js` 已移除 `_playbackSpeed` 依賴，改成獨立的 visual FX runtime config
- scrub / paused static snapshot 與 resumed playback 已共用 per-event active range 判斷

這個 phase 不處理：

- 不恢復 replay speed
- 不改 replay timeline / Grafana window 語意
- 不改 Prometheus lifecycle、session overwrite / reuse、或 replay backfill policy
- 不先把 visual defaults 移入新的後端設定來源；若需要 profile-based defaults，留待後續獨立處理

建議 commit 策略：

- 預設整個 Phase 5 一個 commit
- phase 內可先完成 runtime config、再接 UI、最後調整 persistence，但整體一起提交即可

每次 commit 前至少驗證：

- `0.5x / 1x / 2x / reset` 只影響 visual effect，不影響 replay timeline
- edge / pulse duration 調整後立即生效
- paused / scrubbed snapshot 與播放中的 effect 可見範圍語意一致
- paused 當下可見的 edge / pulse，在按下 `Play` 後會延續剩餘時間，而不是瞬間消失
- 左右方向鍵可依設定步長移動 timeline，且不會在輸入欄位 focus 時誤觸
- live mode 與 replay mode 都能使用
- 重整頁面後設定行為符合預期

Phase 完成判準：

- visual tuning 與 playback semantics 完全解耦

目前進度：

- frontend 已將舊 `Speed` dropdown 改為 `FX` controls
- replay / live timeline 已不再受 visual effect preset 影響
- `0.5x / 1x / 2x` presets、`Reset`、`localStorage` persistence 已落地
- `topology.js` 已改為使用獨立的 visual FX runtime config，而不是 `_playbackSpeed`
- advanced inputs 已收斂為 per-edge / per-pulse duration
- static snapshot 已改為 per-event active range 判斷，而不是全域對稱 window
- pulse duration 已收斂為 user-facing total duration，內部動畫再自動拆成兩段
- timeline 已支援可設定秒數的左右方向鍵 step scrub

### 11.8 Phase 6. Modular structure refactor

建議順序：

1. 先定義目標模組邊界與 root 保留入口
2. 先搬 `profile_config` / `runtime_context` / `state` 到 `runtime/`
3. 再搬 `collector`、`prometheus_service`、`replay_session_service`、`grafana_setup` 到 `services/`
4. 再搬 `parser`、`session_io`、`session_import` 與 replay metrics backfill 到 `replay/`
5. 最後把 server 入口直接切到 `backend/app.py`

建議 commit 策略：

- 預設整個 Phase 6 一個 commit
- 可先在 working tree 內逐步搬動與驗證，再一次提交

每次 commit 前至少驗證：

- `python3 -m py_compile` 可通過所有搬動後的 Python 模組
- `uv run run.py prom status --profile default` 正常
- `uv run run.py live --profile default` 可啟動
- `uv run run.py replay <existing-session> --profile default --backfill=auto` 可啟動
- `uv run import_logs.py --help` 與 `uvicorn backend.app:app` 入口不壞

Phase 完成判準：

- root 不再堆放主要業務模組
- `backend/`、`runtime/`、`services/`、`replay/` 責任邊界清楚
- 舊入口僅作 shim 或正式入口，不再承載實際業務邏輯
- 不改變 live / replay / Prometheus / importer 的對外行為

目前已完成的實作進度：

- `run.py` 已直接以 `backend.app:app` 啟動 uvicorn，`main.py` 不再保留
- `import_logs.py` 已收斂為 CLI shim，實際 importer CLI 移至 `replay/import_logs_cli.py`
- `profile_config` / `runtime_context` / `state` 已搬至 `runtime/`
- `collector`、`grafana_setup`、`prometheus_service`、`replay_session_service` 已搬至 `services/`
- `parser`、`session_io`、`session_import` 已搬至 `replay/`
- `backend/grafana_proxy.py` 已從 server module 抽出 Grafana HTTP / WS proxy helper
- root 現在只保留 `run.py`、`import_logs.py`、`config.py` 與專案設定檔
- smoke validation 已完成：
  - `python3 -m py_compile`
  - `uv run run.py prom status --profile default`
  - `uv run run.py replay sessions/20260517T125146187 --profile default --backfill=auto`
  - `uv run run.py live --profile default`
  - `uv run import_logs.py --help`

### 11.9 Phase 7. Docs convergence

建議順序：

1. 先改 start / setup / profile config 文件
2. 再改 replay / Grafana mental model
3. 最後補 troubleshooting

建議 commit 策略：

- 預設整個 Phase 6 一個 commit
- 若文件量過大，可視情況拆成少數文件 commit，但不是預設要求

每次 commit 前至少驗證：

- 文件內已不再把 replay `playing` 描述為 pseudo-live
- 文件內 profile 設定來源與實作一致
- 新 CLI / 啟動方式能直接對照文件操作

Phase 完成判準：

- 新讀者不需要知道 pseudo-live 歷史也能理解現在系統

### 11.10 每次 commit 前的共用驗證清單

不論在哪個 phase，每次 commit 前都應至少做以下檢查：

1. 語法層
   - Python 檔做 `py_compile`
   - 前端 JS 至少做語法檢查，例如 `node --check`
2. 啟動層
   - `5g-viz` 能正常啟動
   - Prometheus / Grafana / backend 端口與 proxy 不壞
3. 核心路徑
   - live mode 至少能開頁
   - replay mode 至少能載入一個既有 session
4. 回歸風險
   - session list / session info API 不壞
   - Grafana iframe URL 生成邏輯不壞
5. 文件一致性
   - 若 commit 改了操作方式、設定來源、API 語意，文件要同步更新或至少在 plan 中標記未完成項

---

## 12. 風險與注意事項

### 12.1 Prometheus 長期保存帶來的資料累積

一旦不再清空 `nwdaf_*`，需要正視：

- retention
- `out_of_order_time_window`
- disk growth
- 舊 session 數量增加後的 query cost

因此 retention 與 ingest window 都不應再留空，而要顯式設計。

### 12.2 「session 已存在」不代表資料一定完整

Prometheus presence check 只能回答「有沒有樣本」，不能保證：

- backfill 是否完整
- session 是否被中途中斷
- schema 是否與當前 dashboard 相容

因此 overwrite 選項仍應保留。

### 12.3 超大 `out_of_order_time_window` 是實務策略，不是絕對保證

把 `out_of_order_time_window` 設成 `99y` 是本計畫偏好的實務策略，但仍應明確承認：

- 這是「把接受窗口放得極寬」
- 不是「關閉 too-old 檢查」
- 也不是 Prometheus 對任何古老樣本的絕對 ingest 契約

系統仍應保留：

- backfill 失敗訊息
- session status diagnostics
- overwrite / retry 路徑

### 12.4 文件技術債會很多

目前整份 docs 對 pseudo-live 的解釋很多，若程式先改、文件沒跟上，讀者會立即混亂。

### 12.5 Visual controls 容易和 playback controls 混淆

UI 命名若不清楚，使用者會誤以為：

- `2x` 是 replay speed
- 其實是 edge / pulse duration scale

因此 panel 名稱、文案與 reset 語意要特別清楚。

### 12.6 Config migration 需要處理相容期

若從 `.env` 轉成 `config.yaml`，需要注意：

- 舊 profile 目錄的遷移
- `config.yaml` 欄位與內容要對照 `profiles/default/.env` 的實際使用值，而不只對照 `.env.example`
- README / setup 指令同步更新
- 是否提供一次性的 migration helper，把舊 `.env` profile 轉成 `config.yaml`

本輪較偏向：

- 不保留 runtime fallback 到舊 `.env`
- 需要相容時，提供明確 migration 步驟或轉換工具

---

## 13. 預計修改範圍

這份規劃對應到的程式與文件範圍，預期至少會碰到：

### 程式

- `5g-viz/run.py`
- [5g-viz/prometheus_service.py](/home/chingje/testbed/5g-viz/prometheus_service.py)
- `5g-viz/run.py`
- [5g-viz/main.py](/home/chingje/testbed/5g-viz/main.py)
- [5g-viz/metric_player.py](/home/chingje/testbed/5g-viz/metric_player.py)
- [5g-viz/frontend/events.js](/home/chingje/testbed/5g-viz/frontend/events.js)
- [5g-viz/frontend/topology.js](/home/chingje/testbed/5g-viz/frontend/topology.js)
- [5g-viz/frontend/index.html](/home/chingje/testbed/5g-viz/frontend/index.html)
- [5g-viz/config.py](/home/chingje/testbed/5g-viz/config.py)
- [5g-viz/setup.sh](/home/chingje/testbed/5g-viz/setup.sh)
- `5g-viz/config.example.yaml`
- `5g-viz/profiles/default/config.yaml`
- `5g-viz/*service*.py` 或等價 shared service modules
- [5g-viz/profiles/default/topology.yaml](/home/chingje/testbed/5g-viz/profiles/default/topology.yaml)

### 文件

- `docs/5g-viz/design/dvr/*`
- `docs/5g-viz/design/grafana/*`
- `docs/5g-viz/guides/start-here/*`
- `docs/5g-viz/guides/ui-workflows/*`
- `docs/5g-viz/guides/mental-model/*`
- `docs/5g-viz/guides/troubleshooting/*`

---

## 14. 建議的下一步

在進入實作前，目前已確認的決策如下：

1. Prometheus 預設直接採永久 retention + 超大 `out_of_order_time_window`
2. Python control plane 採 `uv run run.py ...` 形式作為主要入口
3. profile 設定完全收斂到 `config.yaml`，不保留 env override 作為正常設定來源
4. profile 選擇改由 CLI `--profile` 傳入，不再依賴 `PROFILE`
5. 臨時測試差異透過複製新 profile 處理，不靠 env 覆蓋
6. session overwrite / reuse 決策採 CLI-only，未來也不規劃前端 UI

目前 Phase 1 到 Phase 5 已完成，因此下一步應直接進入：

1. Phase 7：收斂 README、操作文件與舊有心智模型
