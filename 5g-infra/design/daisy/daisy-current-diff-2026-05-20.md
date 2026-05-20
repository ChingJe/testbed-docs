# Daisy Current Diff Snapshot (2026-05-20)

## 目的

記錄 `5G_Infrastructure/` 內 `daisy` 目前工作樹的差異快照，包含：

- workspace 主 repo 對 `daisy/` 的 diff
- `daisy/daisy` submodule 內部的 dirty working tree
- 針對目前變更的摘要解讀

本文件是以 2026-05-20 當下工作區狀態為準的原始記錄；若後續 working tree 再變動，這份內容不會自動更新。

---

## 總結

目前 `5G_Infrastructure` 的 `daisy` 相關差異可分成兩層：

### 1. 主 repo 層

- `daisy/daisy` submodule 指標從 `7992baa` 移到 `8fccdaa-dirty`
- `daisy/nodes.yaml` 內的 master / API / clients 位址由 `192.168.107.5` 改成 `192.168.127.5`
- 新增未追蹤檔案 `daisy/daisyconfig.json`，內容為 `mongodb://10.0.2.2:27018/`

### 2. submodule 內部 dirty 變更

`daisy/daisy` 內目前有 7 個已修改檔案與一批未追蹤 artifact：

- `examples/07_MTLF_training/client.py`
- `examples/07_MTLF_training/custom_strategy.py`
- `examples/07_MTLF_training/daisyconfig.json`
- `examples/07_MTLF_training/dataset.py`
- `examples/07_MTLF_training/master.py`
- `src/py/daisyfl/common/task_launcher.py`
- `src/py/daisyfl/master/server_api_handler.py`
- `examples/07_MTLF_training/artifacts/*.tar.gz`

其中主要變更方向是：

- 導入 federated scaler aggregation 的 stats round / shared scaler flow
- 支援 fixed scaler 與 seed model / seed scaler
- 將 early stopping / LR patience / local epochs 改成 task config 可調
- 在 server publish path 補上 task-scoped model / scaler 準備流程
- 調整本地 MongoDB URI 與 example import path

### 3. 行為層面的重點

- `client.py` 在 round 1 可只回傳 local scaler statistics，不做正式訓練
- `custom_strategy.py` 會聚合各 client 的 `mean / var / n`，產生全域 `scaler.pkl`
- `dataset.py` 支援載入 task-scoped shared scaler；若要求 shared scaler 卻不存在，會直接報錯
- `server_api_handler.py` 支援 `USE_FIXED_SCALER`、`SEED_MODEL_PATH`、`SEED_SCALER_PATH`
- `task_launcher.py` 會把 `task_config` 傳進 strategy constructor

---

## Raw Record

以下保留當下查得的原始輸出。

### A. `5G_Infrastructure` 主 repo：`daisy` status

```text
 M daisy/daisy
 M daisy/nodes.yaml
?? daisy/daisyconfig.json
```

### B. `5G_Infrastructure` 主 repo：`daisy` diff stat

```text
 daisy/daisy      | 2 +-
 daisy/nodes.yaml | 8 ++++----
 2 files changed, 5 insertions(+), 5 deletions(-)
```

### C. `5G_Infrastructure` 主 repo：`daisy` diff file list

```text
daisy/daisy
daisy/nodes.yaml
```

### D. `5G_Infrastructure` 主 repo：submodule log style diff

```text
Submodule daisy/daisy contains untracked content
Submodule daisy/daisy contains modified content
Submodule daisy/daisy 7992baa..8fccdaa:
  > feat: implement graceful early stopping in MTLF training.
  > feat: implement automated client lifecycle management
  > feat: following up updates for dynamic model deploying
  > deature : Dynamic model deployment when accepting tasks
  > correction: upgrade the config logic so it's cleaner
  > feat: Centralizing configuration in `daisyconfig.json`.
  > feat: Dynamic client deployment, and configurable MongoDB URI.
  > update requirements.txt to support mongodb
  > feat: Implement MongoDB data loading and enhanced feature aggregation for MTLF training
  > feat: Add named training session support for model and scaler saving.
  > change for testing packing integrity with NWDAF testing
  > feat: Allow `MODEL_META` to be specified in task configurations with default values and update example network addresses.
  > feat: Implement asynchronous task execution in the master API with callback support and model artifact packaging/download.
  > update readme.md to make it cleaner
  > feat: Refactor data loading to prevent scaler saving during evaluation, improve graceful shutdown handling, and adjust model output classes.
```

### E. `5G_Infrastructure` 主 repo：`daisy/daisy` pointer diff

```diff
diff --git a/daisy/daisy b/daisy/daisy
index 7992baa..8fccdaa 160000
--- a/daisy/daisy
+++ b/daisy/daisy
@@ -1 +1 @@
-Subproject commit 7992baab3d0855b3412944ef90d5cb429eff5dad
+Subproject commit 8fccdaaf3de20b429727ace8cf2b5a5093a4f9c3-dirty
```

### F. `5G_Infrastructure` 主 repo：`daisy/nodes.yaml` diff

```diff
diff --git a/daisy/nodes.yaml b/daisy/nodes.yaml
index 9b2b2a1..abe0c9a 100644
--- a/daisy/nodes.yaml
+++ b/daisy/nodes.yaml
@@ -1,14 +1,14 @@
 # two-layer
 master:
-    address: "192.168.107.5:8887"
-    api_ip: "192.168.107.5"
+    address: "192.168.127.5:8887"
+    api_ip: "192.168.127.5"
     api_port: 9887
     type: "master"
     children:
         - client0:
-            address: "192.168.107.5:10087"
+            address: "192.168.127.5:10087"
             type: "client"
         - client1:
-            address: "192.168.107.5:10088"
+            address: "192.168.127.5:10088"
             type: "client"
```

### G. `5G_Infrastructure` 主 repo：未追蹤 `daisyconfig.json` 內容

```json
{
    "MONGO_URI": "mongodb://10.0.2.2:27018/"
}
```

### H. `daisy/daisy` submodule：status

```text
 M examples/07_MTLF_training/client.py
 M examples/07_MTLF_training/custom_strategy.py
 M examples/07_MTLF_training/daisyconfig.json
 M examples/07_MTLF_training/dataset.py
 M examples/07_MTLF_training/master.py
 M src/py/daisyfl/common/task_launcher.py
 M src/py/daisyfl/master/server_api_handler.py
?? examples/07_MTLF_training/artifacts/
```

### I. `daisy/daisy` submodule：diff stat

```text
 examples/07_MTLF_training/client.py          |  67 +++++++++++-
 examples/07_MTLF_training/custom_strategy.py | 110 +++++++++++++++++---
 examples/07_MTLF_training/daisyconfig.json   |   4 +-
 examples/07_MTLF_training/dataset.py         | 147 ++++++++++++++++++++++-----
 examples/07_MTLF_training/master.py          |  11 +-
 src/py/daisyfl/common/task_launcher.py       |   2 +-
 src/py/daisyfl/master/server_api_handler.py  |  56 ++++++++--
 7 files changed, 346 insertions(+), 51 deletions(-)
```

### J. `daisy/daisy` submodule：HEAD 與最近 commit

```text
8fccdaaf (HEAD -> NWDAF-daisy, origin/NWDAF-daisy) feat: implement graceful early stopping in MTLF training.
aff30a24 feat: implement automated client lifecycle management
d96fcb70 feat: following up updates for dynamic model deploying
9acdacb0 deature : Dynamic model deployment when accepting tasks
fd32ec5d correction: upgrade the config logic so it's cleaner
```

### K. `daisy/daisy` submodule：完整 dirty status（含 untracked artifacts）

```text
 M examples/07_MTLF_training/client.py
 M examples/07_MTLF_training/custom_strategy.py
 M examples/07_MTLF_training/daisyconfig.json
 M examples/07_MTLF_training/dataset.py
 M examples/07_MTLF_training/master.py
 M src/py/daisyfl/common/task_launcher.py
 M src/py/daisyfl/master/server_api_handler.py
?? examples/07_MTLF_training/artifacts/0263b376c65c4b0ba5668e5ba31a41ad.tar.gz
?? examples/07_MTLF_training/artifacts/091f6b8887614bf48edfdaa96bf767af.tar.gz
?? examples/07_MTLF_training/artifacts/0a5cf4d88e2c49d788fcfe7985f8af82.tar.gz
?? examples/07_MTLF_training/artifacts/0e994923e55f44a58f70bde291aaa7ab.tar.gz
?? examples/07_MTLF_training/artifacts/10d7cff34e384713bb369ee543f3b827.tar.gz
?? examples/07_MTLF_training/artifacts/124e618d066c4a6fa69d8aa3e1989caf.tar.gz
?? examples/07_MTLF_training/artifacts/1e45f67bfc7c48928cbdb0065c1bc538.tar.gz
?? examples/07_MTLF_training/artifacts/214c39c5efd1453d82a9d82e49390648.tar.gz
?? examples/07_MTLF_training/artifacts/2a5765d63fb54442b714242d2f18ae33.tar.gz
?? examples/07_MTLF_training/artifacts/2f7e5d36012440c381d050abcc742058.tar.gz
?? examples/07_MTLF_training/artifacts/31558f89116d44dcaa3f1ae0645b09b8.tar.gz
?? examples/07_MTLF_training/artifacts/315ed115e6fb4e50ae45dab9c1db474d.tar.gz
?? examples/07_MTLF_training/artifacts/353c65127ad14c1c95439fb37c0728f3.tar.gz
?? examples/07_MTLF_training/artifacts/3c3662c82d8844138918380b00be3402.tar.gz
?? examples/07_MTLF_training/artifacts/3de16c4c9d8448cbbe5eb5548293f4bb.tar.gz
?? examples/07_MTLF_training/artifacts/454d5f4aa25348dab80c9a96b9ec547e.tar.gz
?? examples/07_MTLF_training/artifacts/49dcd64968894e26a55a623e0f191c29.tar.gz
?? examples/07_MTLF_training/artifacts/4d4192fd42cd405ab13b048bc6862e74.tar.gz
?? examples/07_MTLF_training/artifacts/521dd9f470244146a7582ca2609ee3d8.tar.gz
?? examples/07_MTLF_training/artifacts/56723775d6404bdcb29fb0f6f3dc4044.tar.gz
?? examples/07_MTLF_training/artifacts/58407d22934d443d9b1dfec5680e5ae9.tar.gz
?? examples/07_MTLF_training/artifacts/6034a955886d461888187d86c0986e6e.tar.gz
?? examples/07_MTLF_training/artifacts/62b25a56288a46d2886e2673edefa01c.tar.gz
?? examples/07_MTLF_training/artifacts/63386cc80d094587a76314fe6b4d0baa.tar.gz
?? examples/07_MTLF_training/artifacts/69c07975583c4a049a4de70b7d9b463c.tar.gz
?? examples/07_MTLF_training/artifacts/6c98de7e9f8f4d2895ee0ed5b313686d.tar.gz
?? examples/07_MTLF_training/artifacts/6e78abe644a543fc868c4b5c7c5b817e.tar.gz
?? examples/07_MTLF_training/artifacts/6f3c54e1bf4441b49fcfe4313cf8e3dd.tar.gz
?? examples/07_MTLF_training/artifacts/768b239142d94bc494c3e1c21361beae.tar.gz
?? examples/07_MTLF_training/artifacts/79873c1e2c5c4dc3a7db710b544b6669.tar.gz
?? examples/07_MTLF_training/artifacts/7c91881f56084ad5997cff54f9de9c8c.tar.gz
?? examples/07_MTLF_training/artifacts/7fe0410b418d438aaeafffefe792455a.tar.gz
?? examples/07_MTLF_training/artifacts/8b9054c4e6274a16a4060fe71aaf8c25.tar.gz
?? examples/07_MTLF_training/artifacts/916c7f6a450549cc864eaa4d1977f5d6.tar.gz
?? examples/07_MTLF_training/artifacts/94012b1ae62d48e2ac7abcf5d4645ad4.tar.gz
?? examples/07_MTLF_training/artifacts/96797b03b71c4f96804a77c8e6b82065.tar.gz
?? examples/07_MTLF_training/artifacts/975f0d44d7aa4bf1a39db5fb3a71a25a.tar.gz
?? examples/07_MTLF_training/artifacts/9f10e30db8c84221a4ed0b9418e59d65.tar.gz
?? examples/07_MTLF_training/artifacts/9f2a0aba9c54414591884e4b7054bc8e.tar.gz
?? examples/07_MTLF_training/artifacts/a276a7eb565149798b70df45e4a6e443.tar.gz
?? examples/07_MTLF_training/artifacts/b026fc97113049679ecfde3c7c5e48aa.tar.gz
?? examples/07_MTLF_training/artifacts/bcaa0e2b915d4290881c7ed44de3d6f3.tar.gz
?? examples/07_MTLF_training/artifacts/c1ed1565b1d34ee1bd6e2df97354d901.tar.gz
?? examples/07_MTLF_training/artifacts/ca486f8f89cc43c495f107c513e6ed89.tar.gz
?? examples/07_MTLF_training/artifacts/d18f6f823ac44f90837d6378f6719107.tar.gz
?? examples/07_MTLF_training/artifacts/e88879d4464c4da0bef7877fabfbb9d6.tar.gz
?? examples/07_MTLF_training/artifacts/f1d70befda854a0aa6623a5bf95c1559.tar.gz
?? examples/07_MTLF_training/artifacts/f5eadf56816d4d9bad09ebf85e3ac53d.tar.gz
```

### L. `client.py` diff

```diff
diff --git a/examples/07_MTLF_training/client.py b/examples/07_MTLF_training/client.py
index 4aa83ea1..f8428ce4 100644
--- a/examples/07_MTLF_training/client.py
+++ b/examples/07_MTLF_training/client.py
@@ -1,3 +1,11 @@
+import os
+import sys
+from pathlib import Path
+
+DAISY_SRC_PY = Path(__file__).resolve().parents[2] / "src" / "py"
+if str(DAISY_SRC_PY) not in sys.path:
+    sys.path.insert(0, str(DAISY_SRC_PY))
+
 import warnings
 from collections import OrderedDict
 
@@ -13,6 +21,7 @@ from daisyfl.common import (
     CID,
     ACCURACY,
     DATA_SAMPLES,
+    CURRENT_ROUND,
 )
 import argparse
 import uuid
@@ -25,6 +34,7 @@ parser.add_argument('--data_dirs', nargs='+', default=['ees_training_data'], hel
 parser.add_argument('--tid', type=str, default=None, help="TID to fetch data from MongoDB")
 parser.add_argument('--group_id', type=str, default=None, help="Group ID for the isolated dataset")
 parser.add_argument('--model_meta', type=str, default=None, help="JSON string of MODEL_META from task config")
+parser.add_argument('--use_fixed_scaler', action='store_true', help="Require a pre-seeded shared scaler and skip scaler aggregation flow")
 args = parser.parse_args()
 
 # Parse MODEL_META with defaults
@@ -33,14 +43,21 @@ _model_meta = _json.loads(args.model_meta) if args.model_meta else {}
 _model_cfg = _model_meta.get("model", {})
 _infer_cfg = _model_meta.get("inference", {})
 
+_train_cfg = _model_meta.get("training", {})
 _input_dim = _model_cfg.get("input_size", 10)
 _num_channels = _model_cfg.get("num_channels", [32, 64, 64, 64])
 _seq_length = _infer_cfg.get("seq_length", 30)
 _out_seq_len = _infer_cfg.get("out_seq_len", 5)
+_local_epochs = int(_train_cfg.get("local_epochs", 3))
 
 warnings.filterwarnings("ignore", category=UserWarning)
 device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")
 
+print(
+    f"[Client] startup group={args.group_id} tid={args.tid} "
+    f"use_fixed_scaler={args.use_fixed_scaler} seq_length={_seq_length} out_seq_len={_out_seq_len}"
+)
+
 def train(net, trainloader, epochs, lr=0.001):
     """Train the model on the training set using Huber Loss."""
     criterion = torch.nn.HuberLoss()
@@ -79,10 +96,24 @@ def test(net, testloader):
 # Use MODEL_META values for dataset and model construction
 trainloader, testloader, output_dim = get_dataloaders(
     data_dirs=args.data_dirs, tid=args.tid, group_id=args.group_id,
-    batch_size=32, seq_length=_seq_length, out_seq_len=_out_seq_len)
+    batch_size=32, seq_length=_seq_length, out_seq_len=_out_seq_len,
+    save_scaler=False, require_shared_scaler=args.use_fixed_scaler)
 
 net = get_model(input_dim=_input_dim, num_classes=output_dim, num_channels=_num_channels).to(device)
 
+def reload_dataloaders() -> None:
+    global trainloader, testloader
+    trainloader, testloader, _ = get_dataloaders(
+        data_dirs=args.data_dirs,
+        tid=args.tid,
+        group_id=args.group_id,
+        batch_size=32,
+        seq_length=_seq_length,
+        out_seq_len=_out_seq_len,
+        save_scaler=False,
+        require_shared_scaler=True,
+    )
+
 class DaisyTrainer(fl.client.NumPyTrainer):
     def get_parameters(self, config):
         return [val.cpu().numpy() for _, val in net.state_dict().items()]
@@ -94,13 +125,45 @@ class DaisyTrainer(fl.client.NumPyTrainer):
 
     def fit(self, parameters, config):
         self.set_parameters(parameters)
+
+        if config.get("is_stats_round"):
+            from dataset import get_local_stats
+
+            print(f"\n[Client] Stats round: calculating local scaler statistics for group={args.group_id}")
+            stats = get_local_stats(
+                data_dirs=args.data_dirs,
+                tid=args.tid,
+                group_id=args.group_id,
+                seq_length=_seq_length,
+                out_seq_len=_out_seq_len,
+            )
+            config[METRICS] = {"scaler_stats": stats}
+            return self.get_parameters(config={}), config
+
+        if args.use_fixed_scaler and config.get(CURRENT_ROUND, 0) == 1:
+            print(f"[Client] Fixed scaler mode: training starts immediately with pre-seeded shared scaler for group={args.group_id}")
+
+        if args.tid and (not args.use_fixed_scaler) and config.get(CURRENT_ROUND, 0) == 2:
+            print(f"[Client] Round 2: reloading dataloaders with aggregated global scaler for group={args.group_id}")
+            reload_dataloaders()
+
         lr = config.get("lr", 0.001)
-        train(net, trainloader, epochs=3, lr=lr)
+        train(net, trainloader, epochs=_local_epochs, lr=lr)
         config[METRICS] = { DATA_SAMPLES: len(trainloader.dataset) }
         return self.get_parameters(config={}), config
 
     def evaluate(self, parameters, config):
         self.set_parameters(parameters)
+
+        if config.get("is_stats_round"):
+            config[METRICS] = {
+                "MAE": 0.0,
+                ACCURACY: 0.0,
+                LOSS: 0.0,
+                DATA_SAMPLES: len(testloader.dataset),
+            }
+            return config
+
         loss, mae = test(net, testloader)
         config[METRICS] = {
             "MAE": mae,
```

### M. `custom_strategy.py` diff

```diff
diff --git a/examples/07_MTLF_training/custom_strategy.py b/examples/07_MTLF_training/custom_strategy.py
index 3cfb9694..2263477e 100644
--- a/examples/07_MTLF_training/custom_strategy.py
+++ b/examples/07_MTLF_training/custom_strategy.py
@@ -1,27 +1,103 @@
-import sys
+import os
+
+import joblib
+import numpy as np
+from sklearn.preprocessing import StandardScaler
 from daisyfl.strategy.fedavg import FedAvg
-from daisyfl.common import LOSS
+from daisyfl.common import LOSS, METRICS, TID, DATA_SAMPLES, CURRENT_ROUND
 
 class EarlyStoppingFedAvg(FedAvg):
-    def __init__(self, *args, **kwargs):
+    def __init__(self, *args, task_config=None, **kwargs):
+        cfg = task_config or {}
+        kwargs.setdefault("num_clients_fit", int(cfg.get("NUM_CLIENTS_FIT", 2)))
+        kwargs.setdefault("num_clients_evaluate", int(cfg.get("NUM_CLIENTS_EVALUATE", 2)))
         super().__init__(*args, **kwargs)
+        self.max_es_patience = int(cfg.get("ES_PATIENCE", 5))
+        self.max_lr_patience = int(cfg.get("LR_PATIENCE", 3))
         self.best_loss = float('inf')
         self.es_patience = 0
         self.lr_patience = 0
-        self.current_lr = 1e-3
-        
+        self.current_lr = float(cfg.get("INITIAL_LR", 1e-3))
+        self.use_fixed_scaler = bool(cfg.get("USE_FIXED_SCALER", False))
+
     def configure_fit(self, parameters, config, **kwargs):
         config["lr"] = self.current_lr
+        current_round = config.get(CURRENT_ROUND, 0)
+        if (not self.use_fixed_scaler) and current_round == 1:
+            print("\n[NWDAF CustomStrategy] Round 1 configured as stats round (federated scaler aggregation enabled)")
+            config["is_stats_round"] = True
+        elif self.use_fixed_scaler and current_round == 1:
+            print("\n[NWDAF CustomStrategy] Fixed scaler mode enabled: skipping stats round and training with pre-seeded scaler")
         return super().configure_fit(parameters, config, **kwargs)
 
+    def aggregate_fit(self, results, **kwargs):
+        is_stats_round = False
+        if results:
+            _, fit_res = results[0]
+            is_stats_round = bool(fit_res.config.get("is_stats_round"))
+
+        if not is_stats_round:
+            return super().aggregate_fit(results, **kwargs)
+
+        stats_list = []
+        for _, fit_res in results:
+            metrics = fit_res.config.get(METRICS, {})
+            stats = metrics.get("scaler_stats")
+            if stats is not None:
+                stats_list.append(stats)
+
+        if not stats_list:
+            raise RuntimeError("Stats round completed without receiving scaler statistics from clients")
+
+        total_n = sum(int(stats["n"]) for stats in stats_list)
+        if total_n <= 0:
+            raise RuntimeError("Stats round produced non-positive total sample count")
+
+        num_features = len(stats_list[0]["mean"])
+        global_mean = np.zeros(num_features, dtype=np.float64)
+        total_ex2 = np.zeros(num_features, dtype=np.float64)
+        for stats in stats_list:
+            weight = float(stats["n"]) / float(total_n)
+            local_mean = np.asarray(stats["mean"], dtype=np.float64)
+            local_var = np.asarray(stats["var"], dtype=np.float64)
+            global_mean += local_mean * weight
+            total_ex2 += (local_var + np.square(local_mean)) * weight
+
+        global_var = np.maximum(total_ex2 - np.square(global_mean), 0.0)
+        global_scaler = StandardScaler()
+        global_scaler.mean_ = global_mean
+        global_scaler.var_ = global_var
+        global_scaler.scale_ = np.where(global_var > 0.0, np.sqrt(global_var), 1.0)
+        global_scaler.n_samples_seen_ = int(total_n)
+
+        _, fit_res = results[0]
+        tid = fit_res.config.get(TID)
+        scaler_dir = os.path.join("model", tid) if tid else "model"
+        scaler_path = os.path.join(scaler_dir, "scaler.pkl")
+        os.makedirs(scaler_dir, exist_ok=True)
+        joblib.dump(global_scaler, scaler_path)
+
+        print(
+            f"\n[NWDAF CustomStrategy] Stats round complete: clients={len(stats_list)} "
+            f"global_n={total_n} scaler={scaler_path}"
+        )
+
+        return fit_res.parameters, {DATA_SAMPLES: total_n}
+
     def aggregate_evaluate(self, results, **kwargs):
         metrics = super().aggregate_evaluate(results, **kwargs)
         if metrics is None or LOSS not in metrics:
             return metrics
-            
+
+        if results:
+            _, evaluate_res = results[0]
+            if evaluate_res.config.get("is_stats_round"):
+                print("\n[NWDAF CustomStrategy] Skipping early-stopping update for stats round")
+                return metrics
+
         val_loss = metrics[LOSS]
         print(f"\n[NWDAF CustomStrategy] Round Validation Loss: {val_loss:.4f} | Current LR: {self.current_lr}")
-        
+
         if val_loss < self.best_loss:
             self.best_loss = val_loss
             self.es_patience = 0
@@ -29,20 +105,24 @@ class EarlyStoppingFedAvg(FedAvg):
         else:
             self.es_patience += 1
             self.lr_patience += 1
-            print(f"[NWDAF CustomStrategy] Val Loss did not improve. ES Patience: {self.es_patience}/10 | LR Patience: {self.lr_patience}/5")
-            
+            print(f"[NWDAF CustomStrategy] Val Loss did not improve. "
+                  f"ES Patience: {self.es_patience}/{self.max_es_patience} | "
+                  f"LR Patience: {self.lr_patience}/{self.max_lr_patience}")
+
         # Early Stopping check
-        if self.es_patience >= 10:
-            print(f"[NWDAF CustomStrategy] Early stopping triggered! Validation loss plateaued for 10 rounds.")
+        if self.es_patience >= self.max_es_patience:
+            print(f"[NWDAF CustomStrategy] Early stopping triggered! "
+                  f"Validation loss plateaued for {self.max_es_patience} rounds.")
             print(f"[NWDAF CustomStrategy] Master ending current task gracefully.")
             class EarlyStoppingException(Exception):
                 pass
-            raise EarlyStoppingException("Validation loss plateaued for 10 rounds")
-            
+            raise EarlyStoppingException(
+                f"Validation loss plateaued for {self.max_es_patience} rounds")
+
         # ReduceLROnPlateau check
-        if self.lr_patience >= 5:
+        if self.lr_patience >= self.max_lr_patience:
             self.current_lr *= 0.5
-            self.lr_patience = 0  # Reset LR cooldown
+            self.lr_patience = 0
             print(f"[NWDAF CustomStrategy] Plateau reached. Halving Learning Rate to {self.current_lr}")
 
         return metrics
```

### N. `dataset.py` diff

```diff
diff --git a/examples/07_MTLF_training/dataset.py b/examples/07_MTLF_training/dataset.py
index eae889ea..b88d674f 100644
--- a/examples/07_MTLF_training/dataset.py
+++ b/examples/07_MTLF_training/dataset.py
@@ -1,9 +1,13 @@
+from __future__ import annotations
+
 import json
 import torch
 import numpy as np
 import os
 from torch.utils.data import Dataset, DataLoader
 from sklearn.preprocessing import StandardScaler
+import joblib
+from typing import Any, Optional
 
 def parse_notifications_json(json_path):
     with open(json_path, 'r') as f:
@@ -77,6 +81,43 @@ def parse_notifications_list(data):
         
     return measurements
 
+def notifications_to_log_feature_rows(data):
+    measurements = parse_notifications_list(data)
+    rows = []
+    for m in measurements:
+        rows.append(np.log1p(np.array(m['features'], dtype=np.float32)))
+    if not rows:
+        return np.empty((0, 10), dtype=np.float32)
+    return np.stack(rows)
+
+def build_shared_scaler_from_notifications(data):
+    rows = notifications_to_log_feature_rows(data)
+    scaler = StandardScaler()
+    if len(rows) > 0:
+        scaler.fit(rows)
+    return scaler, rows
+
+def resolve_shared_scaler_path(tid):
+    if not tid:
+        return None
+    return os.path.join('model', tid, 'scaler.pkl')
+
+def load_tid_documents(tid: str, group_id: Optional[str], cfg: dict[str, Any]) -> list[dict[str, Any]]:
+    from pymongo import MongoClient
+
+    mongo_uri = cfg.get("MONGO_URI", "mongodb://localhost:27017/")
+    m_client = MongoClient(mongo_uri)
+    col = m_client["daisy_mtlf"]["training_data"]
+
+    query = {"tid": tid}
+    if group_id:
+        query["group_id"] = group_id
+
+    docs = list(col.find(query))
+    if not docs:
+        raise ValueError(f"No data found in MongoDB for TID: {tid}, Group: {group_id}")
+    return docs
+
 def create_isolated_sliding_windows(ue_data_streams, seq_length, out_seq_len):
     samples = []
     for ue_ip, stream in ue_data_streams.items():
@@ -179,7 +220,61 @@ class EESTCNDataset(Dataset):
         x, y = self.samples[idx]
         return torch.tensor(x, dtype=torch.float32), torch.tensor(y, dtype=torch.float32)
 
-def get_dataloaders(data_dirs=None, tid=None, group_id=None, batch_size=32, seq_length=30, out_seq_len=5, save_scaler=True):
+def get_local_stats(
+    data_dirs=None,
+    tid=None,
+    group_id=None,
+    seq_length=30,
+    out_seq_len=5,
+):
+    """Calculate local scaler statistics on this client's own rows only."""
+    cfg_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), "daisyconfig.json")
+    cfg = {}
+    if os.path.exists(cfg_path):
+        with open(cfg_path, "r") as f:
+            cfg = json.load(f)
+
+    if tid is not None:
+        docs = load_tid_documents(tid, group_id, cfg)
+        dataset = EESTCNDataset(
+            mongo_data=docs,
+            seq_length=seq_length,
+            out_seq_len=out_seq_len,
+            scaler=None,
+        )
+    else:
+        if data_dirs is None:
+            data_dirs_str = cfg.get("DATA_DIRS", "ees_training_data")
+            data_dirs = [d.strip() for d in data_dirs_str.split(',')]
+        elif isinstance(data_dirs, str):
+            data_dirs = [data_dirs]
+        dataset = EESTCNDataset(
+            data_dirs=data_dirs,
+            run_indices=list(range(1, 13)),
+            seq_length=seq_length,
+            out_seq_len=out_seq_len,
+            scaler=None,
+        )
+
+    scaler = dataset.scaler
+    if not hasattr(scaler, "mean_"):
+        raise RuntimeError("Local scaler statistics are unavailable because no feature rows were produced")
+    return {
+        "mean": scaler.mean_.tolist(),
+        "var": scaler.var_.tolist(),
+        "n": int(scaler.n_samples_seen_),
+    }
+
+def get_dataloaders(
+    data_dirs=None,
+    tid=None,
+    group_id=None,
+    batch_size=32,
+    seq_length=30,
+    out_seq_len=5,
+    save_scaler=True,
+    require_shared_scaler=False,
+):
     # Read daisyconfig.json once
     import os
     cfg_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), "daisyconfig.json")
@@ -189,28 +284,35 @@ def get_dataloaders(data_dirs=None, tid=None, group_id=None, batch_size=32, seq_
             cfg = json.load(f)
 
     if tid is not None:
-        from pymongo import MongoClient
         from torch.utils.data import Subset
-        
-        mongo_uri = cfg.get("MONGO_URI", "mongodb://localhost:27017/")
-        m_client = MongoClient(mongo_uri)
-        col = m_client["daisy_mtlf"]["training_data"]
-        
-        query = {"tid": tid}
-        if group_id:
-            query["group_id"] = group_id
-            
-        docs = list(col.find(query))
-        if not docs:
-            raise ValueError(f"No data found in MongoDB for TID: {tid}, Group: {group_id}")
-        
-        full_dataset = EESTCNDataset(mongo_data=docs, seq_length=seq_length, out_seq_len=out_seq_len)
-        
-        if save_scaler:
-            import joblib
-            scaler_dir = os.path.join('model', tid) if tid else 'model'
-            os.makedirs(scaler_dir, exist_ok=True)
-            joblib.dump(full_dataset.scaler, os.path.join(scaler_dir, 'scaler.pkl'))
+
+        docs = load_tid_documents(tid, group_id, cfg)
+
+        scaler_path = resolve_shared_scaler_path(tid)
+        shared_scaler = None
+        if scaler_path and os.path.exists(scaler_path):
+            shared_scaler = joblib.load(scaler_path)
+            save_scaler = False
+            print(
+                f"[Dataset] TID={tid} group={group_id} loaded shared scaler from {scaler_path} "
+                f"(require_shared_scaler={require_shared_scaler})"
+            )
+        elif require_shared_scaler:
+            raise FileNotFoundError(
+                f"Shared scaler is required for TID {tid}, but {scaler_path} does not exist"
+            )
+        else:
+            print(
+                f"[Dataset] TID={tid} group={group_id} shared scaler missing; "
+                f"building in-memory local bootstrap dataset"
+            )
+
+        full_dataset = EESTCNDataset(
+            mongo_data=docs,
+            seq_length=seq_length,
+            out_seq_len=out_seq_len,
+            scaler=shared_scaler,
+        )
             
         total_len = len(full_dataset)
         train_len = int(0.8 * total_len)
@@ -231,7 +333,6 @@ def get_dataloaders(data_dirs=None, tid=None, group_id=None, batch_size=32, seq_
     train_dataset = EESTCNDataset(data_dirs=data_dirs, run_indices=list(range(1, 13)), seq_length=seq_length, out_seq_len=out_seq_len)
     
     if save_scaler:
-        import joblib
         os.makedirs('model', exist_ok=True)
         joblib.dump(train_dataset.scaler, os.path.join('model', 'scaler.pkl'))
```

### O. `master.py` diff

```diff
diff --git a/examples/07_MTLF_training/master.py b/examples/07_MTLF_training/master.py
index b6459692..17ac33e7 100644
--- a/examples/07_MTLF_training/master.py
+++ b/examples/07_MTLF_training/master.py
@@ -1,9 +1,16 @@
-import daisyfl as fl
 import argparse
 import numpy as np
+import os
+import sys
+from pathlib import Path
+
+DAISY_SRC_PY = Path(__file__).resolve().parents[2] / "src" / "py"
+if str(DAISY_SRC_PY) not in sys.path:
+    sys.path.insert(0, str(DAISY_SRC_PY))
+
+import daisyfl as fl
 from model import get_model
 from dataset import get_dataloaders
-import os
 
 parser = argparse.ArgumentParser()
 parser.add_argument("--server_address", type=str, help="my grpc server port")
```

### P. `examples/07_MTLF_training/daisyconfig.json` diff

```diff
diff --git a/examples/07_MTLF_training/daisyconfig.json b/examples/07_MTLF_training/daisyconfig.json
index 681390da..4a3fb587 100644
--- a/examples/07_MTLF_training/daisyconfig.json
+++ b/examples/07_MTLF_training/daisyconfig.json
@@ -1,4 +1,4 @@
 {
-    "MONGO_URI": "mongodb://localhost:27017/",
+    "MONGO_URI": "mongodb://127.0.0.1:27018",
     "DATA_DIRS": "ees_training_data"
-}
\ No newline at end of file
+}
```

### Q. `task_launcher.py` diff

```diff
diff --git a/src/py/daisyfl/common/task_launcher.py b/src/py/daisyfl/common/task_launcher.py
index 4eff90af..1187b857 100644
--- a/src/py/daisyfl/common/task_launcher.py
+++ b/src/py/daisyfl/common/task_launcher.py
@@ -115,7 +115,7 @@ class TaskLauncher:
             log(ERROR, "Can't load Class \"{}\" from \"{}\".".format(metrics_handler_path[1], metrics_handler_path[0]))
             raise Exception("Can't load Class \"{}\" from \"{}\".".format(metrics_handler_path[1], metrics_handler_path[0]))
         
-        strategy_instance = strategy(self.client_manager)
+        strategy_instance = strategy(self.client_manager, task_config=config)
         metrics_handler_instance = metrics_handler()
         operator_instance = operator(communicator=self.communicator, strategy=strategy_instance, metrics_handler=metrics_handler_instance)
```

### R. `server_api_handler.py` diff

```diff
diff --git a/src/py/daisyfl/master/server_api_handler.py b/src/py/daisyfl/master/server_api_handler.py
index e222c274..60ce927b 100644
--- a/src/py/daisyfl/master/server_api_handler.py
+++ b/src/py/daisyfl/master/server_api_handler.py
@@ -14,7 +14,7 @@
 # ==============================================================================
 
 from flask import Flask, request, make_response, Response, send_file
-from typing import Callable
+from typing import Callable, Optional
 from daisyfl.common.task_manager import TaskManager
 from daisyfl.utils.logger import log
 from daisyfl.utils.logger import WARNING, ERROR, INFO, DEBUG
@@ -25,6 +25,7 @@ import requests as http_requests
 import tarfile
 import json
 import os
+import shutil
 from pymongo import MongoClient
 
 # Keys used in the task config for artifact packaging
@@ -119,20 +120,50 @@ class ServerListener:
             # Dynamically initialize model from MODEL_META if provided
             model_meta = js.get("MODEL_META")
             task_id = js.get(TID, "unknown")
+            use_fixed_scaler = bool(js.get("USE_FIXED_SCALER", False))
+            seed_model_path = js.get("SEED_MODEL_PATH")
+            seed_scaler_path = js.get("SEED_SCALER_PATH")
             
             # Use TID as model storage directory: model/{tid}/model.npy
             if task_id != "unknown":
                 model_path = os.path.join("model", task_id, "model.npy")
                 js[MODEL_PATH] = model_path  # Update task config so TaskManager uses TID-scoped path
+                js[_SCALER_PATH] = os.path.join("model", task_id, "scaler.pkl")
             else:
                 model_path = js.get(MODEL_PATH, "model/model.npy")
-                
-            if model_meta:
-                try:
+            scaler_path = js.get(_SCALER_PATH, os.path.join(os.path.dirname(model_path) or "model", "scaler.pkl"))
+
+            log(
+                INFO,
+                "publish_task tid=%s fixed_scaler=%s seed_model=%s seed_scaler=%s model_path=%s scaler_path=%s",
+                task_id,
+                use_fixed_scaler,
+                seed_model_path,
+                seed_scaler_path,
+                model_path,
+                scaler_path,
+            )
+
+            try:
+                if seed_model_path:
+                    log(INFO, "Task %s uses warm-start seed model: %s", task_id, seed_model_path)
+                    self._copy_seed_file(seed_model_path, model_path, "seed model")
+                elif model_meta:
+                    log(INFO, "Task %s uses fresh init from MODEL_META", task_id)
                     self._init_model_from_meta(model_meta, model_path)
+                else:
+                    log(WARNING, "Task %s has neither seed model nor MODEL_META; existing model path will be used as-is", task_id)
                 except Exception as e:
-                    log(ERROR, "Failed to init model from MODEL_META: %s", e)
-                    return {"error": f"Model init failed: {e}"}, 500
+                    log(ERROR, "Failed to prepare model seed for task %s: %s", task_id, e)
+                    return {"error": f"Model init failed: {e}"}, 500
+
+            if use_fixed_scaler:
+                try:
+                    log(INFO, "Task %s uses fixed scaler seed: %s", task_id, seed_scaler_path)
+                    self._copy_seed_file(seed_scaler_path, scaler_path, "fixed scaler")
+                except Exception as e:
+                    log(ERROR, "Failed to prepare fixed scaler for task %s: %s", task_id, e)
+                    return {"error": f"Fixed scaler init failed: {e}"}, 500
 
             spawned_clients = []
 
@@ -165,6 +196,8 @@ class ServerListener:
                                    "--group_id", str(gid)]
                             if model_meta_json:
                                 cmd += ["--model_meta", model_meta_json]
+                            if use_fixed_scaler:
+                                cmd += ["--use_fixed_scaler"]
                             p = subprocess.Popen(cmd)
                             spawned_clients.append(p)
                         # Wait briefly for clients to start up and connect
@@ -173,6 +206,7 @@ class ServerListener:
                         log(WARNING, "No groups found in MongoDB for TID %s. No clients spawned.", task_id)
                 except Exception as e:
                     log(ERROR, "Failed to auto-spawn clients: %s", e)
+                    return {"error": f"Task preparation failed: {e}"}, 500
 
             if callback_url:
                 # Async path: return 202 immediately, train in background
@@ -267,6 +301,16 @@ class ServerListener:
         log(INFO, "Initialized model from MODEL_META -> %s (input=%d, output=%d, channels=%s)",
             model_path, input_size, num_classes, num_channels)
 
+    def _copy_seed_file(self, source_path: Optional[str], dest_path: str, label: str) -> None:
+        """Copy a seed artifact into the task-scoped model directory."""
+        if not source_path:
+            raise ValueError(f"{label} path is required")
+        if not os.path.isfile(source_path):
+            raise FileNotFoundError(f"{label} not found: {source_path}")
+        os.makedirs(os.path.dirname(dest_path) or ".", exist_ok=True)
+        shutil.copy2(source_path, dest_path)
+        log(INFO, "Seeded %s -> %s", label, dest_path)
+
     def _package_artifacts(self, task_config: dict) -> str:
         """
         Package model artifacts into a tar.gz file.
```

---

## 補充解讀

### 1. 這不是只有環境參數變更

雖然主 repo 層看起來只有 submodule pointer、IP、Mongo URI 這類調整，但 submodule 內部 dirty working tree 顯示目前本地 Daisy 已有實質行為修改，尤其集中在 `07_MTLF_training` 的 scaler lifecycle 與 task preparation。

### 2. 目前 retrain path 偏向 federated shared scaler

從 `client.py`、`custom_strategy.py`、`dataset.py` 的組合來看，現在的訓練路徑已不只是 local scaler 各自 fit，而是：

1. round 1 蒐集 local scaler stats
2. master 聚合全域 scaler
3. round 2 後各 client reload dataloader 改用 shared scaler

這代表目前本地 Daisy 已經朝 federated scaler aggregation 收斂。

### 3. fixed scaler / warm-start 能力也已開始進來

`server_api_handler.py` 已有：

- `SEED_MODEL_PATH`
- `SEED_SCALER_PATH`
- `USE_FIXED_SCALER`

因此目前本地修改不只是在修 scaler consistency，也在為 continue learning / warm-start / seeded retrain 舖路。

### 4. artifact directory 屬於執行產物

目前 `examples/07_MTLF_training/artifacts/` 下有大量未追蹤 `.tar.gz`，比較像訓練或 callback 產生的輸出，而不是設計本身的來源碼變更。若之後要進一步整理 working tree，可單獨再判斷這批產物要不要納入 ignore 或搬移。
