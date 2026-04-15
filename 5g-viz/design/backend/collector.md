# Collector

本文描述 `collector.py` 在 `5g-viz` 中的責任、輸入設定與失敗處理方式。內容以目前 `collector.py` 與 `main.py` 的實作為準。

## 1. 定位

`collector.py` 只在 `live` 模式下啟動，負責把遠端 VM 上的 log 行轉成 queue item，交給 `main.py` 的 `_process_queue()` 做後續 parser 與 side effects。

它目前不負責：

- 解析 log 內容
- 寫入 session 檔案
- 直接更新 Prometheus metrics
- 在 replay 模式下讀取 `events.jsonl`

換句話說，`collector` 的責任邊界是「把遠端 log 穩定送進本地 asyncio queue」。

## 2. 設定來源

`main.py` 在 live mode 啟動時，會先載入目前 profile 的 `topology.yaml`，再把其中的 `ssh_sources` 傳給：

```python
collector.start(_queue, _topo_config.get("ssh_sources", []))
```

每個 `ssh_sources` entry 至少包含：

- `name`：來源名稱，用於 log 訊息
- `host_env` / `port_env` / `user_env` / `key_env`：SSH 連線資訊對應的環境變數名稱
- `logs`：要 tail 的 log 定義清單

`collector` 不直接把 host、port、key 寫在 YAML 裡，而是用 `_build_conn_cfg()` 在執行時讀取 profile `.env`：

- `host` 預設 `127.0.0.1`
- `port` 預設 `22`
- `username` 預設 `vagrant`
- `client_keys` 來自 `key_env`
- `known_hosts` 固定設為 `None`

這代表：

- `topology.yaml` 保存的是「要讀哪些環境變數」
- 實際 SSH 憑證仍留在 profile `.env`

## 3. 支援的 log 定義

目前 `logs` 支援兩種路徑解析方式。

### 固定路徑

若 log entry 提供 `path_env`，`collector` 會直接讀取該環境變數對應的完整遠端路徑，並對它執行：

```text
tail -F <path>
```

### 最新子目錄

若 log entry 設了：

- `dir_env`
- `filename`
- `latest_subdir: true`

則 `collector` 會：

1. 先在遠端執行 `ls -t <dir>/ | head -1`
2. 找出最新子目錄名稱
3. 組成 `<dir>/<subdir>/<filename>`
4. 對該檔案執行 `tail -F`

這是目前 `free5gc.log` 使用的模式，用來跟隨最新一次 NF 啟動時產生的 log 目錄。

## 4. 執行模型

### Source 層級

`start(queue, sources)` 會對每個 source 啟動一個 `_connect_and_tail()` task，並用 `asyncio.gather()` 長期維持。

### 連線層級

每個 `_connect_and_tail()` 都會：

1. 建立一條 `asyncssh.connect(...)`
2. 依該 source 底下的每個 log 建立 background task
3. 等待其中任一 task 結束
4. 取消其餘 task
5. 重新建立整個 source 的 SSH 連線與 tail task

因此 reconnect 的單位是「單一 source」，不是單一 log。

### Log 層級

每個 log path 都由 `_tail_log()` 負責：

- 透過 `conn.create_process("tail -F ...")` 啟動遠端程序
- 逐行讀取 `proc.stdout`
- 每讀到一行就 `queue.put(...)`

推入 queue 的資料格式固定為：

```json
{
  "source": "free5gc" | "nwdaf",
  "line": "<原始 log 行>"
}
```

`collector` 不在這個階段附加 parser 欄位，也不做時間戳處理。

## 5. 最新子目錄的切換行為

對 `latest_subdir: true` 的 log，`collector` 除了 tail 目前檔案，還會另外啟動 `_watch_subdir()`。

`_watch_subdir()` 每 30 秒重新檢查一次 `ls -t <dir>/ | head -1`。若最新子目錄改變，會：

- 印出重新連線訊息
- 結束 `_watch_subdir()` task
- 讓 `_connect_and_tail()` 因 `FIRST_COMPLETED` 返回
- 取消舊的 tail task
- 重新解析新子目錄並重新 tail

因此它不是在原連線內熱切換檔案，而是靠整個 source reconnect 來切到新路徑。

## 6. 失敗處理與重試

### 沒有最新子目錄可讀

若 `latest_subdir` 模式下暫時找不到任何子目錄，`collector` 會：

- 印出 `no log dir yet`
- 每 5 秒重試一次，直到有結果

### 遠端 `tail -F` 結束

`_tail_log()` 捕捉 `asyncssh.ProcessError` 後不會直接失敗，而是等待 2 秒再重新啟動 `tail -F`。

這讓遠端服務重啟、檔案輪替或短暫 EOF 時，讀取能自動恢復。

### SSH 連線錯誤

`_connect_and_tail()` 會捕捉：

- `OSError`
- `asyncssh.Error`

出錯時會印出錯誤並在 5 秒後整體重連。

## 7. 與主流程的介面

`collector` 對主應用只暴露一個 async API：

```python
await collector.start(queue, sources)
```

它不需要知道 parser rule、session ID 或 state。真正的下游副作用都在 `main.py`：

1. `_process_queue()` 從 queue 取出 item
2. `parser.parse_line(...)` 產生事件
3. 事件再被寫入：
   - `_events`
   - `events.jsonl`
   - Prometheus metrics
   - WebSocket 廣播

因此 queue item 是 `collector` 與其餘 backend 的唯一資料契約。

## 8. 目前限制

- queue 目前沒有顯式上限；若 parser 或廣播比 collector 慢，記憶體壓力會累積在本地 queue
- 遠端命令依賴 shell 可執行 `ls`、`head`、`tail -F`
- `latest_subdir` 目前只支援「取最新一層子目錄」，不支援更複雜的目錄規則
- `collector` 只支援 SSH source；本機檔案、journald、Kafka 等其他來源目前不在設計範圍內
