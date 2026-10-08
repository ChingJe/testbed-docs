# Documentation

撰寫、修改或 review 文件（包括 plans 與 durable review records）時使用本模組。重大 implementation planning 另用
[Planning](planning.md)；交付與文件狀態同步的時機見 [Delivery](delivery.md)。

## Document Ownership And Status

- stable workflow／review rules 放在本 policy；workspace routing 放在 root `AGENTS.md`；phase-specific decisions
  與 conformance map 放在 active plan。
- 未完成工作與 findings 放在 `plans/`；confirmed design 放在 `design/`；experiment definition 放在
  `experiments/`；只有具備 required evidence 並通過 user review 的結果放入 `records/`。
- Prototype 能 render、focused tests 通過或 implementation 存在，不構成完成 record。
- 文件狀態應使用能反映 evidence 的 open state，例如 `Implementation`、`Review Pending`、
  `Implementation Complete / Verification Incomplete`、`Ready for User Review` 或 `Completed`。
- 不在 README 複製容易失效的 current progress；README 只提供穩定入口、責任與 navigation。
- Plan 擁有 task-specific decisions 與 acceptance，並連結通用 workflow 規則而不重述其內容。

## Language

文件 prose language 依序由 explicit user instruction、existing document dominant language、sibling series
language、repository default 決定。Code identifiers、paths、schema fields 與 API names 保持 English。

在 context 中檢查改到的 prose 是否語言一致；需要判斷 series 慣例時再參考 current sibling 文件。語言與技術內容
可以一起 review；小幅修改不需要另做完整文件的 language-consistency pass。

## Verification

確認變更內容符合要求，且受影響的連結仍可使用。純 prose 變更不需要 application tests。
