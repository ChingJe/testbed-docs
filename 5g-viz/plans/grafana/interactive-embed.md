# ⑥ Grafana 互動縮放 + ④ Legend Toggle — 實作規劃

對應 `5g-viz-improvements.md` 的項目 ④ 和 ⑥。
④ 依賴 ⑥，且 ⑥ 完成後 ④ 自動生效，因此合併規劃。

---

## 現況

嵌入方式：**public dashboard**（token-based）

```
events.js:  iframe.src = `${_grafanaBase}/public-dashboards/${_grafanaToken}`
```

- `grafana_setup.py` → `_get_or_create_public_token()` 建立 public dashboard token
- `main.py` → `GET /api/grafana-token` 回傳 `{base, token}` 給前端
- public dashboard 限制：**不能拖拉縮放時間**、**不能點 legend toggle 系列**

## 方案：切換到 anonymous viewer + kiosk mode

| 項目 | public dashboard (現在) | anonymous viewer (目標) |
|---|---|---|
| Time range 拖拉縮放 | ✗ | ✓ (drag-to-zoom on panel) |
| Legend click toggle | ✗ | ✓ |
| 需要 token | ✓ | ✗ (anonymous access) |
| grafana.ini 設定 | `allow_embedding` | `allow_embedding` + `[auth.anonymous]` |
| iframe URL | `/public-dashboards/{token}` | `/d/{uid}?orgId=1&kiosk&theme=dark&refresh=5s` |
| 安全性 | 僅持有 token 者可看 | 同網路皆可看 (Viewer role) |

> 安全性降低可接受——testbed 內網環境。

---

## 修改清單

### 1. `setup.sh` — 加入 anonymous auth 設定

目前已有 `[security] allow_embedding = true` 的寫入邏輯，新增相同模式：

```ini
[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

檢查步驟：
1. `grep` 檢查 `grafana.ini` 是否已有 `[auth.anonymous]` 且 `enabled = true`
2. 若無，append 到 `grafana.ini` 並 `systemctl restart grafana-server`
3. 無 sudo 時列印手動操作提示（同現有 `allow_embedding` 邏輯）

### 2. `frontend/events.js` — 改 iframe URL

```js
// Before
fetch('/api/grafana-token')
  .then(r => r.json())
  .then(cfg => { _grafanaBase = cfg.base; _grafanaToken = cfg.token; });

// ... ensureChart():
iframe.src = `${_grafanaBase}/public-dashboards/${_grafanaToken}`;

// After
fetch('/api/grafana-config')
  .then(r => r.json())
  .then(cfg => { _grafanaBase = cfg.base; _dashboardUid = cfg.uid; });

// ... ensureChart():
iframe.src = `${_grafanaBase}/d/${_dashboardUid}?orgId=1&kiosk&theme=dark&refresh=5s&from=now-3m&to=now`;
```

`kiosk` 模式隱藏 Grafana nav/sidebar，最大化 panel 空間。
使用者仍可在 panel 上 **drag-to-zoom**（選取時間範圍），`Ctrl+Z` 還原。

### 3. `main.py` — 重新命名並簡化 endpoint

```python
# Before
@app.get("/api/grafana-token")
async def grafana_token():
    return {"base": GRAFANA_BASE, "token": _grafana_token}

# After
@app.get("/api/grafana-config")
async def grafana_config():
    return {"base": GRAFANA_BASE, "uid": "nwdaf-traffic"}
```

- 移除全域變數 `_grafana_token`
- 移除 `_setup_grafana()` 中的 token 取得邏輯（setup 只需建 dashboard，不需 public token）

### 4. `grafana_setup.py` — 移除 public token 邏輯

刪除：
- `_get_or_create_public_token()` 函式（~20 行）
- `setup()` 回傳值改為 `None`（不再回傳 token）
- `GRAFANA_PUBLIC_TOKEN` import

保留：
- `_get_or_create_datasource_uid()` — 仍需要
- `_build_panel()` — 仍需要
- dashboard 建立邏輯 — 仍需要
- `_clear_prometheus_metrics()` — 仍需要

### 5. `config.py` + `.env.example` — 清理 token 設定

- `config.py`：移除 `GRAFANA_PUBLIC_TOKEN` 變數
- `.env.example`：移除 `GRAFANA_PUBLIC_TOKEN` 行

---

## ④ Legend Toggle

`grafana_setup.py` 的 `_build_panel()` 已有：

```python
"options": {"legend": {"displayMode": "list", "placement": "bottom"}}
```

切換到 anonymous viewer 後，legend toggle **自動生效**——點擊 legend 項目即可隱藏/顯示該系列。不需額外程式碼改動，只需驗證行為正確。

---

## 實作順序

```
1. setup.sh        — 加 anonymous auth 設定（需 sudo + restart grafana）
2. grafana_setup.py — 移除 public token 邏輯
3. config.py       — 移除 GRAFANA_PUBLIC_TOKEN
4. main.py         — 簡化 endpoint
5. events.js       — 改 iframe URL
6. .env.example    — 清理
7. 驗證            — ⑥ drag-to-zoom + ④ legend toggle
```

## 注意事項

1. **grafana.ini 寫入需要 sudo** — 同現有 `allow_embedding` 邏輯，無 sudo 時給手動操作提示
2. **anonymous auth 開放所有 dashboard** — testbed 環境可接受；如需限制可在 Grafana org 裡只放 nwdaf-traffic
3. **kiosk 模式無時間選擇器 UI** — drag-to-zoom、`Ctrl+Z` 還原可用；如需完整時間控制可改用不加 `kiosk` 參數（但 UI 較雜）
4. **舊 .env 的 `GRAFANA_PUBLIC_TOKEN`** — 程式碼移除後即使 .env 仍有此行也不會出錯（`os.getenv` 不存在只是不被使用）
