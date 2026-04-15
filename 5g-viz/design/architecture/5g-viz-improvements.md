# 5g-viz Planned Improvements

老師 review 後整理的待改進項目，依難易度排序。

---

## 簡單

### ~~1. 深色背景統一~~ ✅ 已完成（91270d5）

**問題**：前端 topology 區與 Grafana 面板背景色不一致。Grafana 預設純黑（`#000`），topology 區目前用 `#1a1a2e`，整體視覺割裂。同時部分文字、節點邊框的對比度不夠，在投影或亮色螢幕下難以辨識。

**解決方向**：`index.html` 改 CSS 背景色，`topology.js` 對應調整節點邊框與文字顏色以維持足夠對比度。

### ~~2. 邊 label 命名不統一~~ ✅ 已完成（91270d5）

**問題**：topology 上各條邊的 label 風格不一致，混雜了口語描述（如 `store`、`provision`）和介面名稱（如 `EventSubscription`），看起來不專業，也讓讀者難以判斷哪個是標準介面、哪個只是自行命名。應統一使用一套命名風格。

**解決方向**：`topology.js` 的 `EDGE_STYLE` 常數和各 `flashEdge` 呼叫裡的 label 字串，統一命名風格（全用介面名稱，不混口語描述）。實作時追加了 `LABEL_SHORT` 縮寫 map（canvas 上顯示縮寫，hover tooltip 顯示完整介面名稱），繞過 Cytoscape.js 節點遮蓋 edge label 的限制。AccuracyReport 與 ModelProvision 兩條互反邊以 `arc-above` / `arc-above-rev` 往相反方向彎曲避免重疊。

### ~~3. 節點顯示名稱設定~~ ✅ 已完成（91270d5）

**問題**：節點 ID 與顯示名稱目前綁在一起（如 `UPF-EES` 對應 `upf_ees`），且 `UPF-EES` 與 `UPF-EES2` 命名不對稱——有 2 就應該有 1。若要修改顯示名稱目前需要直接改程式碼，沒有獨立的設定層。

**解決方向**：`topology.js` 加一個 `DISPLAY_NAME` map，把 node id 和顯示文字分離，UPF-EES 改成 UPF-EES1 也在此一併處理。

### ~~4. Grafana 曲線 filter（UL / DL toggle）~~ ✅ 已完成（daedde3）

**問題**：每張 Grafana 圖有四條線（ground truth UL/DL + predicted UL/DL），線條密集時互相遮蓋，無法單獨檢視某一條。目前沒辦法 toggle 隱藏特定線。

**解決方向**：確認 `grafana_setup.py` 的 `_build_panel()` legend 設定正確，讓使用者可以點 legend 項目隱藏個別曲線。實際互動性取決於 Grafana 的嵌入模式（見依賴關係）。

---

## 中等

### ~~5. Topology 節點／邊 filter~~ ✅ 已完成（bc44093）

**問題**：topology 上同時顯示所有 NF 之間的訊號流動，debug 時只關心特定 NF 卻無法隱藏其他節點或邊，視覺很雜。目前也沒有辦法只看某一條 SBI call 類型的邊。

**解決方向**：`index.html` 加 filter 側欄（checkbox 列表），`topology.js` 用 cytoscape 的 `show()`/`hide()` 控制節點與邊的可見性，節點和邊各自獨立 toggle。NWDAF 為 compound parent，隱藏時 AnLF / MTLF 子節點跟著隱藏，checkbox 變灰不可操作。

### ~~6. Grafana 互動縮放（時間範圍選取）~~ ✅ 已完成（daedde3）

**問題**：Grafana 目前以 public dashboard token 嵌入 iframe，無法互動（不能拖拉選時間範圍、不能 zoom in 特定區段）。當曲線上有異常或模型切換點時，使用者無法就地放大細看。

**解決方向**：改用 Grafana anonymous viewer + kiosk mode，iframe 改指向 `/d/{uid}?orgId=1&kiosk&theme=dark&refresh=5s`，讓使用者可以在 iframe 內直接拖拉時間範圍。`setup.sh` 自動設定 `[auth.anonymous]`。header 加入 ↺ Live 按鈕（含分鐘數輸入）供縮放後一鍵回到 live 視窗。

---

## 困難

### ~~7. 動態 topology 設定~~ ✅ 已完成（bc44093）

**問題**：目前所有節點（ID、位置、label）和邊的樣式都硬編碼在 `topology.js`，若要用在不同的 5G 部署配置或其他研究場景，必須修改程式碼。缺乏讓使用者自行定義 topology 佈局的機制，限制了工具的通用性。

**解決方向**：設計 YAML config schema（`topology.yaml`：nodes / nf_aliases / edge_styles / layout），後端加 `GET /api/topology-config` endpoint，`topology.js` 啟動時 fetch 並動態渲染。

### 8. SSH log sources config-driven

**問題**：目前要連哪台 VM、tail 哪個 log file、source tag 叫什麼，都硬編碼在 `collector.py` 和 `config.py` 裡。同一台 VM 新增 log file 要改 `collector.py`；新增 VM 要同時改 `collector.py` 和 `config.py`。與 topology、event reactions 已經 config-driven 的部分不一致。

**解決方向**：`topology.yaml` 加 `ssh_sources` section，定義每台 VM 的連線設定（env var 名稱）和要 tail 的 log 清單（path + source tag）。`collector.py` 改成動態讀取 sources list，單一泛用函式取代各 VM 專屬函式。新增 VM 或新增 log file 只需改 `topology.yaml` 和 `.env`，不動程式碼。

### 9. DVR 模式（暫停 / 回放 / 跳回 live）

**問題**：目前只支援 live tail——畫面永遠是最新狀態，錯過的事件無法回頭看。實驗過程中想回頭確認某個時間點的 topology 狀態（例如某次 retrain 觸發當下信號流程）是辦不到的。理想的行為是像 DVR：可以在任意時間點暫停、向前/向後 scrub，也可以一鍵跳回 live。

**解決方向**：後端在 `main.py` 維護一個帶 timestamp 的 event ring buffer，加 `GET /api/events?from=&to=` API。前端加時間軸控制列（暫停、scrub、go live），scrub 時把 topology 狀態從 t=0 重播到指定時間點重建。

---

## 依賴關係

```
④ legend toggle ────── depends on ────── ⑥ Grafana 互動縮放
    (public dashboard 模式下 legend 無互動，④ 做了也沒用)

② 邊 label ──┐
③ 顯示名稱 ──┼── 可獨立先做，之後會被 ⑦ 動態 config 吸收
⑤ filter   ──┘        (filter 的選項列表可從 config 推導)

⑧ DVR ── 完全獨立，不依賴其他項目
① 背景 ── 完全獨立
```

## 建議執行順序

1. ~~**①③②** — 純改字串／CSS，快速收割~~ ✅ 已完成
2. ~~**⑥** — 解鎖 Grafana 互動，順帶讓 **④** 生效~~ ✅ 已完成
3. ~~**⑤⑦** — filter UI + config 系統~~ ✅ 已完成
5. **⑧** — SSH sources config-driven，完善 config-driven 覆蓋範圍
6. **⑨** — DVR，最後獨立開發
