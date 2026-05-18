# Parser

> Historical note: the event semantics described here remain useful, but file/module references may still point to the older pre-refactor backend layout.

本文描述 `parser.py` 與 `rules/` 如何把原始 log 行轉成 `5g-viz` 的事件資料模型。

## 1. 定位

`parser.py` 是 live pipeline 中從 raw log 到 structured event 的第一層語意轉換。

它的輸入是：

- `collector.py` 推入 queue 的單行文字
- source tag，例如 `free5gc`、`nwdaf`

它的輸出是：

- 一筆 event dict
- 或 `None`，表示此 log 行目前沒有對應的事件語意

parser 本身不更新 state、不寫檔，也不更新 metrics。

## 2. 基本解析流程

`parse_line(line, source="")` 的流程固定分成兩段：

1. 先用 `BASE` regex 解析 logrus 風格的基礎欄位
2. 再依 `rules.ALL_RULES` 逐條比對，找出第一個符合的規則

### 基礎欄位

目前 `BASE` 會從一行 log 中嘗試擷取：

- `time`
- `level`
- `msg`
- `CAT`
- `NF`
- `extra`

其中 `CAT` 與 `NF` 是 optional。剩餘沒有被前段 regex 吃掉的部分會先放進 `extra`，再用一條 key/value regex 抽成 `extra_kv`。

基礎解析失敗時，`parse_line()` 直接回傳 `None`。

## 3. `NF` / `CAT` 的 fallback

程式裡有一個目前很重要的補救邏輯：

```python
if not base.get("nf"):
    base["nf"] = base["extra_kv"].get("NF")
if not base.get("cat"):
    base["cat"] = base["extra_kv"].get("CAT")
```

原因是部分 log，特別是 UPF 相關輸出，會把 `NF` / `CAT` 放在較後面的 key/value 區段，而不是固定出現在 `msg` 前方。

因此 parser 的實際比對來源不是只有 base regex 的命名群組，也包含 `extra_kv` fallback 結果。

## 4. `source` 欄位的角色

`collector` 在 queue item 中提供 `source`，`parser` 會把它掛到 `base["source"]`。

這個欄位不是原始 log 內建的字串，而是 collector 根據 `ssh_sources[*].logs[*].source` 補上的來源標籤。rule 可以拿它做精確過濾，例如：

- 同一台 VM 上 tail 多種 log
- 需要區分 `free5gc` 與 `nwdaf` 來源時

## 5. Rule registry 與載入方式

`rules/__init__.py` 會動態掃描 `rules/` 底下的模組，收集：

- `RULES`
- `METRIC_HANDLERS`
- `set_session_id()`

parser 實際使用的是匯總後的 `ALL_RULES`。

目前 rule 模組分工如下：

- `rules/smf.py`：SMF 啟動與 `SMF -> UPF` 訂閱相關事件
- `rules/nwdaf_sub.py`：Consumer / NWDAF / SMF 訂閱鏈路
- `rules/nwdaf.py`：NWDAF 內部分析、retraining、ADRF 與 metrics 相關事件

### 現況載入順序

`rules/__init__.py` 目前是用 `pkgutil.iter_modules()` 掃描 `rules/` 目錄，再依掃描結果逐一 import。以目前 repo 內的檔名與執行環境，實際順序是：

1. `nwdaf.py`
2. `nwdaf_sub.py`
3. `smf.py`

因此 `ALL_RULES` 目前也是按這個模組順序串接。再加上 parser 採 first-match-wins，rule 的先後會直接影響哪一條規則先吃到某筆 log。

## 6. Rule match 條件

每條規則至少包含：

- `match`
- `event`
- `build`

其中 `match` 支援四種條件：

- `nf`：與 base `nf` 完整相等
- `cat`：與 base `cat` 完整相等
- `source`：與 collector source tag 完整相等
- `msg`：一個 regex，先比 `base["msg"]`，若失敗再比整行原始 `line`

`msg` 的「先比 message，再回退整行」讓 parser 可以同時處理：

- 純 message 型規則
- 需要從整行其他區段抓資訊的規則

## 7. First-match-wins 行為

`parse_line()` 會依 `ALL_RULES` 順序逐條嘗試；第一個匹配成功的 rule 會立刻：

1. 建立 `{"type": rule["event"], "time": base["time"]}`
2. 呼叫 `rule["build"](msg_match, base)` 產生額外欄位
3. 回傳最終 event

因此目前 parser 的語意是 first-match-wins，而不是多重命中。

這代表兩件事：

- 同一行 log 不會同時產生多個事件
- 規則若有重疊，前面的 rule 會遮蔽後面的 rule

## 8. Event payload 的形成方式

parser 產生的 event 一定至少有：

```json
{
  "type": "<event_type>",
  "time": "<ISO 8601 timestamp>"
}
```

其餘欄位完全由 `build()` 決定。這些欄位通常來自兩個來源：

- `msg_match.group(...)`：從 regex 命名群組抽出
- `base` / `base["extra_kv"]`：從原始 key/value 欄位讀出

例如：

- `rules/smf.py` 會從 `selected_upf_api_root` 推導 `UPF-EES` 或 `UPF-EES2`
- `rules/nwdaf.py` 會把 `aggregated_slot` 中的大量欄位縮成前端與 metrics 真正需要的子集

因此 event log 並不是原始 log 的完整鏡像，而是經過 rule 選擇後的結構化語意資料。

## 9. 一個完整解析範例

以下用 `aggregated_slot` 這條路徑說明 raw log 如何變成 event：

```text
time="2026-04-15T06:33:30Z" level="info" msg="latest aggregated slot: sub-001 group=group-test-001 ts=2026-04-15T06:33:30Z ulVol=1234 dlVol=5678 totalVol=6912 ulPkts=10 dlPkts=12 totalPkts=22 ulThr=12.3 dlThr=45.6 ulPktThr=1.2 dlPktThr=3.4" CAT="AnLF" NF="NWDAF"
```

先經過 `BASE` regex，得到的核心欄位大致是：

```json
{
  "time": "2026-04-15T06:33:30Z",
  "level": "info",
  "msg": "latest aggregated slot: sub-001 group=group-test-001 ts=2026-04-15T06:33:30Z ulVol=1234 dlVol=5678 totalVol=6912 ulPkts=10 dlPkts=12 totalPkts=22 ulThr=12.3 dlThr=45.6 ulPktThr=1.2 dlPktThr=3.4",
  "cat": "AnLF",
  "nf": "NWDAF",
  "extra": "",
  "extra_kv": {},
  "source": "nwdaf"
}
```

接著 parser 依 `ALL_RULES` 順序比對，最後命中 `rules/nwdaf.py` 的 `aggregated_slot` 規則，產生：

```json
{
  "type": "aggregated_slot",
  "time": "2026-04-15T06:33:30Z",
  "sub_id": "sub-001",
  "target": "group=group-test-001",
  "ts": "2026-04-15T06:33:30Z",
  "ul_vol": 1234.0,
  "dl_vol": 5678.0,
  "ul_thr": 12.3,
  "dl_thr": 45.6
}
```

這筆 event 之後才會再被 `main.py` 拿去更新 metrics、寫入 `events.jsonl`、推到 WebSocket。

## 10. 與 metrics 的關係

parser 本身不碰 Prometheus，但 `rules/` 模組同時也是 metric handler 的註冊來源。

這表示：

- 事件型別與欄位 schema 由 parser rule 決定
- 哪些事件會進一步寫入 metrics，則由同一組 rule 模組中的 `METRIC_HANDLERS` 決定

兩者共享同一批 event type 名稱，但執行時序不同：

1. parser 先產生 event
2. `main.py` 再用 event type 對應到 metric handlers

## 11. 目前限制

- 目前只支援單行、logrus 風格的文字 log；多行 stack trace 或 JSON log 不在範圍內
- parser 只保留 rule 真正需要的欄位，未被 `build()` 取用的內容會在 event 階段消失
- 目前沒有 parser-level 的錯誤事件；不匹配的行只會安靜地回傳 `None`
- rule 順序會直接影響語意；若未來增加更寬鬆的 regex，必須留意是否遮蔽既有規則
