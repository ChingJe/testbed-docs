# 5g-viz Notes

本目錄收錄 `5g-viz` 的背景筆記與工作過程文件。

這一層不是 canonical reference，也不是 onboarding 主入口。

其中不少文件保留了當時的 runtime 心智，例如 `start.sh`、pseudo-live、`main.py`、profile `.env` 等。這些內容應視為歷史上下文，而不是現況操作指南。

它的價值在於保留：

- 當時的實作脈絡
- 會議裡的討論與決策背景
- 內部閱讀與理解系統時留下的筆記

## 結構

- `meetings/`：會議紀錄
- `impl/`：實作過程、改善紀錄與局部決策
- `internals/`：系統內部閱讀筆記

## 什麼時候適合讀 `notes/`

適合的情況包括：

- 想知道某個設計是怎麼一步步長出來的
- 想補上下文，而不是只看目前系統的定稿描述
- 想查歷史討論、取捨理由或除錯過程

## 與其他目錄的邊界

- [`../guides/start-here/`](../guides/start-here/README.md)、[`../guides/ui-workflows/`](../guides/ui-workflows/README.md)、[`../guides/mental-model/`](../guides/mental-model/README.md)：面向讀者的理解層
- [`../design/`](../design/README.md)：目前系統行為的 canonical reference
- [`../plans/`](../plans/README.md)：規劃與設計探索

若需求是確認「系統現在實際怎麼運作」，應優先看 `design/*`，而不是 `notes/*`。
