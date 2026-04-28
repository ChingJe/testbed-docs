# GPU 環境設置 — Retrain Replay & Daisy

本文說明如何為 `tools/retrain_replay` 及 `daisy/examples/07_MTLF_training` 建立支援 GPU 推論與訓練的本地環境。

## 環境概覽

| 元件 | 路徑 | Python | torch |
|------|------|--------|-------|
| retrain_replay | `NWDAF/NWDAF/tools/retrain_replay/` | 3.12 | 2.5.1+cu121 |
| daisy 07 | `daisy/daisy/examples/07_MTLF_training/` | 3.8 | 2.4.1+cu121 |

CUDA 需求：Driver ≥ 535、CUDA 12.x（cu121 wheel 向下相容至 12.2）。

---

## retrain_replay 環境

`pyproject.toml` 已設定 PyTorch CUDA index：

```toml
[[tool.uv.index]]
name = "pytorch-cu121"
url = "https://download.pytorch.org/whl/cu121"
explicit = true

[tool.uv.sources]
torch = [{ index = "pytorch-cu121" }]
```

建立或重建環境：

```bash
cd NWDAF/NWDAF/tools/retrain_replay
uv sync
```

`uv.lock` 已鎖定 `torch==2.5.1+cu121`，直接 `uv sync` 即可，不需額外指定 index。

---

## daisy 07 環境

daisy 仍需 Python 3.8（`daisyfl` 相依），且 `requirements.txt` 寫死 `torch~=2.0.0`（無 cu121 wheel），需分兩步安裝。

### 初次建立

```bash
cd daisy/daisy/examples/07_MTLF_training

# 1. 建 venv（指定 Python 3.8）
uv venv .venv --python /usr/bin/python3.8

# 2. 安裝 daisy 本體
cd ../../..   # 到 daisy/daisy/
uv pip install . --python examples/07_MTLF_training/.venv/bin/python

# 3. 安裝 example 其餘依賴（排除 torch，避免版本衝突）
cd examples/07_MTLF_training
grep -v "^torch" requirements.txt | uv pip install -r /dev/stdin

# 4. 安裝 CUDA torch（Python 3.8 最高支援 torch 2.4.x）
uv pip install \
  --extra-index-url https://download.pytorch.org/whl/cu121 \
  "torch>=2.1.0,<2.5.0" "torchvision>=0.16.0,<0.20.0"
```

### 驗證

```bash
NWDAF/NWDAF/tools/retrain_replay/.venv/bin/python \
  -c "import torch; print(torch.__version__, torch.cuda.is_available())"
# 期望輸出：2.5.1+cu121 True

daisy/daisy/examples/07_MTLF_training/.venv/bin/python \
  -c "import torch; print(torch.__version__, torch.cuda.is_available())"
# 期望輸出：2.4.1+cu121 True
```

---

## replay_config.yaml

`daisy.python_bin` 需指向 daisy venv（本地設定，不 commit）：

```yaml
daisy:
  python_bin: /home/chingje/testbed/5G_Infrastructure/daisy/daisy/examples/07_MTLF_training/.venv/bin/python
```

---

## 注意事項

- `torch~=2.0.0`（requirements.txt）無 CUDA 12.x wheel，直接 `pip install -r requirements.txt` 會裝 CPU 版。分步安裝是必要的。
- Python 3.8 的 torch 上限為 2.4.x（2.5 起僅支援 3.9+）。
- retrain_replay 的 `pyproject.toml` 和 `uv.lock` 已 commit CUDA 設定，重建時只需 `uv sync`。
- daisy venv 不在 git 追蹤範圍內，機器換環境後需重新執行上述步驟。
