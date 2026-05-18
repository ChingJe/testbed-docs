# Collector

本文描述目前 `services/collector.py` 在 `5g-viz` 中的責任、設定來源與失敗處理方式。

## 1. 定位

collector 只在 `live` 模式下啟動，負責把遠端 VM 上的 log 行轉成 queue item，交給 backend 的 live consumer loop 做後續 parser 與 side effects。

它不負責：

- 解析 log 內容
- 寫入 session 檔案
- 直接更新 Prometheus metrics
- replay 模式下讀取 `events.jsonl`

換句話說，collector 的責任邊界就是：

> 把遠端 log 穩定送進本地 `asyncio.Queue`

## 2. 設定來源

目前 collector sources 定義在：

- `profiles/<profile>/config.yaml -> collector.sources`

不再透過 `topology.yaml` 的 `ssh_sources` 或 `*_env` 間接解析。

每個 source 目前至少包含：

- `name`
- `host`
- `port`
- `user`
- `key_path`
- `logs`

而每個 log 目前支援：

- 固定 `path`
- `latest_subdir + dir + filename`

## 3. 支援的 log 定義

### 固定路徑

若 log entry 提供 `path`，collector 會直接對該路徑執行：

```text
tail -F <path>
```

### 最新子目錄

若 log entry 設了：

- `dir`
- `filename`
- `latest_subdir: true`

則 collector 會：

1. 在遠端執行 `ls -t <dir>/ | head -1`
2. 找出最新子目錄名稱
3. 組成 `<dir>/<subdir>/<filename>`
4. 對該檔案執行 `tail -F`

這個模式主要用在 `free5gc` 類型的最新啟動目錄。

## 4. 執行模型

### source 層級

`start(queue, sources)` 會對每個 source 啟動一個 `_connect_and_tail()` task，並用 `asyncio.gather()` 長期維持。

### 連線層級

每個 `_connect_and_tail()` 都會：

1. 建立一條 `asyncssh.connect(...)`
2. 依該 source 底下的每個 log 建立 background task
3. 等待其中任一 task 結束
4. 取消其餘 task
5. 重新建立整個 source 的 SSH 連線與 tail task

因此 reconnect 的單位是單一 source，而不是單一 log。

### log 層級

每個 log path 都由 `_tail_log()` 負責：

- 透過 `conn.create_process("tail -F ...")` 啟動遠端程序
- 逐行讀取 `proc.stdout`
- 每讀到一行就 `queue.put(...)`

推入 queue 的資料格式固定為：

```json
{
  "source": "free5gc" | "nwdaf",
  "line": "<raw log line>"
}
```

collector 不在這個階段附加 parser 欄位，也不做時間戳處理。

## 5. 與 backend 主流程的介面

目前對主應用只暴露一個 async API：

```python
await collector.start(queue, sources)
```

之後的責任由 backend 接手：

1. live consumer loop 從 queue 取出 item
2. `replay/parser.py` 產生結構化事件
3. 事件再被寫入：
   - `_events`
   - `events.jsonl`
   - Prometheus metrics
   - WebSocket 廣播

因此 queue item 是 collector 與其餘 backend 的唯一資料契約。

## 6. 最新子目錄的切換行為

對 `latest_subdir: true` 的 log，collector 會另外啟動 watcher，定期重新檢查目前最新子目錄。

若最新子目錄改變，會：

- 結束目前 source 內的 tail task
- 讓 source reconnect
- 重新解析新子目錄並重新 `tail -F`

因此它不是在原連線內熱切換檔案，而是靠整個 source reconnect 切到新路徑。

## 7. 失敗處理與重試

### 找不到最新子目錄

若 `latest_subdir` 模式下暫時找不到任何子目錄，collector 會記 warning 並定期重試，直到有結果。

### 遠端 `tail -F` 結束

`_tail_log()` 捕捉遠端程序異常後不會直接放棄，而是等待後再重新啟動 `tail -F`。

這讓遠端服務重啟、檔案輪替或短暫 EOF 時，讀取能自動恢復。

### SSH 連線錯誤

`_connect_and_tail()` 會捕捉 `OSError` 與 `asyncssh.Error`，之後按 source 單位整體重連。

## 8. 目前限制

- queue 目前沒有顯式上限；若 parser 或廣播比 collector 慢，記憶體壓力會累積在本地 queue
- 遠端命令依賴 shell 可執行 `ls`、`head`、`tail -F`
- `latest_subdir` 目前只支援取最新一層子目錄
- collector 目前只支援 SSH source；本機檔案、journald、Kafka 等來源不在目前範圍內
