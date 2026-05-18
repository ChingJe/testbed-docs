# Rendering

本文描述目前 Grafana dashboard 如何查 Prometheus、切換 session，並在 live / replay 下呈現不同圖表語意。

## 1. Dashboard 結構

dashboard 由 `services/grafana_setup.py` 動態生成，主要包含：

- 每個 `grafana.groups` 對應一張 traffic panel
- degradation / chronic / deviation 相關 panel
- retrain / policy annotations

panel 數量與排列不是手動維護 JSON，而是由 Python 根據 profile 設定生成。

## 2. Traffic panel 查哪些 series

每張 traffic panel 都查四條核心 PromQL：

- `nwdaf_ground_truth_ul_bytes`
- `nwdaf_ground_truth_dl_bytes`
- `nwdaf_predicted_ul_bytes`
- `nwdaf_predicted_dl_bytes`

查詢條件綁兩個維度：

- `session="$session"`
- `target="group=<group_id>"`

prediction query 仍固定使用：

```promql
offset 5s
```

它的目的不是修改寫入 timestamp，而是讓 prediction 線在圖上和對應 slot 的 ground truth 做視覺對齊。

## 3. `session` variable 的角色

dashboard 內建一個 template variable `session`，查詢來源是：

```promql
query_result(count by (session) (count_over_time(nwdaf_session_anchor[365d])))
```

這代表 Grafana 可切換的 session 候選集合，取決於 retention 視窗內仍存在 `nwdaf_session_anchor` 樣本的 session。

前端 iframe 透過：

```text
var-session=<session_id>
```

指定要看的 session。

## 4. Annotations

目前 retrain annotations 仍由下列事件 pulse metric 推導：

- `nwdaf_retrain_start_event`
- `nwdaf_retrain_done_event`

透過 `unless ... offset <step>` 保留連續高電位中的第一個 step，避免 annotation 重複。

## 5. 三種主要圖表語意

### Live

live 下 Grafana 通常查的是：

- 原始 live session
- `from=now-<window>m`
- `to=now`
- `refresh=<cfg.grafana.refresh>`

這時圖表對應的是目前 exporter / scrape 狀態。

### Replay paused / scrubbed

replay 停住或拖曳時，Grafana 查的是：

- 原始 replay session
- 絕對 `from/to`
- `refresh=off`

這時圖表最接近「錄製當時的固定歷史圖」。

### Replay playing

replay 播放時，Grafana 仍查：

- 原始 replay session

但查詢時間窗改成：

- historical relative range
- 例如 `from=now-3887m&to=now-3884m`
- `refresh=<cfg.grafana.refresh>`

因此目前 replay `playing` 與 `paused` 的差異不在 session 是否切換，而在查詢時間語意：

- `paused`：固定歷史窗
- `playing`：會隨 `now` 平滑滑動的歷史相對窗

## 6. Replay backfill 的角色

replay 啟動時，backend 會視 `backfill_policy` 決定是否把 metric event 以原始 timestamp remote write 回 Prometheus。

常見情況：

- `auto`：若 Prometheus 已有該 session，直接 reuse；否則 backfill
- `overwrite`：先刪該 session，再 backfill
- `skip`：完全不 backfill；Grafana 對該 session 會是空圖

這也是為什麼 replay `playing` 雖然不再需要 pseudo-live remap，仍然需要 replay backfill 作為原始 session metrics 的來源。

## 7. Chart Window 的作用

`Chart Window` 控制的是 iframe query 的顯示寬度。

在不同模式下：

- live：改變 `now-window ~ now`
- replay paused：改變絕對歷史 trailing window
- replay playing：改變 historical relative range 的寬度

因此播放中改動 `Chart Window`，仍然會造成 Grafana iframe 明顯重新同步，但不再需要另一條 pseudo-live metrics stream。

## 8. Deviation 與 `model_swap`

live 與 replay 在 `model_swap` 的底層機制仍不完全相同：

- live：用 exporter-side series termination
- replay backfill：以 historical remote-write 語意重建等價結果

實務上圖表效果已盡量對齊，但仍不保證每個細節與 live runtime 完全同構。

## 9. 目前限制

- dashboard 變數索引仍依賴 `nwdaf_session_anchor`
- prediction `offset 5s` 是固定常數
- query 與 panel 結構由 Python 生成，不是手工維護 dashboard JSON
