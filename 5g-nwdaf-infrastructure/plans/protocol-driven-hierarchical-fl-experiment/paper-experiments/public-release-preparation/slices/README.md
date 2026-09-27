# 公開發布準備 Slice 計畫

本分類保存[公開發布準備主計畫](../../Public%20Release%20Preparation%20Master%20Plan.md)的逐階段盤點、決策、實作與
review。Slice 文件隨進度逐份建立，不預先建立尚未開始的後續計畫。

目前文件：

- [Slice 1：公開基線、Tracking 與保留範圍盤點](./Slice%201%20Public%20Baseline%20Tracking%20and%20Retention%20Inventory.md)：
  盤點 component source identity、testbed consumer、正式實驗保留資產、本地 artifacts、公開風險與 cleanup disposition。
- [Slice 2：正式 Tracking 切換與最小公開清理](./Slice%202%20Formal%20Tracking%20and%20Minimal%20Cleanup.md)：
  將已確認 disposition 展開成 exact tracking、source cleanup、測試縮減與驗證順序；NWDAF upstream portability commit 已完成並
  驗證，testbed tracking、source cleanup 與 focused verification 已完成並通過使用者 review，依使用者要求延後至 Slice 3
  一起提出 commit。
- [Slice 3：公開文件、Clean Checkout 與發布關卡](./Slice%203%20Public%20Documentation%20Clean%20Checkout%20and%20Release%20Gates.md)：
  盤點並規劃 repository-local 文件重寫、final exposure／license review、commit 前後分離的 clean-checkout verification、
  real-environment smoke 與 default-branch／visibility gates；fresh-provision GPU smoke、`main` fast-forward 與一次性 Gitleaks
  scan 已確認，目前為 `Planning / Decisions Confirmed`。
