# Metric Event Modeling Notes

本文件整理 DVR / replay 實作過程中發現的 metrics 建模問題，特別是 `retrain` annotation 的表示方式。這些議題影響 live 與 replay 兩條路徑，不應被視為 DVR 單一路線的後續工作，因此從 DVR 主線抽離並整理到 `plans/architecture/`。

## 範圍

主要討論：

- `retrain` 事件是否應該繼續以 counter 推導
- event metric、counter、gauge pulse 的語意差異
- live / replay / backfill / pseudo-live 是否應共享同一套事件建模

## 背景

目前 `retrain` annotation 仍沿用：

```promql
idelta(nwdaf_retrain_total{session="$session"}[15s]) > 0
```

這種做法的問題在於：

- 事件本質上是離散事件，但目前用 counter 的變化來推導
- 取樣密度會影響 annotation 是否能被偵測到
- replay / backfill / pseudo-live 為了讓 annotation 成立，需要額外照顧 sample pattern

因此它較像是「現有系統可運作的折衷表示法」，而不是語意最直接的建模。

## 與 DVR 的關係

下列內容仍屬 DVR 主線，留在 `plans/dvr/`：

- replay pseudo-live 如何把目前的 metric 寫回 Prometheus
- pre-seed 如何配合現有 dashboard / annotation

下列內容屬於更高一層的 metrics 建模議題，留在本文件：

- `retrain` 是否應該改為 event metric
- 若改動，如何同時適用於 live 與 replay
- 是否要保留 `nwdaf_retrain_total` 只做統計用途

## 目前已知事實

1. 現有 `counter + idelta()` 方案可以運作，但語意較曲折。
2. replay 較容易暴露這個問題，因為回填與 pseudo-live 都要顧及 sample pattern。
3. 問題不是 replay 專屬；只要 annotation 想要穩定表達某個離散事件，live 也會受同一套建模限制影響。

## 待評估方向

### 1. 維持現狀：counter + idelta / increase

- 變更最小
- 但繼續承受 sample density 與 query 語意帶來的複雜度

### 2. Gauge pulse

- 每次事件發生時短時間設為 1，再回到 0
- 比 counter 直覺，但仍需要 pulse 寬度策略

### 3. 獨立 event metric

- 將 `retrain` 明確表示為事件，而不是從 counter 變化推導
- 語意最直接，也較符合 debug 與 replay 的需求
- 需要單獨設計 label 與 cardinality 控制

## 目前傾向

若後續要整理 `retrain` annotation，較合理的方向是：

1. 保留 `nwdaf_retrain_total` 供統計與趨勢查詢使用。
2. 另增一條專用的 retrain event metric，供 Grafana annotation 使用。
3. live 與 replay 共用同一套事件建模，不再讓 replay 單獨承擔 sample pattern 補救邏輯。

## 結論

這是 DVR 實作過程中發現的延伸建模問題，但不阻塞當前 DVR 主流程完成。後續若要正式調整 retrain annotation，應以 metrics modeling 任務單獨規劃，而不是混在 DVR roadmap 裡處理。
