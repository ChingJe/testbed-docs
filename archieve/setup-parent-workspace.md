# 工作區遷移指示

## 目標結構

```
~/testbed/                          ← 新的 parent 資料夾（已建立）
├── CLAUDE.md                   ← parent 層指示（需建立）
├── 5G_Infrastructure/          ← 從 ~/5G_Infrastructure/ 搬過來
└── 5g-viz/                     ← 新建 visualizer repo
```

---

## 步驟一：搬移 5G_Infrastructure

```bash
mv ~/5G_Infrastructure ~/testbed/5G_Infrastructure
```

### Vagrant 善後（重要）

**VM 不需要重建**。VM 存在 VirtualBox 裡，跟目錄路徑無關。

移動後只需要清掉 vagrant 的舊路徑索引：

```bash
vagrant global-status --prune
```

之後進各元件目錄正常 `vagrant up`，vagrant 會重新連上原本的 VM：

```bash
cd ~/testbed/5G_Infrastructure/5GC && vagrant up
# 以此類推
```

---

## 步驟二：建立 5g-viz repo

```bash
cd ~/testbed
mkdir 5g-viz
cd 5g-viz
git init
```

建立基本目錄結構（對應 .agent/visualizer-plan.md 的規劃）：

```bash
mkdir -p frontend
touch main.py collector.py parser.py state.py
touch frontend/index.html frontend/topology.js frontend/charts.js frontend/events.js
touch README.md
```

---

## 步驟三：建立 parent 層 CLAUDE.md

在 `~/testbed/CLAUDE.md` 建立：

```markdown
# testbed 工作區

## 子目錄說明

- `5G_Infrastructure/`：5G testbed，含 Vagrant VM、free5gc config、NWDAF 等，
  詳細說明見 `5G_Infrastructure/CLAUDE.md`。
- `5g-viz/`：Testbed 視覺化系統，讀取各 VM log，以 WebSocket 即時推送到前端。
  詳細規劃見 `5G_Infrastructure/.agent/visualizer-plan.md`。

## 對話語言

只使用繁體中文（加上必要的英文技術詞彙）。
```

---

## 步驟四：遷移 memory 檔案

舊 memory 路徑：
```
~/.claude/projects/-home-chingje-5G-Infrastructure/memory/
```

新 memory 路徑（從 ~/testbed/ 啟動 Claude Code 後會自動建立）：
```
~/.claude/projects/-home-chingje-testbed/memory/
```

搬移指令（先確認舊路徑存在）：

```bash
OLD=~/.claude/projects/-home-chingje-5G-Infrastructure/memory
NEW=~/.claude/projects/-home-chingje-testbed/memory

mkdir -p "$NEW"
cp "$OLD"/*.md "$NEW"/ 2>/dev/null || echo "no memory files to copy"
```

---

## 步驟五：確認

```bash
# 目錄結構正確
ls ~/testbed/

# git 狀態正常
cd ~/testbed/5G_Infrastructure && git status

# vagrant 記錄乾淨
vagrant global-status

# 5g-viz 初始化正確
ls ~/testbed/5g-viz/
```

---

## 背景資訊（給新 session 的 Claude）

視覺化系統的詳細規劃在：
```
~/testbed/5G_Infrastructure/.agent/visualizer-plan.md
```

Testbed 操作流程在：
```
~/testbed/5G_Infrastructure/.agent/workflow.md
```

### 視覺化系統重點摘要

- **後端**：Python FastAPI + asyncssh，tail 各 VM log，broadcast WebSocket events
- **前端**：Vanilla JS + Cytoscape.js（NF 拓樸）+ ECharts（曲線圖）
- **Log 來源**：
  - 5GC VM：`~/free5gc/log/<latest>/free5gc.log`（SMF、PFCP 等事件）
  - 5GC VM：`~/nwdaf.log`（UPF 流量、ML inference、accuracy monitor）
- **觀察重點**：SMF 訂閱轉發、UPF-EES 回報、NWDAF 推論、accuracy monitor
- **VM SSH**：透過 `vagrant ssh` 或直接 SSH 進各 VM（VirtualBox NAT，5GC VM IP: 192.168.125.5）

### Testbed VM 資訊

使用的 VM（其他不用）：
- `5GC`：free5gc core + NWDAF
- `UPF-EES`：UPF-EES（N4: 192.168.125.10）
- `UPF-EES2`：UPF-EES2（N4: 192.168.125.11）
- `gNB`：UERANSIM gNB + ue1~3
- `gNB2`：UERANSIM gNB2 + ue4~6

MongoDB 跑在 host port 27018（`mongosh mongodb://127.0.0.1:27018`）。
