# Grafana Same-Origin Proxy Plan

本文件整理 `5g-viz` 在外部網路環境下的 Grafana 嵌入修正方案，並記錄最後實作結果。

核心想法不是要求外部瀏覽器直接打 `http://<host>:3000`，而是讓瀏覽器只連 `5g-viz` 已可達的入口 `http://<host>:8765`，再由 `5g-viz` backend 反向代理 Grafana。

> 狀態：已於 2026-05-08 實作並驗證可用。本文已從提案收斂為實作記錄。

---

## 背景

目前 `5g-viz` 前端的 Grafana iframe 直接使用：

```text
GRAFANA_BASE + /d/<uid>?...
```

也就是說，瀏覽器會自行連到 `GRAFANA_BASE` 指定的主機與 port。

在實驗室內部網路時，這通常可行；但若外部使用者只能進到 `5g-viz` 的 `8765`，而無法直連 `3000`，就會出現：

- `5g-viz` 主畫面可開
- topology / event log 正常
- Grafana iframe 空白或載入失敗
- 外部直接開 `http://<host>:3000/login` 得到 timeout

這代表問題不在 iframe 組 URL 的前端程式，而在於外部網路路徑根本到不了 Grafana。

---

## 已確認現況

本規劃不是純假設，而是基於目前主機與 Grafana 的實際觀察：

### 1. 主機上的 Grafana 服務正常運作

- `grafana-server.service` 為 `active (running)`
- 主機上 `:3000` 有 listen
- 主機本身具備 `140.113.110.77` 這個介面位址

這表示 Grafana 不是沒啟動，也不是只綁在 `localhost`。

### 2. 外部到 `:3000` 的路徑不通

外部瀏覽器直接打：

```text
http://140.113.110.77:3000/login
```

得到 `ERR_CONNECTION_TIMED_OUT`。

這代表問題不在 dashboard UID、iframe URL 字串、或前端 session 切換邏輯，而是在：

- 主機本地防火牆
- 校園/上游 ACL
- 或部署上只對外開放 `8765`、未對外開放 `3000`

### 3. Grafana 的嵌入必要設定已存在

已確認 `/etc/grafana/grafana.ini` 至少有：

```ini
[security]
allow_embedding = true

[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

因此目前 external timeout 不是因為：

- `X-Frame-Options` 阻擋 iframe
- Grafana 要求登入卻沒有 anonymous viewer

### 4. `server` 子路徑設定原本尚未配置

最初確認時，`[server]` 區段仍接近預設狀態，尚未看到：

```ini
root_url = ...
serve_from_sub_path = true
```

這一點不會造成外部 `:3000` timeout，但會直接影響本方案是否能穩定以 `/grafana/` 子路徑代理。

後續已補上：

```ini
[server]
root_url = http://140.113.110.77:8765/grafana/
serve_from_sub_path = true
```

且必須放在真正的 `[server]` 區段內；若誤放到 `[environment]` 或其他 section，Grafana 仍會吐出 `<base href="/">` 與 `/public/...` 根路徑資源，造成 iframe 半殘頁面。

---

## 問題定義

目前 `GRAFANA_BASE` 同時承擔兩種不同角色：

1. **瀏覽器入口**
前端 iframe 依靠它組出 Grafana dashboard URL。

2. **後端 API 入口**
`grafana_setup.py` 依靠它呼叫 Grafana admin API 建 datasource / dashboard。

這兩種角色在「外部 3000 不可達」的情況下會衝突：

- 瀏覽器需要一個外部可達位址
- backend 其實最穩的是打 `http://localhost:3000`

因此不能再只用一個 `GRAFANA_BASE` 同時解決兩件事。

---

## 方案摘要

採用 same-origin proxy 方案：

1. 瀏覽器端不再直接連 `:3000`
2. `5g-viz` 提供 `/grafana/...` 路徑，將請求代理到本機 Grafana
3. backend 管理 API 與 frontend iframe 入口分離

預期結果：

- 外部使用者只需要能連到 `http://<host>:8765`
- Grafana 區塊透過 `http://<host>:8765/grafana/...` 載入
- `grafana_setup.py` 仍可穩定透過本機 loopback 打 admin API

---

## 目標

- 解決外部無法直接連到 `:3000` 時的 Grafana iframe 失敗
- 保留目前 Grafana dashboard 與 DVR 的互動模式
- 避免為了外部存取去強迫打通校園網路或主機防火牆上的 `3000`
- 讓部署設定更明確區分「browser-facing URL」與「backend-facing URL」

## 非目標

- 不重新設計 dashboard panel
- 不改變 replay / pseudo-live / backfill 資料流
- 不處理 Grafana 帳號權限模型的大幅重構
- 不直接在此規劃中引入外部反向代理（如 nginx）

---

## 設定模型調整

### 新契約

新增並區分兩個設定：

| 變數 | 用途 | 典型值 |
|---|---|---|
| `GRAFANA_BASE` | 前端 iframe 使用的 base URL | `/grafana` |
| `GRAFANA_API_BASE` | backend 呼叫 Grafana API 使用的 base URL | `http://localhost:3000/grafana` |

### 調整原則

- `GRAFANA_BASE` 不再要求是外部直接可達的獨立 Grafana host:port
- `GRAFANA_BASE` 可以是相對路徑 `/grafana`
- `GRAFANA_API_BASE` 則固定代表 backend 真正要打的 Grafana 服務位置
- 當 Grafana 啟用 `/grafana/` 子路徑後，`GRAFANA_API_BASE` 也必須跟著包含 `/grafana`

---

## 實作結果

### 1. `config.py`

已新增：

- `GRAFANA_API_BASE = os.getenv("GRAFANA_API_BASE", "http://localhost:3000")`

保留：

- `GRAFANA_BASE`

說明：

- `GRAFANA_BASE` 給 `main.py` 的 `/api/grafana-config`
- `GRAFANA_API_BASE` 給 `grafana_setup.py` 與 Grafana proxy upstream
- runtime 建議值最終收斂為 `http://localhost:3000/grafana`

### 2. `grafana_setup.py`

已把所有目前使用 `GRAFANA_BASE` 發 API 的地方改成 `GRAFANA_API_BASE`。

重點：

- datasource / dashboard 建立都應直接打 backend 可見的 Grafana
- 不應經由 `/grafana/...` 這種給瀏覽器用的代理路徑自我繞回

### 3. `main.py`

已新增 Grafana proxy 路由群組：

- `GET/HEAD/POST/PUT/PATCH/DELETE/OPTIONS /grafana`
- `GET/HEAD/POST/PUT/PATCH/DELETE/OPTIONS /grafana/{path:path}`
- `WS /grafana/{path:path}`

目前已正確代理：

- dashboard HTML
- JS / CSS / font / image 資源
- panel 內發出的 Grafana API 請求
- Grafana live websocket 路徑

### 代理層最終行為

- 目標 upstream：`GRAFANA_API_BASE`
- 轉發 query string
- 保留必要的 content type
- 保留 status code
- 對 Grafana 回應 headers 做最小必要調整

至少要注意：

- `Location`
- `Set-Cookie`
- `Content-Type`
- `Cache-Control`
- WebSocket subprotocol / 雙向轉送

### 4. `/api/grafana-config`

仍回傳：

```json
{
  "base": "<GRAFANA_BASE>",
  "uid": "nwdaf-traffic"
}
```

但此時 `base` 預期會是：

```text
/grafana
```

而不是 `http://<host>:3000`

### 5. `frontend/events.js`

前端不需要知道 proxy 細節，只需繼續使用 `/api/grafana-config` 回來的 `base`。

因此預期修改很小：

- 若 `base` 為 `/grafana`，現有 `_grafanaUrl()` 應仍可正常工作

這代表本方案的改動應盡量集中在 backend 與 config，而非前端行為。

### 6. `.env.example` / `README.md` / docs

已更新文件與範例：

```env
GRAFANA_BASE=/grafana
GRAFANA_API_BASE=http://localhost:3000/grafana
```

並明確說明：

- 外部使用者走 `8765`
- backend 管理 API 走本機 `3000/grafana`

---

## Grafana 系統設定需求

若 Grafana 要掛在 `/grafana/` 子路徑下，除了 `5g-viz` backend 代理外，Grafana 自身也需要知道它被從子路徑提供。

最終需要在 `/etc/grafana/grafana.ini` 設定：

```ini
[server]
root_url = http://<host>:8765/grafana/
serve_from_sub_path = true
```

若外部入口未來改成 HTTPS，則 `root_url` 也必須改成對應的 `https://.../grafana/`。

### 實作時的關鍵教訓

已確認缺的不是 `allow_embedding` 或 anonymous auth，而是：

- `root_url`
- `serve_from_sub_path`

而且這兩個值必須位於真正的 `[server]` 區段內。

實作過程中出現過兩個關鍵錯誤：

1. `root_url` / `serve_from_sub_path` 一度被放到 `[environment]` 後方  
   結果 Grafana 仍輸出 `<base href="/">`，前端改抓 `/public/...`，導致 iframe 顯示
   “Grafana has failed to load its application files”。

2. `GRAFANA_API_BASE` 一度仍設成 `http://localhost:3000`  
   在 Grafana 啟用子路徑後，backend 先打 root path，再被 Grafana redirect 回 `8765/grafana/...`，最後在 proxy 內自我繞圈到 timeout。  
   正確值必須是 `http://localhost:3000/grafana`。

### 仍需保留的既有設定

```ini
[security]
allow_embedding = true

[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

理由：

- `allow_embedding` 仍是 iframe 必要條件
- anonymous viewer 仍是目前互動式嵌入的假設

---

## 為什麼不用直接打通 3000

直接讓外部瀏覽器打 `:3000` 的問題在於：

- 要依賴主機防火牆
- 要依賴校園網路/上游 ACL
- 可能還要處理 HTTPS / mixed content
- 對外暴露額外 port

相較之下，same-origin proxy 有幾個優點：

- 使用者只需要一個入口
- 與 `5g-viz` 本身對外開放策略一致
- 避免 iframe 指向另一個外部不可達 origin
- 後續若做 HTTPS，也只需處理 `8765` 這個入口

---

## 風險與注意事項

### 1. Grafana 子路徑支援

若 `root_url` / `serve_from_sub_path` 沒設好，常見症狀是：

- HTML 載到，但 JS/CSS 404
- iframe 顯示半殘頁面
- redirect 路徑跑回 `/login` 而非 `/grafana/login`

這個風險在本案中實際發生過，根因不是程式碼，而是 `grafana.ini` 的 section placement 錯誤。

### 2. Cookie / redirect 處理

若 Grafana 後續不再使用 anonymous auth，而需要登入 cookie：

- proxy 必須正確轉發 `Set-Cookie`
- path/domain/samesite 可能需要額外注意

目前 anonymous viewer 為主，這個風險較低，但代理層不應把相關 header 直接吃掉。

### 3. WebSocket / SSE

Grafana 實際會嘗試連 `/grafana/api/live/ws`。若未攔截這條 websocket，請求會落入 `StaticFiles`，觸發：

```text
assert scope["type"] == "http"
```

因此最終實作中已加入 `/grafana/{path}` 的 websocket proxy，而不是只處理 HTTP。

### 4. 相對路徑與 URL 組合

既有前端假設 `_grafanaBase` 可以直接拿來字串拼接；若使用 `/grafana`，理論上可行，但實作時仍需驗證：

- `base + /d/<uid>` 是否正確
- 不會出現重複 slash 或缺少 slash

---

## 驗證結果

### 功能驗證結果

已確認：

1. 內部網路可開 `http://<host>:8765`
2. 外部網路可開 `http://<host>:8765`
3. 兩邊都可看到 Grafana iframe
4. chart query、annotations、Prometheus datasource 請求皆回 `200`
5. Grafana live websocket 已不再造成 `StaticFiles` assertion

### 部署前置驗證結果

在開始改 code 前，已確認：

1. 外部目前已可穩定連到 `http://<host>:8765`
2. 外部目前無法直接連到 `http://<host>:3000`
3. 主機本地 Grafana service 正常
4. `allow_embedding = true`
5. anonymous viewer 已啟用

這五點成立，證明本案前提正確，適合直接走 same-origin proxy，而不是再花時間排查前端 iframe 邏輯。

### 路徑驗證結果

已驗證：

1. `http://127.0.0.1:3000/login` 會 `301` 到 `http://140.113.110.77:8765/grafana/login`
2. `http://<host>:8765/grafana/...` 會正確代理 dashboard 與 API 路徑
3. 在修正 `root_url` / `serve_from_sub_path` 與 `GRAFANA_API_BASE` 後，不再出現 `grafana proxy error: timed out`

### 回歸驗證結果

已確認：

1. `grafana_setup.py` 仍能建立 dashboard
2. `GET /api/grafana-config` 仍回傳 `{base, uid}`
3. 現有 `GRAFANA_GROUPS`、dashboard UID、DVR chart window 行為不變

---

## 最終設定摘要

### `.env`

```env
GRAFANA_BASE=/grafana
GRAFANA_API_BASE=http://localhost:3000/grafana
```

### `/etc/grafana/grafana.ini`

```ini
[server]
root_url = http://140.113.110.77:8765/grafana/
serve_from_sub_path = true
```

並保留：

```ini
[security]
allow_embedding = true

[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

---

## 決策結論

本案最終採用 **方案 B：Grafana same-origin proxy via `5g-viz`**，且已完成實作。

原因不是它最少改動，而是它最符合目前已知限制：

- 外部可達 `8765`
- 外部不可達 `3000`
- 現有 Grafana 服務仍在本機正常運作

因此最合理的工程方向，是讓 Grafana 借道 `8765` 提供給外部瀏覽器，而不是把 `3000` 的外部可達性當成前提。
