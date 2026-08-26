# 5G NWDAF Infrastructure 文件

本分類保存新版 `5G_NWDAF_Infrastructure` testbed 的計畫、設計、實驗與驗證紀錄。
舊 `5G_Infrastructure`、歷史 replay experiments 與舊 VM 操作資料繼續留在 `5g-infra/`，
不和本分類共用時間線。

## 文件導覽

| 分類 | 用途 |
| --- | --- |
| [development_policy.md](development_policy.md) | 新版 testbed 的 planning、implementation、deployment、review 與 verification 規則 |
| [plans/](plans/README.md) | 尚未完成或即將開始的 implementation plans |
| [design/](design/README.md) | 已確認且仍有效的 testbed architecture decisions |
| [operations/](operations/README.md) | 實驗室或機器特定的操作補充與遷移 runbooks |
| [experiments/](experiments/README.md) | 實驗設計、scenario matrix 與 acceptance criteria |
| [records/](records/README.md) | 已完成的建置、變更、validation 與 run evidence |
| [archive/](archive/README.md) | 已被取代且不再作為目前依據的文件 |

## 文件責任

Infrastructure source repository 內的文件負責與當前程式碼綁定的 commands、configuration
reference 與一般操作介面。本分類不複製那些內容，而是保存跨 repository 的 testbed planning、
site-specific decisions、experiment design 與執行 evidence。

新增文件時應先判斷它描述的是未完成工作、已確認設計、操作補充、實驗定義或已完成紀錄，
再放入對應分類。Active plan 完成後應將結果整理為 record；被新計畫取代的舊內容才移入
`archive/`。

開始修改新版 testbed 的程式、設定、lifecycle 或 implementation-oriented plan 前，先讀 workspace root
`AGENTS.md`、[development_policy.md](development_policy.md) 與 active plan。對話經過 context compaction、
summarization 或 handoff 後，必須從磁碟完整重讀這三者，不假設 root `AGENTS.md` 會自動重新套用。
